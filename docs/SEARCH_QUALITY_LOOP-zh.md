# CodeGraph 语言验证指南

你需要验证 CodeGraph 完全支持某种特定的编程语言。用户会给你一个本地克隆的真实世界、流行的开源代码库路径。你的工作是使用 CodeGraph 的 API 对其运行一系列逼真的提示词，并验证结果是否足够好，足以声称该语言是**已覆盖且受支持**的。

在 LLM 能够可靠地使用 CodeGraph 的 MCP 工具导航该代码库之前——找到正确的符号、理解调用链、探索子系统、为实际任务获取有用的上下文——该语言**不算**已验证。

## 设置

### 1. 构建和索引

```bash
npm run build
rm -rf <codebase_path>/.codegraph
node dist/bin/codegraph.js init -iv <codebase_path>
```

`-iv` 标志会输出详细信息，显示提取进度、节点/边计数和计时。

### 2. 快速完整性检查

```bash
# 验证节点是否使用正确的限定名提取
sqlite3 <codebase_path>/.codegraph/codegraph.db \
  "SELECT name, kind, qualified_name FROM nodes WHERE kind = 'method' LIMIT 10;"

# 好：file.go::StructName::method_name  （存在所有者类型）
# 差：file.go::file.go::method_name     （缺少所有者类型——需要 getReceiverType）

# 检查边计数
sqlite3 <codebase_path>/.codegraph/codegraph.db \
  "SELECT kind, COUNT(*) FROM edges GROUP BY kind ORDER BY COUNT(*) DESC;"

# 检查节点种类分布
sqlite3 <codebase_path>/.codegraph/codegraph.db \
  "SELECT kind, COUNT(*) FROM nodes GROUP BY kind ORDER BY COUNT(*) DESC;"
```

如果方法的 `qualified_name` 中缺少所有者类型，请先修复此问题（参见[添加 getReceiverType](#添加-getreceivertype)），然后再继续完整的测试套件。

## 测试套件

针对代码库运行**所有**以下测试类别。直接使用 Node.js API——下面的测试脚本是模板。根据你正在测试的代码库中的实际类型、方法和子系统调整查询。

**每个测试的通过标准：**结果是否给 LLM 提供了足够正确的信息来回答问题或完成任务？如果你是 LLM，你会信任这些结果吗？

---

### 测试 1：`codegraph_explore` — 深度探索（最重要）

这是 LLM 使用的主要工具。它必须返回按文件分组的相关源代码，并包含针对自然语言查询的正确关系。使用**至少 5 种不同的查询类型**进行测试：

```bash
node -e "
const { CodeGraph } = require('./dist/index.js');
async function test() {
  const cg = await CodeGraph.open('<codebase_path>');

  const queries = [
    // A. 子系统探索——宽泛的主题，应找到正确的文件和关键类
    '缓存系统是如何工作的？',

    // B. 特定类/类型深入——应返回该类、其方法和相关类型
    'CacheBuilder 配置和构建过程',

    // C. 横切关注点——应在多个文件中找到实现
    '错误是如何处理和传播的？',

    // D. 数据流问题——应追踪多个层
    '数据如何从输入流向存储？',

    // E. 实现细节——特定方法行为
    '驱逐策略如何决定移除哪些条目？',
  ];

  for (const query of queries) {
    console.log(\`\n========================================\`);
    console.log(\`查询：\${query}\`);
    console.log(\`========================================\`);

    const subgraph = await cg.findRelevantContext(query, {
      searchLimit: 8, traversalDepth: 3, maxNodes: 80, minScore: 0.2,
    });

    // 显示入口点——这些是 LLM 首先看到的内容
    console.log(\`\n入口点（\${subgraph.roots.length} 个）：\`);
    for (const rootId of subgraph.roots.slice(0, 8)) {
      const node = subgraph.nodes.get(rootId);
      if (node) console.log(\`  \${node.name}（\${node.kind}）— \${node.filePath}:\${node.startLine}\`);
    }

    // 显示文件分布——正确的文件是否浮现出来？
    const fileGroups = new Map();
    for (const node of subgraph.nodes.values()) {
      if (!fileGroups.has(node.filePath)) fileGroups.set(node.filePath, []);
      fileGroups.get(node.filePath).push(node.name);
    }
    console.log(\`\n文件（\${fileGroups.size} 个）：\`);
    for (const [file, nodes] of [...fileGroups.entries()].sort((a,b) => b[1].length - a[1].length).slice(0, 8)) {
      console.log(\`  \${file}（\${nodes.length} 个符号）：\${nodes.slice(0, 6).join(', ')}\`);
    }

    // 显示边分布——关系是否被捕获？
    const edgeKinds = new Map();
    for (const edge of subgraph.edges) {
      edgeKinds.set(edge.kind, (edgeKinds.get(edge.kind) || 0) + 1);
    }
    console.log(\`\n边（\${subgraph.edges.length} 条）：\`);
    for (const [kind, count] of [...edgeKinds.entries()].sort((a,b) => b - a)) {
      console.log(\`  \${kind}：\${count}\`);
    }

    console.log(\`\n总计：\${subgraph.nodes.size} 个节点，\${subgraph.edges.length} 条边，\${fileGroups.size} 个文件\`);
  }

  await cg.close();
}
test().catch(console.error);
"
```

**每个查询需要检查的内容：**
- 入口点对问题是否有意义？
- 正确的文件是否浮现出来（不仅仅是测试文件或不相关的代码）？
- 是否有混合的边类型（calls、contains、extends、implements）——不仅仅是 `contains`？
- 节点数量是否感觉合理？太少（<5）意味着搜索失败；太多不相关的节点意味着噪音。

---

### 测试 2：`codegraph_search` — 符号查找

测试搜索特定符号是否能返回正确排名正确的结果。

```bash
node -e "
const { CodeGraph } = require('./dist/index.js');
async function test() {
  const cg = await CodeGraph.open('<codebase_path>');

  const searches = [
    // A. 按名称查找类
    { query: 'CacheBuilder', kinds: ['class'], desc: '查找特定类' },

    // B. 特定类型上的方法（经典的消歧测试）
    { query: 'CacheBuilder build', kinds: ['method'], desc: '特定类上的方法' },

    // C. 通用方法名——仍应找到相关的方法
    { query: 'get', kinds: ['method'], desc: '通用方法名' },

    // D. 接口/特征
    { query: 'Cache', kinds: ['interface'], desc: '查找接口' },

    // E. 枚举
    { query: 'Strength', kinds: ['enum'], desc: '查找枚举' },
  ];

  for (const s of searches) {
    console.log(\`\n--- \${s.desc}：\"\${s.query}\"（种类：\${s.kinds}）---\`);
    const results = cg.searchNodes(s.query, { limit: 10, kinds: s.kinds });
    for (const r of results) {
      console.log(\`  \${r.score.toFixed(1)} | \${r.node.name}（\${r.node.kind}）| \${r.node.qualifiedName}\`);
    }
    if (results.length === 0) console.log('  *** 无结果 ***');
  }

  await cg.close();
}
test().catch(console.error);
"
```

**需要检查的内容：**
- 目标符号是否排名前 3？
- 对于像 `get` 这样的通用名称，结果是否包含有助于消歧的限定名？
- 是否有零结果的查询？那是 bug。

---

### 测试 3：`codegraph_callers` / `codegraph_callees` — 调用链追踪

测试调用关系是否正确提取。

```bash
node -e "
const { CodeGraph } = require('./dist/index.js');
async function test() {
  const cg = await CodeGraph.open('<codebase_path>');

  // 选择 3-4 个重要方法并检查它们的调用图
  const symbols = ['build', 'get', 'put', 'invalidate'];

  for (const sym of symbols) {
    // 查找符号
    const results = cg.searchNodes(sym, { limit: 5, kinds: ['method'] });
    if (results.length === 0) { console.log(\`\${sym}：未找到\`); continue; }

    const node = results[0].node;
    console.log(\`\n--- \${node.name}（\${node.qualifiedName}）---\`);

    // 检查被调用者（它调用了什么？）
    const callees = cg.getCallees(node.id);
    console.log(\`  被调用者（\${callees.length} 个）：\${callees.slice(0, 10).map(c => c.node.name).join(', ')}\`);

    // 检查调用者（谁调用了它？）
    const callers = cg.getCallers(node.id);
    console.log(\`  调用者（\${callers.length} 个）：\${callers.slice(0, 10).map(c => c.node.name).join(', ')}\`);
  }

  await cg.close();
}
test().catch(console.error);
"
```

**需要检查的内容：**
- 方法是否有调用者和被调用者？如果一个方法两者都为 0，则边提取可能已损坏。
- 调用者/被调用者是否有意义？`build()` 方法应该调用类似构造器的东西，并被设置/初始化代码调用。
- 计数是否合理？流行代码库中的核心方法应该有多个调用者。

---

### 测试 4：`codegraph_impact` — 变更影响分析

测试影响半径是否正确识别受影响的代码。

```bash
node -e "
const { CodeGraph } = require('./dist/index.js');
async function test() {
  const cg = await CodeGraph.open('<codebase_path>');

  // 选择一个许多东西都依赖的核心类或接口
  const results = cg.searchNodes('<CoreClass>', { limit: 1, kinds: ['class', 'interface'] });
  if (results.length === 0) { console.log('未找到'); return; }

  const node = results[0].node;
  console.log(\`影响分析：\${node.name}（\${node.kind}）— \${node.filePath}\`);

  const impact = cg.getImpactRadius(node.id, 2);
  console.log(\`\n受影响的节点：\${impact.nodes.size}\`);
  console.log(\`受影响的边：\${impact.edges.length}\`);

  // 按文件分组
  const files = new Map();
  for (const n of impact.nodes.values()) {
    if (!files.has(n.filePath)) files.set(n.filePath, []);
    files.get(n.filePath).push(n.name);
  }
  console.log(\`受影响的文件：\${files.size}\`);
  for (const [file, nodes] of [...files.entries()].sort((a,b) => b[1].length - a[1].length).slice(0, 10)) {
    console.log(\`  \${file}：\${nodes.slice(0, 5).join(', ')}\`);
  }

  await cg.close();
}
test().catch(console.error);
"
```

**需要检查的内容：**
- 更改核心接口/类是否会显示广泛的影响半径？
- 受影响的文件是否合理（导入/扩展/使用它的东西）？
- 影响半径是否非空？核心类型上的零影响意味着缺少边。

---

### 测试 5：边提取质量

直接验证该语言是否提取了主要的边类型。

```bash
node -e "
const { CodeGraph } = require('./dist/index.js');
async function test() {
  const cg = await CodeGraph.open('<codebase_path>');

  // 检查整体边分布
  console.log('=== 边分布 ===');
  // （使用上面完整性检查中的 sqlite3 查询）

  // 查找一个继承自另一个类的类
  const classes = cg.searchNodes('', { limit: 100, kinds: ['class'] });
  let foundExtends = false, foundImplements = false;
  for (const r of classes) {
    const callees = cg.getCallees(r.node.id);
    // getCallees 返回所有出边，检查 extends/implements
    // 更好的方式：使用图遍历
  }

  // 验证特定关系类型是否存在
  const checks = [
    { desc: 'contains 边（类 → 方法）', query: 'SELECT COUNT(*) FROM edges WHERE kind = \"contains\"' },
    { desc: 'calls 边', query: 'SELECT COUNT(*) FROM edges WHERE kind = \"calls\"' },
    { desc: 'imports 边', query: 'SELECT COUNT(*) FROM edges WHERE kind = \"imports\"' },
    { desc: 'extends 边', query: 'SELECT COUNT(*) FROM edges WHERE kind = \"extends\"' },
    { desc: 'implements 边', query: 'SELECT COUNT(*) FROM edges WHERE kind = \"implements\"' },
  ];
  // 通过 sqlite3 运行这些（在完整性检查部分中显示）

  await cg.close();
}
test().catch(console.error);
"
```

```bash
sqlite3 <codebase_path>/.codegraph/codegraph.db "
  SELECT kind, COUNT(*) as cnt FROM edges GROUP BY kind ORDER BY cnt DESC;
"
```

**需要检查的内容：**
- `contains` 应该是最常见的（结构层次结构）。
- `calls` 应该很丰富——如果接近零，则该语言的调用提取已损坏。
- `imports` 应该存在——如果为零，则导入解析已损坏。
- 如果该语言有继承，`extends` 和 `implements` 应该存在——如果为零，`extractInheritance()` 可能无法处理该语言的 AST。

---

### 测试 6：节点提取完整性

验证所有预期的节点种类是否都被提取。

```bash
sqlite3 <codebase_path>/.codegraph/codegraph.db "
  SELECT kind, COUNT(*) as cnt FROM nodes GROUP BY kind ORDER BY cnt DESC;
"
```

**每种语言需要检查的内容：**

| 节点种类 | 是否预期？ | 备注 |
|-----------|-----------|------|
| `file` | 总是 | 每个源文件一个 |
| `class` | 如果语言有类 | |
| `method` | 如果语言有方法 | 应在 `qualified_name` 中包含所有者类型 |
| `function` | 如果语言有顶层函数 | |
| `interface` | 如果语言有接口/协议 | |
| `enum` | 如果语言有枚举 | |
| `enum_member` | 如果语言有枚举 | 枚举内部的值 |
| `import` | 总是 | 每个导入语句一个 |
| `variable` / `field` | 通常 | 字段、常量、顶层变量 |
| `struct` | 如果语言有结构体 | Go、Rust、C、Swift |
| `trait` | 如果语言有特征 | Rust |

如果预期的节点种类计数为 0，则该语言提取器缺少该 AST 类型。

---

### 测试 7：真实世界的 LLM 提示词

这是最终也是最重要的测试。模拟开发人员实际向使用 CodeGraph 的 LLM 提出的各种问题。对于每个提示词，运行 `findRelevantContext`（驱动 `codegraph_explore` 的函数），并评估返回的上下文是否能让 LLM 给出正确、完整的答案。

**运行至少 5 种这些提示词风格，根据实际代码库进行调整：**

```bash
node -e "
const { CodeGraph } = require('./dist/index.js');
async function test() {
  const cg = await CodeGraph.open('<codebase_path>');

  const prompts = [
    // 1. \"X 是如何工作的？\"——子系统理解
    '缓存驱逐策略是如何工作的？',

    // 2. \"X 在哪里实现的？\"——符号定位
    'LRU 驱逐逻辑在哪里实现的？',

    // 3. \"什么调用了 X？\"——使用发现
    '哪些代码触发了缓存失效？',

    // 4. \"我想改变 X，什么会受影响？\"——影响评估
    '如果我更改 Cache 接口，还有什么会受影响？',

    // 5. \"X 和 Y 如何交互？\"——跨组件关系
    'CacheBuilder 如何连接到 LocalCache？',

    // 6. \"向我展示从 A 到 B 的流程\"——数据/控制流
    '当缓存条目过期时会发生什么？',

    // 7. \"X 的所有实现是什么？\"——多态性
    '哪些类实现了 Cache 接口？',

    // 8. Bug 调查提示词
    '缓存条目在应该被驱逐时没有被驱逐——我应该查看哪里？',
  ];

  for (const prompt of prompts) {
    console.log(\`\n========================================\`);
    console.log(\`提示词：\${prompt}\`);
    console.log(\`========================================\`);

    const subgraph = await cg.findRelevantContext(prompt, {
      searchLimit: 8, traversalDepth: 3, maxNodes: 80, minScore: 0.2,
    });

    console.log(\`结果：\${subgraph.nodes.size} 个节点，\${subgraph.edges.length} 条边，\${subgraph.roots.length} 个入口点\`);

    console.log('入口点：');
    for (const rootId of subgraph.roots.slice(0, 5)) {
      const node = subgraph.nodes.get(rootId);
      if (node) console.log(\`  \${node.name}（\${node.kind}）— \${node.filePath}:\${node.startLine}\`);
    }

    const fileGroups = new Map();
    for (const node of subgraph.nodes.values()) {
      if (!fileGroups.has(node.filePath)) fileGroups.set(node.filePath, []);
      fileGroups.get(node.filePath).push(node.name);
    }
    console.log('最重要的文件：');
    for (const [file, nodes] of [...fileGroups.entries()].sort((a,b) => b[1].length - a[1].length).slice(0, 5)) {
      console.log(\`  \${file}（\${nodes.length}）：\${nodes.slice(0, 5).join(', ')}\`);
    }

    // 通过/失败判断
    const hasEntryPoints = subgraph.roots.length > 0;
    const hasEdges = subgraph.edges.length > 0;
    const hasMultipleFiles = fileGroups.size > 1;
    console.log(\`\\n裁决：\${hasEntryPoints && hasEdges && hasMultipleFiles ? '通过' : '失败 — 需要调查'}\`);
  }

  await cg.close();
}
test().catch(console.error);
"
```

**每个提示词需要检查的内容：**
- 它是否返回入口点？零入口点 = 完全失败。
- 入口点是否与问题**相关**？（不仅仅是恰好共享一个词的随机符号。）
- 它是否跨越多个文件？大多数真实问题涉及跨文件理解。
- 关系是否存在？LLM 需要理解符号之间的连接方式，而不仅仅是一个名称列表。
- **你**能否从这些上下文中回答问题？

---

## 诊断失败

| 症状 | 可能原因 | 修复位置 |
|---------|-------------|---------|
| 方法在 `qualified_name` 中缺少所有者类型 | 语言需要 `getReceiverType` | `src/extraction/languages/<lang>.ts` |
| `codegraph_explore` 返回不相关的文件 | 通用名称泛滥 FTS；共置提升不起作用 | `src/db/queries.ts`：`findNodesByExactName`，`src/context/index.ts` |
| 零条 `calls` 边 | `callTypes` 缺失或 AST 节点类型错误 | `src/extraction/languages/<lang>.ts`：`callTypes` |
| 零条 `extends`/`implements` 边 | `extractInheritance()` 不处理该语言的 AST | `src/extraction/tree-sitter.ts`：`extractInheritance()` |
| 缺少节点种类（没有枚举，没有接口） | 提取器中未列出 AST 类型 | `src/extraction/languages/<lang>.ts`：`enumTypes`、`interfaceTypes` 等 |
| 搜索词从查询中删除 | 该词在停用词列表中 | `src/search/query-utils.ts`：`STOP_WORDS` |
| 嵌套方法的 `qualified_name` 缺少类 | 提取未正确遍历父堆栈 | `src/extraction/tree-sitter.ts`：`visitNode()` |
| 缺少导入边 | `extractImport` 对此语法返回 null | `src/extraction/languages/<lang>.ts`：`extractImport` |
| C++ 类/结构体/枚举在宏命名空间中缺失 | 像 `NLOHMANN_JSON_NAMESPACE_BEGIN` 这样的宏导致 tree-sitter 将命名空间块错误解析为 `function_definition` | `src/extraction/languages/c-cpp.ts`：`isMisparsedFunction` 过滤坏名称；`src/extraction/tree-sitter.ts`：`visitFunctionBody` 提取结构节点 |
| C++ 类从 `.h` 头文件中丢失 | `.h` 文件默认为 `c` 语言，其 `classTypes: []` | `src/extraction/grammars.ts`：`looksLikeCpp()` — 基于内容的启发式方法在检测到 C++ 模式时将 `.h` 文件提升为 `cpp` |
| Ruby 模块中的方法在 `qualified_name` 中缺少所有者 | Ruby `module` AST 节点未被提取 | `src/extraction/languages/ruby.ts`：`visitNode` 钩子提取模块；`src/extraction/tree-sitter.ts`：`isInsideClassLikeNode` 包含 `module` 种类 |
| TypeScript 抽象类缺失 | `abstract_class_declaration` 不在 `classTypes` 中 | `src/extraction/languages/typescript.ts`：`classTypes` — 添加 `abstract_class_declaration` |
| 单表达式箭头函数静默丢弃 | `extractName` 在表达式主体中找到标识符而不是返回 `<anonymous>` | `src/extraction/tree-sitter.ts`：`extractName` — 对 `arrow_function`/`function_expression` 节点跳过标识符搜索 |
| Kotlin 接口/枚举被提取为类 | `class_declaration` 首先匹配 `classTypes`；`interfaceTypes`/`enumTypes` 从不触发 | `src/extraction/languages/kotlin.ts`：`classifyClassNode` 检测 AST 子节点中的 `interface`/`enum` 关键字 |
| Kotlin 函数提取到零个调用 | Tree-sitter 语法不使用字段名，因此 `getChildByField(node, 'function_body')` 返回 null | `src/extraction/languages/kotlin.ts`：`resolveBody` 按类型（`function_body`、`class_body`、`enum_class_body`）查找主体 |
| Kotlin `navigation_expression` 调用未干净解析 | `navigation_expression` 落入 `getNodeText`，产生带有括号的混乱名称 | `src/extraction/tree-sitter.ts`：`extractCall` — 通过从 `navigation_suffix > simple_identifier` 提取方法名来处理 `navigation_expression` |
| Kotlin `fun interface` 声明不可见 | tree-sitter-kotlin 不支持 `fun interface` 语法（Kotlin 1.4+），产生 ERROR 或错误解析为 `function_declaration` | `src/extraction/languages/kotlin.ts`：`visitNode` 检测三种错误解析模式：(1) ERROR 节点 + lambda 主体，(2) 带有 `user_type("interface")` 直接子节点且名称在 ERROR 子节点中的 function_declaration，(3) 带有包含 `user_type("interface")` + 名称的 ERROR 子节点的 function_declaration。`isFunInterfaceNode` 检查直接和 ERROR 嵌套的 `user_type` 子节点 |
| Kotlin 类/接口方法在嵌套 `fun interface` 存在时缺失 | Tree-sitter 错误解析了父主体为 ERROR（以 `{` 开头）+ class_body（嵌套接口主体）；`resolveBody` 找到了错误的主体 | `src/extraction/languages/kotlin.ts`：`resolveBody` 优先选择以 `{` 开头的 ERROR 主体；`visitNode` 从 `fun interface` 检测中排除类似主体的 ERROR |
| Svelte `$props()` 解构产生丑陋的变量名 | `let { x, y } = $props()` 有 `object_pattern` 作为变量名节点；`getNodeText` 返回完整模式 | `src/extraction/tree-sitter.ts`：`extractVariable` 跳过 `object_pattern`/`array_pattern` 命名声明符 |
| Svelte 模板函数调用不可见（例如 `class={cn(...)}`） | SvelteExtractor 只解析了 `<script>` 块，错过了模板标记中的调用 | `src/extraction/svelte-extractor.ts`：`extractTemplateCalls` 扫描模板中 `{expression}` 块内的调用模式 |
| Svelte `$state`/`$derived` rune 调用产生噪音 | Rune 是编译器内置函数，不是真正的函数调用 | `src/extraction/svelte-extractor.ts` 从未解析的引用中过滤 `SVELTE_RUNES` 集合 |
| 对象字面量 getter/setter 被提取为独立函数 | `object` 字面量内部的 `method_definition` 被视为类方法 | `src/extraction/tree-sitter.ts`：`extractMethod` 跳过其父节点为 `object`/`object_expression` 的 `method_definition` 节点 |
| JavaScript `class extends` 产生零条继承边 | JS tree-sitter 使用 `class_heritage → identifier`（裸），而不是像 TypeScript 那样的 `class_heritage → extends_clause → identifier` | `src/extraction/tree-sitter.ts`：`extractInheritance` — 当父节点是 `class_heritage` 时处理裸 `identifier`/`type_identifier` 子节点 |
| PHP 特征被提取为类 | `trait_declaration` 在 `classTypes` 中但 `extractClass` 硬编码了 `class` 种类 | `src/extraction/languages/php.ts`：`classifyClassNode` 为 `trait_declaration` 返回 `'trait'`；`src/extraction/tree-sitter-types.ts` 将 `'trait'` 添加到返回类型 |
| PHP 类属性缺失（0 个字段节点） | `extractField` 查找 `variable_declarator` 子节点；PHP 使用 `property_element > variable_name > name` | `src/extraction/tree-sitter.ts`：`extractField` — 处理带有 `variable_name > name` 路径的 `property_element` 子节点 |
| PHP 类常量在类内部被跳过 | `variableTypes` 检查有 `!isInsideClassLikeNode()` 守卫，因此类内部的 `const_declaration` 会漏掉 | `src/extraction/languages/php.ts`：`visitNode` 钩子捕获 `const_declaration`，提取 `const_element > name` 作为 `constant` 种类 |
| PHP `use TraitName` 在类内部不可见 | 类主体中的 `use_declaration` 节点未处理边 | `src/extraction/languages/php.ts`：`visitNode` 钩子从 `use_declaration` 提取特征名并创建 `implements` 未解析引用 |

## 修复问题后

```bash
npm run build
rm -rf <codebase_path>/.codegraph
node dist/bin/codegraph.js init -iv <codebase_path>
# 重新运行上面失败的测试
```

在将语言标记为已验证之前，始终运行完整的测试套件：

```bash
npm test
```

## 添加 `getReceiverType`

**仅适用于方法在 AST 中是顶层或在其所有者类型之外的语言。** 如果语言将方法嵌套在类/结构体主体中（Python、Java、TypeScript、C#），则限定名已包含父节点——在添加任何内容之前，请使用完整性检查进行验证。

### 1. 将钩子添加到语言提取器

在 `src/extraction/languages/<lang>.ts` 中，将 `getReceiverType` 添加到提取器对象：

```typescript
getReceiverType: (node, source) => {
  // 从方法的 AST 节点中提取所有者类型名。
  // 返回类型名字符串，如果不适用则返回 undefined。
  //
  // tree-sitter.ts 中的核心 extractMethod() 将使用它来设置：
  //   qualifiedName = `${filePath}::${receiverType}::${methodName}`
},
```

### 2. 参考：Go 实现

```typescript
// src/extraction/languages/go.ts
getReceiverType: (node, source) => {
  const receiver = getChildByField(node, 'receiver');
  if (!receiver) return undefined;
  const text = getNodeText(receiver, source);
  const match = text.match(/\*?\s*([A-Za-z_][A-Za-z0-9_]*)\s*\)/);
  return match?.[1];
},
```

### 3. 使用位置

`src/extraction/tree-sitter.ts` 中的 `extractMethod()`：

```typescript
const receiverType = this.extractor.getReceiverType?.(node, this.source);
if (receiverType) {
  extraProps.qualifiedName = `${this.filePath}::${receiverType}::${name}`;
}
```

## 关键文件

| 文件 | 角色 |
|------|------|
| `src/extraction/languages/<lang>.ts` | 语言提取器——节点类型、调用类型、`getReceiverType` |
| `src/extraction/tree-sitter.ts` | 核心提取——`extractMethod()`、`extractCall()`、`extractInheritance()` |
| `src/extraction/tree-sitter-types.ts` | `LanguageExtractor` 接口定义 |
| `src/search/query-utils.ts` | `STOP_WORDS`、`extractSearchTerms`、`scorePathRelevance` |
| `src/db/queries.ts` | `searchNodesFTS` (BM25)、`findNodesByExactName`（共置提升） |
| `src/context/index.ts` | `findRelevantContext` — 混合搜索 + 图遍历 |
| `src/mcp/tools.ts` | MCP 工具处理器 — `codegraph_explore` 实现 |

## 语言状态

### 已验证

- [x] **Go** — `getReceiverType` 从 `func (sl *Type) method()` 提取接收者
- [x] **Swift** — 不需要。Tree-sitter 将方法嵌套在类/扩展主体内
- [x] **Java** — 不需要。方法嵌套在类主体中。已针对 Guava 验证
- [x] **Python** — 不需要。方法嵌套在类主体中。已针对 Flask 验证
- [x] **Rust** — `getReceiverType` 向上遍历到父 `impl_item` 来提取类型名。还从结构体到 impl 方法添加了 `contains` 边。已针对 Deno 验证
- [x] **C** — 不需要。C 中没有方法。强大的函数/结构体/枚举提取，调用边密度极好。已针对 Redis 验证
- [x] **C++** — 仅头文件库不需要。`isMisparsedFunction` 钩子过滤由宏引起的错误解析产物（例如 `NLOHMANN_JSON_NAMESPACE_BEGIN`）。`visitFunctionBody` 现在提取宏混淆的"函数"主体内的结构节点（类/结构体/枚举）。基于内容的 `.h` 检测（`grammars.ts` 中的 `looksLikeCpp`）将 C++ 头文件提升为 `cpp` 语言，以便 `.h` 文件中的类被提取。已针对 nlohmann/json 和 gRPC 验证。注意：类外 `Type::method()` 定义需要 `getReceiverType`，但在仅头文件代码库中不常见。
- [x] **C#** — 不需要。方法嵌套在类主体中。为 C# 的 `: Parent, IInterface` 语法在 `extractInheritance` 中添加了 `base_list` 处理。为 C# `property_declaration` 节点添加了 `propertyTypes` 支持。修复了 `extractField` 以处理 C# 的嵌套 `variable_declaration > variable_declarator` 结构。已针对 Jellyfin 验证
- [x] **Ruby** — 不需要 `getReceiverType`。方法嵌套在类主体中。添加了 `visitNode` 钩子以提取 Ruby `module` 节点（关注点、命名空间），具有正确的包含关系和限定名。模块内部的方法获得 `Module::method` 限定名。还为语言钩子将 `ExtractorContext` 与 `pushScope`/`popScope` 连接起来。已针对 Discourse 验证
- [x] **TypeScript** — 不需要 `getReceiverType`。方法嵌套在类主体中。将 `abstract_class_declaration` 添加到 `classTypes` 中，以便正确提取抽象类。修复了单表达式箭头函数提取（`const fn = () => expr` 被静默丢弃，因为 `extractName` 拾取了主体标识符而不是为父名称解析返回 `<anonymous>`）。已针对 Grafana 验证
- [x] **Dart** — 不需要 `getReceiverType`。方法嵌套在类主体中。为基于选择器的方法调用（例如 `object.method()`）添加了裸调用提取。已针对 Flutter 验证
- [x] **Kotlin** — `getReceiverType` 从扩展函数 `fun Type.method()` 提取接收者。添加了 `classifyClassNode` 以区分接口/枚举与类（都使用 `class_declaration` AST 节点）。添加了 `resolveBody` 钩子，因为 Kotlin 的 tree-sitter 语法不使用字段名。为方法调用提取添加了 `navigation_expression` 处理。通过 `extraClassNodeTypes` 添加了 `object_declaration`。为 Kotlin 的 `: Parent, Interface` 语法在 `extractInheritance` 中添加了 `delegation_specifier` 处理。还修复了 `extractInterface` 以访问主体子节点（接口方法之前未被提取）。添加了 `visitNode` 钩子以处理 `fun interface` (SAM) 声明 — tree-sitter-kotlin 不支持此 Kotlin 1.4+ 语法，产生 ERROR 或 function_declaration 错误解析；该钩子检测两种模式并提取接口。已针对 Koin、LeakCanary 验证
- [x] **Svelte** — 自定义 `SvelteExtractor` 将 `<script>` 块委托给 TS/JS 解析器；为每个 `.svelte` 文件创建 `component` 节点。添加了模板表达式调用提取：扫描标记中 `{expression}` 块的函数调用（例如 `class={cn(...)}`），从组件到被调用者创建调用边 — 将 Svelte 调用边从 29 增加到 387。过滤了 Svelte 5 rune 调用（`$state`、`$props`、`$derived`、`$effect`、`$bindable`）。还修复了：解构的 `$props()` 模式（例如 `let { x, y } = $props()`）不再被提取为丑陋的多行变量名（在 `extractVariable` 中跳过 `object_pattern`/`array_pattern`）。对象字面量 getter/setter 方法不再被提取为独立函数。已针对 shadcn-svelte 验证
- [x] **PHP** — 不需要 `getReceiverType`。方法嵌套在类主体中。添加了 `classifyClassNode` 以区分特征与类（`trait_declaration` → `trait` 种类）。将 `'trait'` 添加到 `tree-sitter-types.ts` 中的 `classifyClassNode` 返回类型以及访问器中的处理。修复了 PHP 属性提取：`extractField` 现在处理 `property_element > variable_name > name` AST 结构（添加了 4,366 个字段节点）。添加了 `visitNode` 钩子用于类常量（类内部的 `const_declaration` 被 `variableTypes` 守卫跳过）和特征 `use` 声明（`use HasFactory, SoftDeletes;` 创建 `implements` 边 — 从 636 增加到 1,514）。已针对 Laravel 验证

### 需要验证

（目前无）
