---
name: typescript-best-practices
description: TypeScript 最佳实践。在读取或编辑任何 .ts 或 .tsx 文件时使用。
paths: ["**/*.ts", "**/*.tsx"]
disable-model-invocation: true
---

# TypeScript 最佳实践

先应用 **type-system-discipline** 原则 skill。

| 规则 | 摘要 |
|------|---------|
| 可辨识联合（Discriminated unions） | 用 `kind` 字面量判别项来建模变体，让不可能的状态无法被表示。不要用可选字段大杂烩。 |
| 品牌类型（Branded types） | 用 `& { readonly __brand: "X" }` 给原始类型打品牌，让它们不会被混用。在边界处验证一次。品牌来自验证型类型守卫（`input is AgentId`），绝不来自 `as`。 |
| 构造式建模（Constructive modeling） | 构造出非法值无法被创建的形状。`[T, ...T[]]` 表示非空，`[T, T][]` 表示偶数长度，`start` 加 `duration` 表示一个区间。不是运行时守卫，也不是对细化类型的愿望。 |
| 最简单全函数类型（Simplest total type） | 只要对 `T[]` 的每一次操作都是全函数（total），就保留 `T[]`。只有当松散类型迫使你写出 `!`、类型断言或"这绝不会发生"的抛出时，才加强为 `NonEmpty<T>`。 |
| `unknown` 优于 `any` | 外部数据是 `unknown`。 |
| schema 优于守卫 | 在手写逐属性的类型守卫之前，使用仓库的运行时 schema 库，并从 schema 推断类型，比如 `z.infer`。 |
| 不用 `as` 断言 | 每一个 `as` 都是等着发生的运行时崩溃。只在验证之后才断言。 |
| 收窄层级（Narrowing hierarchy） | 判别项 switch > `in` 操作符 > `typeof`/`instanceof` > 用户定义类型守卫 > `as`。 |
| 类型守卫 | 必须验证其断言。说谎的守卫比 `as` 更糟，因为 bug 藏在一个声称安全的名字后面。把它们命名为 `isX` 或 `hasX`。 |
| 穷尽性（Exhaustiveness） | 在 default 分支里内联 `const _exhaustive: never = x;`，这样新增变体时编译器就会报错。 |
| `satisfies` 优于 `as` | 在不拓宽字面量类型的情况下验证值。 |
| 边界验证（Boundary validation） | 在数据进入的地方解析，转成一个具名的领域类型。`Record<string, unknown>`（无论怎么拼写）到那个解析处为止。内部信任类型。参见 **boundary-discipline** 原则 skill。 |
| 由 schema 派生类型 | 在声明新接口之前，先考虑 `Pick`/`Omit`/`Parameters`/`ReturnType`/`Awaited`/`typeof`。 |
| 对象参数（Object args） | 传对象，不要传位置参数，这样参数顺序是自说明的。热点路径上跳过（逐帧渲染、分词器、解析器）。 |
| 真实测试 | 不要 mock 你能运行的东西。优先使用框架真实的测试原语，带上泄漏/一次性（disposable）检查，并在运行中的构建里验证 UI。只 mock 你在本地无法运行的东西。 |
| 结构化遥测 | 优先使用结构化 logger 诊断，带有足够的上下文，能凭一个 id 完成调试。交付代码中不要有 `console.log`。 |

示例：`references/patterns.md`。
