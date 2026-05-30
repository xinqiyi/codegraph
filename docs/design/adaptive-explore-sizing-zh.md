# 设计 + 状态：自适应 `codegraph_explore` 大小调整（兄弟节点骨架化）

**状态：** 已实现并验证，**默认启用**，在分支 `feat/adaptive-explore-sizing` 上（初始提交 `d6d059f`；**2026-05-29 精炼**——在真实 Agent A/B 测试暴露出回读回归后——参见下面的"精炼"）。逃生口：`CODEGRAPH_ADAPTIVE_EXPLORE=0`。
**动机：** 使 `codegraph_explore` 的输出大小根据*答案*而非始终填满预算上限来调整——这样"兄弟节点密集"的流（一个接口的许多可互换实现）不再比普通的 grep/read 成本更高，同时不会让真正需要广泛源代码的"分散"流挨饿。

> **精炼（2026-05-29）——回读回归。** 初始版本仅基于*非主线 + 多态兄弟节点*来门控。真实 Agent A/B（非确定性探测）显示，这骨架化了两个文件，然后代理**回读**了这些文件，这与目的相悖：OkHttp 的 `RealCall`（它实现了有 9 个实现的 `Lockable` *mixin*，因此触发了兄弟节点信号，即使它是编排器）和 Django 的 `compiler.py`（它*定义了* `SQLCompiler` 并与其子类共置）。两个条件解决了这个问题——一个文件只有在**未被豁免**时才会被骨架化，其中**豁免 = 代理在其中命名了一个可调用对象**（`getResponseWithInterceptorChain`、`SQLCompiler.execute_sql` → 保持完整）**除非该文件定义了一个有 ≥3 个实现的超类型**（一个基类+子类"家族"文件很大，反正会被 Read，因此骨架化它*释放了探索预算*用于代理否则会 Read 的兄弟文件）。结果：OkHttp **成本增加 3% → 成本降低约 10%**（RealCall 完整，0 次回读）；Django **成本增加 10% → 成本降低约 10%**（compiler.py 骨架化释放了 28 KB 预算中的约 6.5 KB；一半的运行的答案中读取次数为 0）。超类型信号最初被用作*豁免*——这是倒退的，使 Django 因预算不足而成本增加 9%；现在它是对命名可调用对象豁免的*覆盖*。下面的单一条件历史被保留以供参考。

---

## TL;DR（太长不读）

`codegraph_explore` 为每个相关文件返回完整的源代码，直到达到其字符预算。对于一个答案涉及许多*同形状*类的问题——例如"OkHttp 如何通过其拦截器链处理请求？"，这涉及约 14 个 `class … : Interceptor` 实现——这意味着约 28 KB 主要是**冗余的完整主体**。因为这些主体在会话的剩余时间中占据着上下文窗口，所以 WITH-CodeGraph 臂的成本*高于* WITHOUT 臂（后者通过约 10 次廉价的 grep 就能回答这个命名良好的拦截器问题）。OkHttp 是该基准测试中的成本异常值（−3%——即*比*原生搜索*更贵*）。

修复：当一个文件**既 (a) 在合成流程主线之外，又 (b) 是多态兄弟节点**时，将其渲染为**骨架**（类 + 成员*签名*，省略主体）而不是完整源代码——保持主线上的示例和机制完整。

- **OkHttp：** 拦截器链流程将 5 个冗余的 `: Interceptor` 实现骨架化，同时保持 `RealInterceptorChain`（调度机制）和 `RealCall`（代理命名的编排器）完整 → **比原生便宜约 10%，0 次 RealCall 回读**（参见精炼中的更正数字；原始的 `28.5k → 16.6k` / "读取次数 1 vs 3" 的数据来自确定性探测查询，而非代理的真实查询）。
- **Django：** QuerySet→SQL 流程骨架化 `compiler.py`（一个基类+子类家族文件），释放预算 → **成本降低约 10%**。（之前声称 Django 是"字节相同 / 0 骨架"是*探测*查询的产物；代理的真实查询确实会浮现 SQLCompiler 家族。）
- **Excalidraw / Tokio / VS Code / Gin：** 在标志开启/关闭时，探索输出**字节相同**（0 个骨架）——它们的流程没有非主线的 ≥3 个实现者的兄弟组。修正后的门控只*添加*了一个豁免条件，因此它骨架化的是原始门控的**严格子集** → 这些代码库被证明保持 0 个骨架（已通过探测验证）。

---

## 问题一目了然

`handleExplore` 收集相关文件，按相关性排序，并填充到 `maxOutputChars`（"完整小文件规则"将任何相关且 ≤220 行的文件完整输出）。预算是一个**目标**，而非上限：

```
OkHttp 探索（已发布）：RealCall（完整）+ RealInterceptorChain（完整）
                        + CallServerInterceptor（完整，8.7k）
                        + Bridge/Connect/Cache/…（完整，每个约 4-5k）   ← 都~同形状
                        = 约 28k，其中大部分是冗余的拦截器主体
```

代理只需要**机制**（`RealInterceptorChain.proceed` 遍历链）+ 每个拦截器实现的**契约** + 可能一个具体的例子。其他五个完整主体是填充——但仅仅*因为它们是可以互换的*。对于分散的问题（Excalidraw 的渲染管线：`mutateElement → … → renderStaticScene`），非主线文件是**不同的步骤**，它们的主体做实际工作——省略它们只会让代理从签名重新构建它们（更多的推理，最终成本更高；参见"死胡同"）。

所以整个游戏就是：**便宜地将"可互换的兄弟节点"与"不同的步骤"区分开来。**

## 门控（精炼版）

一个文件被骨架化当且仅当**全部**条件满足（且 `CODEGRAPH_ADAPTIVE_EXPLORE != 0`）：

1. **存在一条主线。** `buildFlowFromNamedSymbols` 返回其路径节点集（`pathNodeIds`）和代理命名的可调用对象的完整集合（`namedNodeIds`）。如果没有形成主线，则不会发生骨架化。

2. **不在流程主线上。** 文件中的没有符号在追踪链上——该链是代理正在遍历的机制，始终保持完整。

3. **是一个多态兄弟节点。** 文件的类 `implements`/`extends` 一个具有 **≥ 3 个实现者**（`MIN_SIBLINGS`）的超类型——表明它是许多*可互换*实现之一的信号。来自真实的 `implements`/`extends` 边，已缓存。

4. **未被豁免。** 一个文件被**豁免**（保持完整）当且仅当代理在其中**命名了一个可调用对象**——一个被命名的参数/函数是代理要求*查看*的东西（`getResponseWithInterceptorChain`、`SQLCompiler.execute_sql`），而不是一个可互换的叶子——**除非该文件本身定义了一个 ≥3 实现的超类型**。最后这个子句是覆盖规则：一个基类+子类"家族"文件（Django 的 `compiler.py`）很大，反正会被 Read，因此保留完整副本只会消耗探索预算；骨架化它*释放*了该预算用于代理否则会 Read 的兄弟文件。所以：*已命名 ⇒ 豁免，除非它是家族文件 ⇒ 无论如何都骨架化。*

通过两个代码库来验证：

- **`RealInterceptorChain`** — `proceed` 在主线上 → 保持完整（条件 2）。
- **`RealCall`** — 不在主线上，并且通过 **9 个实现的 `Lockable` mixin** 触发兄弟节点信号（不是因为它是可互换的拦截器）。但代理在其中命名了 `getResponseWithInterceptorChain`/`execute`/`enqueue`，并且它没有定义 ≥3 实现的超类型 → **豁免，保持完整**（条件 4）。这就是回读问题的修复：在条件 4 之前它被骨架化了，代理将其回读。
- **`BridgeInterceptor` 及其他 4 个** — 不在主线上，≥3 个兄弟节点实现，仅通过*类型*命名，没有定义超类型 → **骨架化**。这是胜利。
- **Django `compiler.py`** — 不在主线上，是一个兄弟节点（其子类扩展了 `SQLCompiler`），代理在其中命名了 `execute_sql` — *但它定义了 `SQLCompiler` 超类型*，因此覆盖规则触发 → **骨架化**（释放预算）。反而对其进行豁免（错误的第一次尝试）导致了更多的成本和更多的读取。

## 为什么"共享超类型具有 ≥3 个实现者"是信号

使 OkHttp 的拦截器可互相互换的正是它们是**一个接口的 N 个实现**，以多态方式被调用。这是一个*结构性的*属性，图通过 `implements`/`extends` 边记录：

```
14 个类 ──implements──▶ Interceptor      (BridgeInterceptor、CacheInterceptor、
                                           CallServerInterceptor、… )
```

Excalidraw 的 `renderStaticScene`、`Scene`、`Collab` **没有**共同的超类型——≥3 实现者的查询对它们返回空。因此该信号干净地将两个代码库分开，并且（在下文中验证）保持每个非兄弟节点流不变。

`≥ 3` 阈值很重要：1:1 的"服务接口 → 单一实现"对（常见的 Spring/Java 形状）**不是**兄弟节点，保持完整。只有真正的多实现家族（拦截器链、策略/访问者家族、编解码器注册表）触发门控。

## 骨架渲染

对于骨架化的文件，我们输出类 + 成员**签名行**（而非主体）。因为符号节点的 `startLine` 可以指向装饰器/注解（`@Throws`、`@Override`、`@objc`），我们向前扫描最多 4 行以找到实际*命名*符号的行，以便骨架显示真正的签名：

```
#### …/CallServerInterceptor.kt — CallServerInterceptor、intercept、… · skeleton（仅签名；使用 Read 获取完整主体）
```kotlin
30  object CallServerInterceptor : Interceptor {
32  override fun intercept(chain: Interceptor.Chain): Response {
194 private fun shouldIgnoreAndWaitForRealResponse(code: Int): Boolean =
```
```

头部仍然列出文件的符号并显示 `Read for a full body`，这样代理可以在真正需要时拉取特定的实现。

## 验证（精炼门控）

无头 `claude -p`，Opus 4.8，**WITH vs WITHOUT** CodeGraph（真正的基准测试臂，而非初始版本使用的开关探测）。成本 = 中值 `total_cost_usd`。

| 代码库 | WITH→WITHOUT 成本 | WITH 读取次数 | WITHOUT 读取次数 | RealCall/compiler 回读 |
|---|---|---|---|---|
| **OkHttp** (n=4) | **$0.45 → $0.50**（便宜约 10%） | 2 | 3.5 | **0 / —**（RealCall 完整） |
| **Django** (n=6) | **$0.56 → $0.63**（便宜约 10%） | 2 | 8.5 | 一半的运行读取 0 次 |

两者都是 README 中的**成本异常值**（OkHttp 贵 3%，Django 贵 10%），两者都转变为明确的胜利。OkHttp WITH 在所有 4 次运行中都更便宜；Django 在 6 次中的 5 次中（n=6 以观察其高方差）。WITHOUT 基线匹配 README（$0.50/$0.63 对比 $0.57/$0.64），因此收益来自 WITH 臂的改进。

**决定性的检查现在以正确的原因通过**：使用命名可调用对象豁免，OkHttp 的 `RealCall` 保持完整，且**从未**被回读（在修复前，它在 3/4 的运行中被回读）。惰性代码库（Excalidraw / Tokio / VS Code / Gin）保持 **0 个骨架**——已通过探测验证——因为精炼的门控骨架化的是原始门控的严格子集。（初始版本的"开启 vs 关闭，读取次数持平 1 vs 3"的说法来自确定性探测查询，并且**不**适用于代理的真实查询——这种不匹配正是这次精炼所要纠正的。）

## 死胡同（不要再尝试这些）

1. **降级/排名低价值文件**（例如扩宽 `isLowValuePath` 以丢弃 `*-testing-support/` 中的测试夹具）。改善了*内容质量*但**不改善大小**——探索用其他完整主体重新填充释放的预算（28,478 → 28,424）。排名 ≠ 收缩；你必须*骨架化*才能收缩。
2. **按入口节点成员资格门控。** 一个精确的符号包探索查询*命名*了链中的每个参与者，因此它们都是"入口节点"——没有分离，什么也骨架化不了。
3. **依赖接口实现合成器边**（`synthesizedBy:'interface-impl'`）作为兄弟节点信号。它们**没有**为 OkHttp 的 `Interceptor`（一个 Kotlin `fun interface`）创建，因此信号必须来自真实的 `implements`/`extends` 边，而非合成边。
4. **简单的"核心底层"门控**（保留前 N 个完整，骨架化其余部分）——骨架化了 Excalidraw 的*不同*步骤 → **+17% 成本回归**。兄弟节点条件才是使其安全的原因。
5. **因为文件定义了超类型而豁免它**（第一次精炼尝试）。反向：一个基类+子类*家族*文件（Django 的 `compiler.py`，2,266 行）很大，反正会被 Read，因此保留完整只是**消耗了 28 KB 的探索预算并让兄弟文件挨饿**，然后代理会 Read 这些文件——它使 Django **成本增加 9%**（$0.71）。定义超类型反而是一个**覆盖**规则，允许命名的家族文件无论如何都被骨架化。
6. **仅使用确定性探测查询验证骨架化。** 探测（`probe-explore.mjs "<symbol bag>"`）和*代理*的真实探索查询命名符号的方式不同，因此它们形成不同的主线并骨架化不同的文件。探测说"Django：0 个骨架 / 读取次数持平"；真实代理查询骨架化了 `compiler.py` 并回读了它。**始终使用真实代理 A/B（`run-all.sh`）确认，而不仅仅是探测。**

## 代码

- `src/mcp/tools.ts`
  - `adaptiveExploreEnabled()` — 标志（默认开启）。
  - `buildFlowFromNamedSymbols()` — 返回 `{ text, pathNodeIds, namedNodeIds }`。`namedNodeIds` 是代理命名的每个可调用对象（主线的超集）——命名可调用对象豁免读取它。
  - `handleExplore()` — 两个缓存的辅助函数：`isPolymorphicSibling()`（一个节点有指向 ≥3 实现超类型的出向 `implements`/`extends`）和 `definesPolymorphicSupertype()`（一个节点有 ≥3 个入向 `implements`/`extends`——即该文件是家族基类）。骨架化分支：`非主线 && 是多态兄弟节点 && !(文件中已命名 && 未定义超类型)`。
- `__tests__/adaptive-explore-sizing.test.ts` — 7 个测试用例，包括命名可调用对象豁免（RealCall）和超类型家族覆盖（compiler.py）。

## 前沿 / 未来工作

- **家族文件内的逐符号骨架化。** `compiler.py` 被整体骨架化，因此 `SQLCompiler.execute_sql`（基类机制）也变成了签名，在大约一半的 Django 运行中被*回读*。理想情况下，保持基类的方法完整，只省略冗余的子类主体——在不省略答案的情况下缩小负载。整体文件骨架化还不能表达这一点。
- **大的非兄弟文件主导 Django 的残余读取。** `query.py`（3,040 行）和 `sql/query.py` 不是多态家族，因此骨架化无法触及它们；当 28 KB 的聚类视图不足时，代理会 Read 它们。这是探索预算/大文件聚类的前沿，而非骨架化。
- **非接口兄弟家族**（Go `HandlerFunc` 切片、函数指针注册表）没有被捕获——它们没有 `implements`/`extends` 边。Gin 的中间件链，例如，不会触发门控（其处理器是函数，而非接口实现）。
- **示例选择**当*没有*拦截器在主线上时：今天所有兄弟节点都被骨架化，代理依赖接口契约；展示一个作为强制示例可能会读起来稍好一些（未测试）。
