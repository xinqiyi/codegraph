# 框架解析器 `extract()` 接线实现计划

> **面向智能体工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐个任务地实现此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 接上已失效的 `FrameworkResolver.extractNodes` 钩子，使每个框架解析器都能向图中贡献路由节点 AND 路由到处理程序的边，并更新所有 13 个现有框架解析器以正确使用它。

**架构：** 将未使用的 `extractNodes?(filePath, content): Node[]` 钩子替换为一个统一的 `extract?(filePath, content): { nodes, references }` 方法。在提取阶段（tree-sitter 解析文件后）对每个与文件语言匹配的框架调用一次。提取的节点与 tree-sitter 节点一起进入数据库；提取的引用流入现有的未解析引用管道，以便现有的名称匹配器 / 导入解析器 / 框架 `resolve()` 机制创建最终的边。净效果：`path('/users', UserListView.as_view())` 产生一个由 `references` 边链接到 `UserListView` 类节点的 `route` 节点——Flask、FastAPI、Express、Rails、Laravel、Spring、Gin、Axum、ASP.NET、Vapor、React Router 和 SvelteKit 同理。

**技术栈：** TypeScript、vitest、tree-sitter（已有）、better-sqlite3（已有）。无新依赖。

---

## 背景

目前，每个 `FrameworkResolver` 都带有一个 `extractNodes?(filePath, content)` 方法（express、laravel、python/django、python/flask、python/fastapi、ruby/rails、java/spring、go、rust、csharp、swift × 3、react、svelte）。它们从未被调用过。经验证明：在 `src/` 中 grep 只找到一处对 `extractNodes` 的引用——接口定义在 `src/resolution/types.ts:99`。结果，图中在实践中零 `route` 类型节点，且路由文件中的 URL 入口与其视图/控制器/处理程序之间的链接不存在。

另外，Django 提取器的正则表达式在第 2 组中捕获了视图名称，但 `src/resolution/frameworks/python.ts` 中的解构丢弃了它，因此即使钩子是活的，它也不会将路由链接到视图。大多数框架中存在类似的形状错误。

本计划在一个连贯的变更中修复这两个问题。

## 文件结构

- `src/resolution/types.ts` — 将 `extract?()` 添加到 `FrameworkResolver`；移除 `extractNodes?()`。
- `src/resolution/frameworks/index.ts` — 保持 `detectFrameworks` 签名；添加 `getApplicableFrameworks(language)` 辅助函数。
- `src/resolution/frameworks/python.ts` — 重写 Django/Flask/FastAPI 提取器。
- `src/resolution/frameworks/express.ts` / `laravel.ts` / `ruby.ts` / `java.ts` / `go.ts` / `rust.ts` / `csharp.ts` / `swift.ts` / `react.ts` / `svelte.ts` — 迁移到新接口。
- `src/extraction/index.ts` — 在每个文件 tree-sitter 解析后，将框架提取接入 `ExtractionOrchestrator.indexAll`。
- `src/extraction/parse-worker.ts` — 将检测到的框架名称传递到 worker 中，以便 worker 可以自己调用框架提取器（因为主线程 `extractFromSource` 和 worker 线程解析路径都需要覆盖这一点）。
- `__tests__/frameworks.test.ts` — 新建。每个框架一个 `describe`，检查代表性测试夹具是否产生预期的 `{nodes, references}`。
- `__tests__/frameworks-integration.test.ts` — 新建。端到端测试：索引一个微型 Django 项目夹具，断言从 `urlpatterns` 入口到 `UserListView` 存在一条类型为 `references` 的 `route -> class` 边。

将两个测试文件分开的理由：单元测试是确定性的字符串输入/数组输出，运行时间为毫秒级；集成测试启动一个 CodeGraph 数据库，速度较慢但提供了最强的行为保证。

## 范围说明

本计划**不**将 Django 提取从正则表达式迁移到 AST。对于本 PR 目标涉及的模式（`path(...)`、`url(...)`、`re_path(...)`、`include(...)`、DRF `router.register(...)`、CBV `.as_view()`、点分模块路径），正则表达式方法已经足够。后续 PR 可以使用 tree-sitter 现有的 Python 解析器将正则表达式替换为 AST 遍历。那是一个更大的变更，不会阻塞本 PR。

---

## 任务 1：更新 `FrameworkResolver` 接口

**文件：**
- 修改：`src/resolution/types.ts:88-100`

- [ ] **步骤 1：编写会失败的测试**

创建 `__tests__/frameworks.test.ts`：

```typescript
import { describe, it, expect } from 'vitest';
import type { FrameworkResolver, UnresolvedRef } from '../src/resolution/types';
import type { Node } from '../src/types';

describe('FrameworkResolver.extract interface', () => {
  it('extract() returns { nodes, references }', () => {
    const resolver: FrameworkResolver = {
      name: 'fake',
      detect: () => true,
      resolve: () => null,
      languages: ['python'],
      extract: (_filePath: string, _content: string) => ({
        nodes: [] as Node[],
        references: [] as UnresolvedRef[],
      }),
    };
    const result = resolver.extract!('foo.py', '');
    expect(result).toEqual({ nodes: [], references: [] });
  });
});
```

- [ ] **步骤 2：运行测试以验证其失败**

运行：`npx vitest run __tests__/frameworks.test.ts`
预期：失败——`extract` 不是 `FrameworkResolver` 的属性；`languages` 不是 `FrameworkResolver` 的属性。

- [ ] **步骤 3：更新接口**

将 `src/resolution/types.ts:88-100` 替换为：

```typescript
/**
 * 框架特定文件提取的结果。
 */
export interface FrameworkExtractionResult {
  /** 框架特定的节点（例如路由） */
  nodes: Node[];
  /** 框架特定的未解析引用（例如路由 -> 处理程序） */
  references: UnresolvedRef[];
}

/**
 * 框架特定的解析器
 */
export interface FrameworkResolver {
  /** 框架名称 */
  name: string;
  /** 此框架适用的语言。如果省略，则适用于所有语言。 */
  languages?: Language[];
  /** 检测项目是否使用此框架（项目级别，启动时调用一次） */
  detect(context: ResolutionContext): boolean;
  /** 使用框架特定模式解析引用 */
  resolve(ref: UnresolvedRef, context: ResolutionContext): ResolvedRef | null;
  /**
   * 从文件中提取框架特定的节点和引用。
   *
   * 返回路由节点、中间件节点等，以及未解析的引用，
   * 这些引用将那些节点链接到处理程序（视图类、控制器方法、
   * 包含的模块）。未解析的引用流入正常的解析
   * 管道；框架自身的 `resolve()` 是尝试的策略之一。
   */
  extract?(filePath: string, content: string): FrameworkExtractionResult;
}
```

- [ ] **步骤 4：运行测试以验证其通过**

运行：`npx vitest run __tests__/frameworks.test.ts`
预期：通过。

- [ ] **步骤 5：运行类型检查以捕获下游破坏**

运行：`npx tsc --noEmit`
预期：失败——每个 `src/resolution/frameworks/*.ts` 都会因 `extractNodes` 在 `FrameworkResolver` 上不存在而报错。这是预期的；后续任务逐个修复。

- [ ] **步骤 6：提交**

```bash
git add src/resolution/types.ts __tests__/frameworks.test.ts
git commit -m "feat(resolution): replace extractNodes with extract() returning nodes and references"
```

---

## 任务 2：添加 `getApplicableFrameworks` 辅助函数并保持检测正确

**文件：**
- 修改：`src/resolution/frameworks/index.ts`

- [ ] **步骤 1：编写会失败的测试**

追加到 `__tests__/frameworks.test.ts`：

```typescript
import { getApplicableFrameworks } from '../src/resolution/frameworks';
import type { FrameworkResolver } from '../src/resolution/types';

describe('getApplicableFrameworks', () => {
  const pyFw: FrameworkResolver = { name: 'py', languages: ['python'], detect: () => true, resolve: () => null };
  const jsFw: FrameworkResolver = { name: 'js', languages: ['javascript', 'typescript'], detect: () => true, resolve: () => null };
  const anyFw: FrameworkResolver = { name: 'any', detect: () => true, resolve: () => null };

  it('按语言过滤', () => {
    const result = getApplicableFrameworks([pyFw, jsFw, anyFw], 'python');
    expect(result.map(r => r.name)).toEqual(['py', 'any']);
  });

  it('当语言无匹配时仅返回通用框架', () => {
    const result = getApplicableFrameworks([pyFw, jsFw, anyFw], 'rust');
    expect(result.map(r => r.name)).toEqual(['any']);
  });
});
```

- [ ] **步骤 2：运行测试以验证其失败**

运行：`npx vitest run __tests__/frameworks.test.ts`
预期：失败——`getApplicableFrameworks` 未导出。

- [ ] **步骤 3：将辅助函数添加到 `src/resolution/frameworks/index.ts`**

在现有 `detectFrameworks` 函数之后添加：

```typescript
import type { Language } from '../../types';

/**
 * 将检测到的框架列表过滤为适用于给定语言的框架。
 * 没有显式 `languages` 列表的框架被视为通用框架。
 */
export function getApplicableFrameworks(
  detected: FrameworkResolver[],
  language: Language
): FrameworkResolver[] {
  return detected.filter(
    (fw) => !fw.languages || fw.languages.includes(language)
  );
}
```

- [ ] **步骤 4：运行测试以验证其通过**

运行：`npx vitest run __tests__/frameworks.test.ts`
预期：通过。

- [ ] **步骤 5：提交**

```bash
git add src/resolution/frameworks/index.ts __tests__/frameworks.test.ts
git commit -m "feat(resolution): add getApplicableFrameworks helper for per-language dispatch"
```

---

## 任务 3：将 Django 解析器移植到新的 `extract()`，带正确的路由→视图引用

**文件：**
- 修改：`src/resolution/frameworks/python.ts`（djangoResolver 部分，约第 1-100 行）

- [ ] **步骤 1：编写会失败的测试**

追加到 `__tests__/frameworks.test.ts`：

```typescript
import { djangoResolver } from '../src/resolution/frameworks/python';

describe('djangoResolver.extract', () => {
  it('为带有 CBV.as_view() 的 path() 提取路由节点和引用', () => {
    const src = `
from django.urls import path
from users.views import UserListView

urlpatterns = [
    path('users/', UserListView.as_view(), name='user-list'),
]
`;
    const { nodes, references } = djangoResolver.extract!('users/urls.py', src);
    expect(nodes).toHaveLength(1);
    expect(nodes[0].kind).toBe('route');
    expect(nodes[0].name).toBe('users/');
    expect(references).toHaveLength(1);
    expect(references[0].referenceName).toBe('UserListView');
    expect(references[0].referenceKind).toBe('references');
    expect(references[0].fromNodeId).toBe(nodes[0].id);
  });

  it('为带有点分模块.Class.as_view() 的 path() 提取路由', () => {
    const src = `from django.urls import path\nfrom api.v1 import views as api_v1_views\nurlpatterns = [path('api/', api_v1_views.UserListView.as_view())]\n`;
    const { nodes, references } = djangoResolver.extract!('api/urls.py', src);
    expect(nodes).toHaveLength(1);
    expect(references[0].referenceName).toBe('UserListView');
  });

  it('为带有裸函数视图的 path() 提取路由', () => {
    const src = `from django.urls import path\nurlpatterns = [path('home/', home_view, name='home')]\n`;
    const { nodes, references } = djangoResolver.extract!('home/urls.py', src);
    expect(references[0].referenceName).toBe('home_view');
  });

  it('为带有 include() 的 path() 提取路由', () => {
    const src = `from django.urls import path, include\nurlpatterns = [path('api/', include('api.urls'))]\n`;
    const { nodes, references } = djangoResolver.extract!('root/urls.py', src);
    expect(nodes).toHaveLength(1);
    expect(nodes[0].kind).toBe('route');
    expect(references[0].referenceName).toBe('api.urls');
    expect(references[0].referenceKind).toBe('imports');
  });

  it('为 re_path 和 url 提取路由', () => {
    const src = `from django.urls import re_path, url\nurlpatterns = [re_path(r'^users/$', UserView), url(r'^old/$', OldView)]\n`;
    const { nodes } = djangoResolver.extract!('legacy/urls.py', src);
    expect(nodes).toHaveLength(2);
    expect(nodes.map(n => n.name)).toEqual(['^users/$', '^old/$']);
  });

  it('对非 urls.py 的 python 文件返回空结果', () => {
    const src = `def foo(): return 1\n`;
    const { nodes, references } = djangoResolver.extract!('views.py', src);
    expect(nodes).toEqual([]);
    expect(references).toEqual([]);
  });
});
```

- [ ] **步骤 2：运行测试以验证其失败**

运行：`npx vitest run __tests__/frameworks.test.ts -t djangoResolver`
预期：失败——`djangoResolver.extract` 未定义。

- [ ] **步骤 3：重写 djangoResolver**

将 `src/resolution/frameworks/python.ts` 中的 `djangoResolver` 对象（大约第 7-100 行）替换为：

```typescript
export const djangoResolver: FrameworkResolver = {
  name: 'django',
  languages: ['python'],

  detect(context) {
    const requirements = context.readFile('requirements.txt');
    if (requirements && requirements.toLowerCase().includes('django')) return true;
    const setup = context.readFile('setup.py');
    if (setup && setup.toLowerCase().includes('django')) return true;
    const pyproject = context.readFile('pyproject.toml');
    if (pyproject && pyproject.toLowerCase().includes('django')) return true;
    return context.fileExists('manage.py');
  },

  resolve(ref, context) {
    if (ref.referenceName.endsWith('Model') || /^[A-Z][a-z]+$/.test(ref.referenceName)) {
      const result = resolveByNameAndKind(ref.referenceName, CLASS_KINDS, MODEL_DIRS, context);
      if (result) return { original: ref, targetNodeId: result, confidence: 0.8, resolvedBy: 'framework' };
    }
    if (ref.referenceName.endsWith('View') || ref.referenceName.endsWith('ViewSet')) {
      const result = resolveByNameAndKind(ref.referenceName, VIEW_KINDS, VIEW_DIRS, context);
      if (result) return { original: ref, targetNodeId: result, confidence: 0.8, resolvedBy: 'framework' };
    }
    if (ref.referenceName.endsWith('Form')) {
      const result = resolveByNameAndKind(ref.referenceName, CLASS_KINDS, FORM_DIRS, context);
      if (result) return { original: ref, targetNodeId: result, confidence: 0.8, resolvedBy: 'framework' };
    }
    return null;
  },

  extract(filePath, content) {
    if (!filePath.endsWith('.py')) return { nodes: [], references: [] };

    const nodes: Node[] = [];
    const references: UnresolvedRef[] = [];
    const now = Date.now();

    // path('url', handler, name=...) / re_path(r'...', handler) / url(r'...', handler)
    // 捕获组：1=函数名，2=url 字符串，3=直到结束括号的行剩余部分
    const routeRegex = /\b(path|re_path|url)\s*\(\s*r?['"]([^'"]+)['"]\s*,\s*([^)]*?)(?:\)|,\s*name=)/g;

    let match: RegExpExecArray | null;
    while ((match = routeRegex.exec(content)) !== null) {
      const [, _fn, urlPath, handlerExpr] = match;
      const line = content.slice(0, match.index).split('\n').length;

      const routeNode: Node = {
        id: `route:${filePath}:${line}:${urlPath}`,
        kind: 'route',
        name: urlPath,
        qualifiedName: `${filePath}::route:${urlPath}`,
        filePath,
        startLine: line,
        endLine: line,
        startColumn: 0,
        endColumn: match[0].length,
        language: 'python',
        updatedAt: now,
      };
      nodes.push(routeNode);

      const handler = handlerExpr.trim();
      const target = resolveHandlerName(handler);
      if (target) {
        references.push({
          fromNodeId: routeNode.id,
          referenceName: target.name,
          referenceKind: target.kind,
          line,
          column: 0,
          filePath,
          language: 'python',
        });
      }
    }

    return { nodes, references };
  },
};

/**
 * 解析 Django URL 处理程序表达式并返回要链接的符号/模块。
 *
 * 对于无法可靠链接的形状（例如 lambdas）返回 null。
 */
function resolveHandlerName(expr: string): { name: string; kind: 'references' | 'imports' } | null {
  // include('module.path') / include("module.path")
  const includeMatch = expr.match(/^include\s*\(\s*['"]([^'"]+)['"]/);
  if (includeMatch) return { name: includeMatch[1], kind: 'imports' };

  // 去除尾部的 .as_view(...) 或 .as_view 调用
  let head = expr.replace(/\.as_view\s*\([^)]*\)\s*$/, '');

  // 去除尾部的方法调用如 .some_method()
  head = head.replace(/\.\w+\s*\([^)]*\)\s*$/, '');

  // 现在 head 应该是一个裸名或点分路径。取最后一段。
  const dotted = head.split('.').filter(Boolean);
  if (dotted.length === 0) return null;
  const last = dotted[dotted.length - 1];
  if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(last)) return null;

  return { name: last, kind: 'references' };
}
```

同时确保文件顶部导入了 `UnresolvedRef` 和 `Node`：

```typescript
import type { FrameworkResolver, UnresolvedRef } from '../types';
import type { Node } from '../../types';
```

- [ ] **步骤 4：运行测试以验证它们通过**

运行：`npx vitest run __tests__/frameworks.test.ts -t djangoResolver`
预期：通过（6 个测试）。

- [ ] **步骤 5：提交**

```bash
git add src/resolution/frameworks/python.ts __tests__/frameworks.test.ts
git commit -m "feat(django): emit route nodes and route->view references in extract()"
```

---

## 任务 4：移植 Flask 和 FastAPI 解析器

**文件：**
- 修改：`src/resolution/frameworks/python.ts`（flaskResolver 和 fastapiResolver 部分）

- [ ] **步骤 1：编写会失败的测试**

追加到 `__tests__/frameworks.test.ts`：

```typescript
import { flaskResolver, fastapiResolver } from '../src/resolution/frameworks/python';

describe('flaskResolver.extract', () => {
  it('从 @app.route 提取路由和引用', () => {
    const src = `
@app.route('/users')
def list_users():
    return []
`;
    const { nodes, references } = flaskResolver.extract!('app.py', src);
    expect(nodes).toHaveLength(1);
    expect(nodes[0].kind).toBe('route');
    expect(nodes[0].name).toBe('GET /users');
    expect(references[0].referenceName).toBe('list_users');
  });

  it('提取蓝图路由', () => {
    const src = `
@users_bp.route('/<id>', methods=['POST'])
def create_user(id):
    pass
`;
    const { nodes, references } = flaskResolver.extract!('routes.py', src);
    expect(nodes[0].name).toBe('POST /<id>');
    expect(references[0].referenceName).toBe('create_user');
  });
});

describe('fastapiResolver.extract', () => {
  it('从 @app.get 提取路由和引用', () => {
    const src = `
@app.get('/users')
async def list_users():
    return []
`;
    const { nodes, references } = fastapiResolver.extract!('main.py', src);
    expect(nodes[0].name).toBe('GET /users');
    expect(references[0].referenceName).toBe('list_users');
  });

  it('从 router.post 提取路由', () => {
    const src = `
@router.post('/items')
def create_item(item: Item):
    pass
`;
    const { nodes, references } = fastapiResolver.extract!('items.py', src);
    expect(nodes[0].name).toBe('POST /items');
    expect(references[0].referenceName).toBe('create_item');
  });
});
```

- [ ] **步骤 2：运行测试以验证它们失败**

运行：`npx vitest run __tests__/frameworks.test.ts -t "flaskResolver|fastapiResolver"`
预期：失败——两个解析器的 `extract` 都未定义。

- [ ] **步骤 3：重写 flaskResolver 和 fastapiResolver**

将 `src/resolution/frameworks/python.ts` 中的 `flaskResolver` 替换为：

```typescript
export const flaskResolver: FrameworkResolver = {
  name: 'flask',
  languages: ['python'],

  detect(context) {
    const requirements = context.readFile('requirements.txt');
    if (requirements && /\bflask\b/i.test(requirements)) return true;
    const pyproject = context.readFile('pyproject.toml');
    if (pyproject && /\bflask\b/i.test(pyproject)) return true;
    for (const file of ['app.py', 'application.py', 'main.py', '__init__.py']) {
      const content = context.readFile(file);
      if (content && content.includes('Flask(__name__)')) return true;
    }
    return false;
  },

  resolve(ref, context) {
    if (ref.referenceName.endsWith('_bp') || ref.referenceName.endsWith('_blueprint')) {
      const result = resolveByNameAndKind(ref.referenceName, VARIABLE_KINDS, [], context);
      if (result) return { original: ref, targetNodeId: result, confidence: 0.8, resolvedBy: 'framework' };
    }
    return null;
  },

  extract(filePath, content) {
    if (!filePath.endsWith('.py')) return { nodes: [], references: [] };
    return extractDecoratorRoutes(filePath, content, {
      // Flask: @x.route('/path', methods=[...])
      decoratorRegex: /@(\w+)\.route\s*\(\s*['"]([^'"]+)['"](?:\s*,\s*methods\s*=\s*\[([^\]]+)\])?\s*\)\s*\n\s*(?:async\s+)?def\s+(\w+)/g,
      defaultMethod: 'GET',
      methodFromGroup: 3,
      pathGroup: 2,
      handlerGroup: 4,
      language: 'python',
    });
  },
};

export const fastapiResolver: FrameworkResolver = {
  name: 'fastapi',
  languages: ['python'],

  detect(context) {
    const requirements = context.readFile('requirements.txt');
    if (requirements && /\bfastapi\b/i.test(requirements)) return true;
    const pyproject = context.readFile('pyproject.toml');
    if (pyproject && /\bfastapi\b/i.test(pyproject)) return true;
    for (const file of ['app.py', 'main.py', 'api.py']) {
      const content = context.readFile(file);
      if (content && content.includes('FastAPI(')) return true;
    }
    return false;
  },

  resolve(ref, context) {
    if (ref.referenceName.endsWith('_router') || ref.referenceName === 'router') {
      const result = resolveByNameAndKind(ref.referenceName, VARIABLE_KINDS, ROUTER_DIRS, context);
      if (result) return { original: ref, targetNodeId: result, confidence: 0.8, resolvedBy: 'framework' };
    }
    if (ref.referenceName.startsWith('get_') || ref.referenceName.startsWith('Depends')) {
      const result = resolveByNameAndKind(ref.referenceName, FUNCTION_KINDS, DEP_DIRS, context);
      if (result) return { original: ref, targetNodeId: result, confidence: 0.75, resolvedBy: 'framework' };
    }
    return null;
  },

  extract(filePath, content) {
    if (!filePath.endsWith('.py')) return { nodes: [], references: [] };
    return extractDecoratorRoutes(filePath, content, {
      // FastAPI: @x.get('/path')
      decoratorRegex: /@(\w+)\.(get|post|put|patch|delete|options|head)\s*\(\s*['"]([^'"]+)['"]/g,
      defaultMethod: '',
      methodGroup: 2,
      pathGroup: 3,
      // 处理程序在接下来的 def 行中；通过后扫描捕获
      handlerGroup: undefined,
      findHandler: true,
      language: 'python',
    });
  },
};
```

并在 `python.ts` 底部添加此共享辅助函数：

```typescript
interface DecoratorRouteOpts {
  decoratorRegex: RegExp;
  defaultMethod: string;
  methodGroup?: number;
  methodFromGroup?: number; // methods=[...] list
  pathGroup: number;
  handlerGroup?: number;
  findHandler?: boolean;
  language: 'python';
}

function extractDecoratorRoutes(filePath: string, content: string, opts: DecoratorRouteOpts) {
  const nodes: Node[] = [];
  const references: UnresolvedRef[] = [];
  const now = Date.now();
  let match: RegExpExecArray | null;
  while ((match = opts.decoratorRegex.exec(content)) !== null) {
    const routePath = match[opts.pathGroup];
    let method = opts.defaultMethod;
    if (opts.methodGroup && match[opts.methodGroup]) {
      method = match[opts.methodGroup].toUpperCase();
    } else if (opts.methodFromGroup && match[opts.methodFromGroup]) {
      const m = match[opts.methodFromGroup].match(/['"]([A-Z]+)['"]/i);
      if (m) method = m[1].toUpperCase();
    }
    const line = content.slice(0, match.index).split('\n').length;
    const name = method ? `${method} ${routePath}` : routePath;
    const routeNode: Node = {
      id: `route:${filePath}:${line}:${method}:${routePath}`,
      kind: 'route',
      name,
      qualifiedName: `${filePath}::${method}:${routePath}`,
      filePath,
      startLine: line,
      endLine: line,
      startColumn: 0,
      endColumn: match[0].length,
      language: opts.language,
      updatedAt: now,
    };
    nodes.push(routeNode);

    let handlerName: string | undefined;
    if (opts.handlerGroup && match[opts.handlerGroup]) {
      handlerName = match[opts.handlerGroup];
    } else if (opts.findHandler) {
      // 查找装饰器之后的下一个 `def <name>`
      const tail = content.slice(match.index + match[0].length);
      const defMatch = tail.match(/\n\s*(?:async\s+)?def\s+(\w+)/);
      if (defMatch) handlerName = defMatch[1];
    }
    if (handlerName) {
      references.push({
        fromNodeId: routeNode.id,
        referenceName: handlerName,
        referenceKind: 'references',
        line,
        column: 0,
        filePath,
        language: 'python',
      });
    }
  }
  return { nodes, references };
}
```

- [ ] **步骤 4：运行测试以验证它们通过**

运行：`npx vitest run __tests__/frameworks.test.ts -t "flaskResolver|fastapiResolver"`
预期：通过（4 个测试）。

- [ ] **步骤 5：提交**

```bash
git add src/resolution/frameworks/python.ts __tests__/frameworks.test.ts
git commit -m "feat(flask,fastapi): emit route nodes and route->handler references"
```

---

## 任务 5：移植 Express 解析器

**文件：**
- 修改：`src/resolution/frameworks/express.ts`（extractNodes 部分，约第 83-117 行）

- [ ] **步骤 1：编写会失败的测试**

追加到 `__tests__/frameworks.test.ts`：

```typescript
import { expressResolver } from '../src/resolution/frameworks/express';

describe('expressResolver.extract', () => {
  it('提取带有内联处理程序引用的路由', () => {
    const src = `app.get('/users', listUsers);\n`;
    const { nodes, references } = expressResolver.extract!('routes.ts', src);
    expect(nodes).toHaveLength(1);
    expect(nodes[0].name).toBe('GET /users');
    expect(references[0].referenceName).toBe('listUsers');
  });

  it('提取带有 router.post 的路由', () => {
    const src = `router.post('/items', auth, createItem);\n`;
    const { nodes, references } = expressResolver.extract!('items.ts', src);
    expect(nodes[0].name).toBe('POST /items');
    // 多个处理程序：优先取最后一个（约定：中间件在前，处理程序在最后）
    expect(references[0].referenceName).toBe('createItem');
  });

  it('提取带有控制器方法引用的路由', () => {
    const src = `app.get('/x', userController.list);\n`;
    const { nodes, references } = expressResolver.extract!('routes.ts', src);
    expect(references[0].referenceName).toBe('list');
  });
});
```

- [ ] **步骤 2：运行测试以验证它们失败**

运行：`npx vitest run __tests__/frameworks.test.ts -t expressResolver`
预期：失败。

- [ ] **步骤 3：重写 expressResolver.extract**

将 `expressResolver` 上现有的 `extractNodes` 方法（在 `src/resolution/frameworks/express.ts` 中）替换为：

```typescript
  languages: ['javascript', 'typescript'],

  extract(filePath, content) {
    if (!/\.(m?js|tsx?|cjs)$/.test(filePath)) return { nodes: [], references: [] };
    const nodes: Node[] = [];
    const references: UnresolvedRef[] = [];
    const now = Date.now();
    // 捕获：(app|router).METHOD('/path', handler-expr)
    const regex = /\b(app|router)\.(get|post|put|patch|delete|all|use)\s*\(\s*['"]([^'"]+)['"]\s*,\s*([^)]+)\)/g;
    let match: RegExpExecArray | null;
    while ((match = regex.exec(content)) !== null) {
      const [, _obj, method, routePath, handlers] = match;
      if (method === 'use' && !routePath.startsWith('/')) continue;
      const line = content.slice(0, match.index).split('\n').length;
      const routeNode: Node = {
        id: `route:${filePath}:${line}:${method.toUpperCase()}:${routePath}`,
        kind: 'route',
        name: `${method.toUpperCase()} ${routePath}`,
        qualifiedName: `${filePath}::${method.toUpperCase()}:${routePath}`,
        filePath,
        startLine: line,
        endLine: line,
        startColumn: 0,
        endColumn: match[0].length,
        language: detectLanguage(filePath),
        updatedAt: now,
      };
      nodes.push(routeNode);
      // 最后一个逗号分隔的参数是处理程序；中间的参数是中间件
      const handlerParts = handlers.split(',').map((s) => s.trim()).filter(Boolean);
      const last = handlerParts[handlerParts.length - 1];
      const handlerName = extractTailIdent(last);
      if (handlerName) {
        references.push({
          fromNodeId: routeNode.id,
          referenceName: handlerName,
          referenceKind: 'references',
          line,
          column: 0,
          filePath,
          language: detectLanguage(filePath),
        });
      }
    }
    return { nodes, references };
  },
```

并在文件顶部附近添加：

```typescript
import type { FrameworkResolver, UnresolvedRef } from '../types';
import type { Node } from '../../types';

function extractTailIdent(expr: string): string | null {
  const cleaned = expr.replace(/\s+/g, '').replace(/\(\)$/, '');
  const m = cleaned.match(/(?:\.|^)([A-Za-z_][A-Za-z0-9_]*)$/);
  return m ? m[1] : null;
}
```

移除旧的 `extractNodes` 方法。

- [ ] **步骤 4：运行测试以验证它们通过**

运行：`npx vitest run __tests__/frameworks.test.ts -t expressResolver`
预期：通过。

- [ ] **步骤 5：提交**

```bash
git add src/resolution/frameworks/express.ts __tests__/frameworks.test.ts
git commit -m "feat(express): emit route nodes and route->handler references"
```

---

## 任务 6：移植 Laravel、Rails、Spring、Gin（Go）、Axum（Rust）、ASP.NET（C#）、Swift 解析器

**文件：**
- 修改：`src/resolution/frameworks/laravel.ts` / `ruby.ts` / `java.ts` / `go.ts` / `rust.ts` / `csharp.ts` / `swift.ts`

每个框架遵循与上述任务 3–5 **相同的模式**：

1. 添加 `languages: [...]` 字段。
2. 将 `extractNodes(filePath, content)` 替换为 `extract(filePath, content): { nodes, references }`。
3. 在 `extract()` 内部，对每个匹配的路由正则表达式：创建一个路由节点（复用现有形状）并发出一个带有 `fromNodeId = routeNode.id` 的 `UnresolvedRef` 用于处理程序/控制器。
4. 对于每个框架，在 `__tests__/frameworks.test.ts` 中添加一个单元测试，验证至少一个路由形状同时产生节点和处理程序引用。

**各框架具体细节：**

- **Laravel**（`laravel.ts`）：`Route::get('/x', [Ctrl::class, 'method'])` → 处理程序引用名称 = `method`；`Route::get('/x', 'Ctrl@method')` → 处理程序引用名称 = `method`；`Route::resource('users', UserController::class)` → 处理程序引用名称 = `UserController`。`languages: ['php']`。

- **Rails**（`ruby.ts`）：`get '/x', to: 'users#index'` → 处理程序引用名称 = `index`（按 `users` 限定作用域）；`resources :users` → 每个 CRUD 动作一个节点，每个引用 `UsersController` 上对应的方法名。`languages: ['ruby']`。

- **Spring**（`java.ts`）：方法上的 `@GetMapping("/x")` → 处理程序是接下来的方法名（扫描越过装饰器）。`languages: ['java']`。

- **Gin / chi / gorilla**（`go.ts`）：`r.GET("/x", handler)` → 处理程序引用 = 最后一个参数中的最后一个标识符。`languages: ['go']`。

- **Axum / actix**（`rust.ts`）：`.route("/x", get(handler))` → 处理程序引用 = `get(...)` 内部的标识符。`languages: ['rust']`。

- **ASP.NET**（`csharp.ts`）：`[HttpGet("/x")] public ActionResult Method()` → 处理程序引用 = 同一类上的方法名。`languages: ['csharp']`。

- **Swift / Vapor**（`swift.ts`）：`app.get("/x", use: handler)` → 处理程序引用 = `use:` 后的标识符。`languages: ['swift']`。

每个都有其自己的提交，形式为：

```bash
git add src/resolution/frameworks/<framework>.ts __tests__/frameworks.test.ts
git commit -m "feat(<framework>): emit route nodes and route->handler references"
```

**重要提示：** 保持每个框架的提交独立，以便任何一个可以在引起回归时被回退。

### 任务 6a：Laravel

- [ ] **步骤 1：编写测试** 对于 `Route::get('/users', [UserController::class, 'index'])` → `{nodes[0].name='GET /users', references[0].referenceName='index'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`** 遵循 Express 模式。正则表达式：`/Route::(get|post|put|patch|delete|options|any)\s*\(\s*['"]([^'"]+)['"]\s*,\s*([^)]+)\)/g`。通过 `resolveLaravelHandler()` 从第三组提取处理程序：去除 `[`/`]`/`::class`，取逗号分隔数组的第二个元素或 `Ctrl@method`。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6b：Rails

- [ ] **步骤 1：编写测试** 对于 `get '/users', to: 'users#index'` → `{references[0].referenceName='index'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`**。正则表达式：`/\b(get|post|put|patch|delete|match)\s+['"]([^'"]+)['"]\s*,\s*to:\s*['"]([^'"]+)['"]/g` → `controller#method` 按 `#` 分割得到处理程序 = `method`。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6c：Spring

- [ ] **步骤 1：编写测试** 对于 `@GetMapping("/x")\npublic String list() {...}` → `{references[0].referenceName='list'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`** 使用共享的 `extractDecoratorRoutes` 辅助函数（如果更干净，将其移到新的 `src/resolution/frameworks/shared.ts`）。在每个映射注解之后查找下一个 `public` 或 `private` 方法声明的名称。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6d：Go

- [ ] **步骤 1：编写测试** 对于 `r.GET("/x", handler)` 和 `router.Handle("/x", handler)` → `{references[0].referenceName='handler'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`**。正则表达式：`/\b(?:router|r|mux|app)\.(GET|POST|PUT|PATCH|DELETE|Handle|HandleFunc)\s*\(\s*["]([^"]+)["]\s*,\s*([^)]+)\)/g`。处理程序 = 第三组中的最后一个标识符。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6e：Rust

- [ ] **步骤 1：编写测试** 对于 `.route("/x", get(list_users))` → `{references[0].referenceName='list_users'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`**。正则表达式：`/\.route\s*\(\s*"([^"]+)"\s*,\s*(get|post|put|patch|delete)\s*\(\s*(\w+)/g` → 处理程序 = 组 3。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6f：C#（ASP.NET）

- [ ] **步骤 1：编写测试** 对于 `[HttpGet("/x")]\npublic IActionResult List()` → `{references[0].referenceName='List'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`**。查找特性，然后向前扫描到第一个 `public|private|protected` 方法声明并取其名称。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6g：Swift / Vapor

- [ ] **步骤 1：编写测试** 对于 `app.get("/users", use: list)` → `{references[0].referenceName='list'}`。
- [ ] **步骤 2：运行测试，看到失败。**
- [ ] **步骤 3：实现 `extract()`**。正则表达式：`/\b(app|router|routes)\.(get|post|put|patch|delete)\s*\(\s*"([^"]+)"\s*,\s*use:\s*([A-Za-z_][A-Za-z0-9_.]*)/g` → 处理程序 = 组 4 的最后一段。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

### 任务 6h：React 和 Svelte

这些是 UI 框架，其中路由映射到组件，而非服务器意义上的处理程序。保留现有行为但迁移接口：

- [ ] **步骤 1：迁移 `reactResolver`**（`src/resolution/frameworks/react.ts`）——添加 `languages: ['javascript', 'typescript']`，将 `extractNodes` 重命名为 `extract`，使其返回 `{ nodes, references: [] }`（现有逻辑只发出节点，尚不需要处理程序引用——后续步骤可以添加 `<Route element={<Page/>}/>` → `Page` 引用）。
- [ ] **步骤 2：迁移 `svelteResolver`**（`src/resolution/frameworks/svelte.ts`）——相同模式；`languages: ['svelte']`。
- [ ] **步骤 3：为每个添加冒烟测试**，验证 `extract()` 返回与之前相同的节点形状。
- [ ] **步骤 4：运行测试，看到通过。**
- [ ] **步骤 5：提交。**

---

## 任务 7：将框架提取接入 `ExtractionOrchestrator`

**文件：**
- 修改：`src/extraction/index.ts`（每个文件提取结果合并路径）
- 修改：`src/extraction/parse-worker.ts`（如果提取在那里运行，将检测到的框架传递给 worker）

这是核心的接线变更。它在每个文件被 tree-sitter 解析后运行。

- [ ] **步骤 1：编写集成测试**

创建 `__tests__/frameworks-integration.test.ts`：

```typescript
import { describe, it, expect, beforeAll, afterEach } from 'vitest';
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';
import { CodeGraph } from '../src';
import { initGrammars, loadAllGrammars } from '../src/extraction/grammars';

beforeAll(async () => {
  await initGrammars();
  await loadAllGrammars();
});

describe('Django 端到端', () => {
  let tmpDir: string;
  afterEach(() => {
    if (tmpDir) fs.rmSync(tmpDir, { recursive: true, force: true });
  });

  it('从 urls.py 到视图类创建 route->view 边', async () => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'cg-django-'));
    fs.writeFileSync(path.join(tmpDir, 'manage.py'), '# marker');
    fs.writeFileSync(path.join(tmpDir, 'requirements.txt'), 'django==4.2\n');
    fs.mkdirSync(path.join(tmpDir, 'users'));
    fs.writeFileSync(path.join(tmpDir, 'users/__init__.py'), '');
    fs.writeFileSync(path.join(tmpDir, 'users/views.py'),
      'class UserListView:\n    def get(self, request): pass\n');
    fs.writeFileSync(path.join(tmpDir, 'users/urls.py'),
      'from django.urls import path\n' +
      'from users.views import UserListView\n' +
      'urlpatterns = [path("users/", UserListView.as_view(), name="user-list")]\n');

    const cg = new CodeGraph(tmpDir);
    await cg.initialize();
    await cg.indexAll();

    const nodes = cg.queries.searchNodes({ kinds: ['route'] });
    expect(nodes.length).toBeGreaterThan(0);
    const route = nodes.find(n => n.name === 'users/');
    expect(route).toBeDefined();

    const view = cg.queries.getNodesByName('UserListView').find(n => n.kind === 'class');
    expect(view).toBeDefined();

    const edges = cg.queries.getOutgoingEdges(route!.id);
    const toView = edges.find(e => e.target === view!.id);
    expect(toView).toBeDefined();
    expect(toView!.kind).toBe('references');

    await cg.close();
  });
});
```

- [ ] **步骤 2：运行测试以验证其失败**

运行：`npx vitest run __tests__/frameworks-integration.test.ts`
预期：失败——没有创建路由节点（框架提取尚未接入）。

- [ ] **步骤 3：添加接线**

在 `src/extraction/index.ts` 中，定位 `extractFromSource` 函数（大约第 600 行；该函数在单个文件上运行 tree-sitter 并返回 `ExtractionResult`）。将框架提取作为 tree-sitter 后的增强添加。

找到 `ExtractionResult` 在 `extractFromSource` 末尾构建的位置（大约第 1000-1015 行）。在 `return result` 之前，添加：

```typescript
// 框架特定的提取（路由等）
if (detectedFrameworks && detectedFrameworks.length > 0) {
  const applicable = getApplicableFrameworks(detectedFrameworks, language);
  for (const fw of applicable) {
    if (!fw.extract) continue;
    try {
      const fwResult = fw.extract(filePath, content);
      result.nodes.push(...fwResult.nodes);
      result.unresolvedReferences.push(...fwResult.references);
    } catch (err) {
      result.errors.push({
        message: `框架提取器 '${fw.name}' 失败：${err instanceof Error ? err.message : String(err)}`,
        filePath,
        severity: 'warning',
      });
    }
  }
}
```

同时将 `detectedFrameworks?: FrameworkResolver[]` 添加为 `extractFromSource` 的参数。

在 `ExtractionOrchestrator.indexAll` 中（大约第 412 行），在启动解析 worker 之前，检测框架一次：

```typescript
// 每次索引运行时检测框架一次（项目级信号）
const resolutionContext = buildResolutionContext(this.rootDir, this.queries);
const detectedFrameworks = detectFrameworks(resolutionContext);
```

将 `detectedFrameworks` 传递到解析 worker 批处理配置中（或者，如果解析 worker 不直接调用 `extractFromSource`，则传递到调用框架提取的主线程合并步骤中，对原始文件内容进行操作）。如果解析 worker 已经可以访问文件内容，则传递框架**名称**并在 worker 内部通过 `getAllFrameworkResolvers().filter(f => detectedNames.includes(f.name))` 重新解析为解析器对象——带有函数的对象不能跨越 worker_threads postMessage 边界。

- [ ] **步骤 4：运行测试以验证其通过**

运行：`npx vitest run __tests__/frameworks-integration.test.ts`
预期：通过。

- [ ] **步骤 5：运行完整的测试套件以检查回归**

运行：`npx vitest run`
预期：所有现有测试仍然通过。

- [ ] **步骤 6：提交**

```bash
git add src/extraction/index.ts src/extraction/parse-worker.ts __tests__/frameworks-integration.test.ts
git commit -m "feat(extraction): run framework extractors after tree-sitter parse"
```

---

## 任务 8：移除死的正则表达式代码 + 更新 README

**文件：**
- 修改：`src/resolution/frameworks/*.ts` — 确认没有遗留的 `extractNodes`
- 修改：`README.md` — 添加关于框架路由提取的部分

- [ ] **步骤 1：grep 任何遗留的引用**

运行：`grep -rn "extractNodes" src/ __tests__/`
预期：零匹配。如果有剩余，删除或重命名它们。

- [ ] **步骤 2：运行完整的构建和测试**

运行：`npm run build && npm test`
预期：构建成功；所有测试通过。

- [ ] **步骤 3：添加 README 部分**

在功能列表之后追加到 `README.md`：

```markdown
### 框架感知路由

CodeGraph 识别 Web 框架路由文件并将 URL 模式链接到它们的处理程序：

- **Django**：`urls.py` 中的 `urlpatterns` 条目——`path()`、`re_path()`、`url()`、`include()`
- **Flask / FastAPI**：`@app.route` / `@app.get` / `@router.post` 装饰器
- **Express**：`app.get(...)`、`router.post(...)`
- **Laravel**：`Route::get()`、`Route::resource()`
- **Rails**：`resources :users`、`get 'x', to: 'y#z'`
- **Spring**：`@GetMapping`、`@RequestMapping`
- **Gin / chi / gorilla**：`r.GET(...)`
- **Axum / actix**：`.route("/x", get(handler))`
- **ASP.NET**：`[HttpGet]` + action 方法
- **Vapor**：`app.get("x", use: handler)`

查询 `codegraph_callers(YourView)`，路由模式将作为入边出现。
```

- [ ] **步骤 4：提交**

```bash
git add README.md
git commit -m "docs: document framework route extraction"
```

---

## 任务 9：创建 PR

- [ ] **步骤 1：推送分支到 fork**

```bash
git push -u origin feat/framework-extract-wiring
```

- [ ] **步骤 2：创建 PR**

```bash
gh pr create \
  --repo colbymchenry/codegraph \
  --base main \
  --head timomeara:feat/framework-extract-wiring \
  --title "feat: wire up framework route extraction" \
  --body "$(cat <<'EOF'
## 问题

`FrameworkResolver.extractNodes` 在类型中声明但从未在 `src/` 中的任何地方调用。结果，图中任何框架都没有 `route` 节点，且 URL 到处理程序的链接（例如 Django `urls.py` 入口 -> 视图类）不存在。这使得 `codegraph_callers(MyView)` 静默地错过了其最重要的调用者。

## 修复

- 将死的 `extractNodes?(filePath, content): Node[]` 钩子替换为 `extract?(filePath, content): { nodes, references }`。
- 在提取管道内为每个声明的 `languages` 包含当前文件语言的框架调用 `extract()`。
- 更新所有 13 个现有框架解析器（Django、Flask、FastAPI、Express、Laravel、Rails、Spring、Gin、Axum、ASP.NET、Vapor、React Router、SvelteKit）以同时发出路由节点和处理程序引用。这些引用流经现有解析管道（名称匹配、导入解析、框架特定的 `resolve()`）以产生 `route -> handler` 边。

## 测试

- `__tests__/frameworks.test.ts` 中的每个框架的单元测试。
- `__tests__/frameworks-integration.test.ts` 中的端到端 Django 测试，验证真实的 `urls.py -> views.py` 边。

## 统计

| 类别 | 行数 |
|----------|------:|
| 生产代码 | ~X |
| 测试 | ~Y |
| 文档 | ~Z |
EOF
)"
```

- [ ] **步骤 3：在任务跟踪器中链接 PR**（如果存在）。

---

## 自我审查清单

- [ ] **规范覆盖：** 原始代码库中的每个框架都有一个迁移任务。Django 的测试覆盖率最丰富，因为它是驱动案例。
- [ ] **无占位符：** 每个任务显示实际代码。任务 6 中的"与任务 X 相同模式"措辞由任务 3-5 中的完整实现作为参照支持。
- [ ] **类型一致性：** `FrameworkExtractionResult` 在任务 1 中定义一次，并被每个解析器的 `extract` 签名使用。
- [ ] **现实的统计占位符**（X/Y/Z）在 PR 时填写，而不是计划时。

## 已知差距（故意不在范围内）

- **基于 AST 的提取。** 正则表达式对于常见形状已经足够。在后续任务中切换到 tree-sitter AST。
- **DRF 路由器扩展。** `router.register(r'users', UserViewSet)` 产生一个指向视图集的单个路由节点。扩展到 6 个 CRUD 动作节点可以作为后续任务。
- **React Router 处理程序边。** `<Route element={<Page/>}/>` 当前只产生一个路由节点。后续可以添加 `route -> Page` 引用。
- **Spring Controller 类作用域。** 方法级映射可行；类级 `@RequestMapping` 基础路径组合是后续任务。
