# TypeScript 模式

`SKILL.md` 中每条规则的代码示例。底层原则与语言无关。参见 **type-system-discipline** 和 **boundary-discipline** 原则 skill。

## 品牌类型（Branded types）

给原始类型打品牌，让它们不会被混用。在边界处验证一次。下游代码信任该类型。

```ts
type AgentId = string & { readonly __brand: "AgentId" };

function isAgentId(input: string): input is AgentId {
  return isUUID(input);
}

function parseAgentId(input: string): AgentId {
  if (!isAgentId(input)) throw new Error(`Invalid agent id: ${input}`);
  return input;
}

function focusAgent(id: AgentId): void {
  /* input is trusted */
}
```

遵循 `readonly __brand: 'X'` 的形状。不要发明新约定。

## 可辨识联合（Discriminated unions）

用字面量判别项来建模变体。每个变体共享同一个字段名，且每个变体的值都是唯一的，这样不可能的组合就无法被表示。

```ts
// Don't. Boolean + optionals lets contradictory states exist.
type DiffState = { loading: boolean; diff?: GitDiff; error?: string };

// Do. Only valid states exist.
type DiffState =
  | { kind: "loading" }
  | { kind: "ready"; diff: GitDiff }
  | { kind: "error"; error: string };
```

选一个判别项名字（`kind`、`type`、`tag`）并坚持用它。

## 构造式建模（Constructive modeling）

用全部合法的部件构造类型，而不是用运行时检查去限制一个松散类型。

非空，通过可变元组：

```ts
type NonEmpty<T> = [T, ...T[]];

// Don't: T[] plus a length check every caller must repeat
function pickWinner(entries: string[]): string {
  if (entries.length === 0) throw new Error("no entries");
  return entries[Math.floor(Math.random() * entries.length)];
}

// Do: an empty value of the type can't exist
function pickWinner(entries: NonEmpty<string>): string {
  return entries[Math.floor(Math.random() * entries.length)];
}
```

当传来的是一个普通的 `T[]` 时，用守卫收窄一次。这个事实随后会在类型中传递：

```ts
const isNonEmpty = <T>(arr: T[]): arr is NonEmpty<T> => arr.length > 0;
```

偶数长度，用成对表示：

```ts
type Pairs<T> = [T, T][];
```

时间区间，用起点加持续时间表示：

```ts
// Don't: a comment holds the invariant
type TimeRange = { start: Date; end: Date }; // start <= end

// Do: a negative range can't be written; derive end when needed
type TimeRange = { start: Date; durationMs: number };
```

让 `durationMs` 保持为普通数字。只有当原始数字可能被传到期望持续时间的地方时，才给它打品牌（按"品牌类型"一节），而不是出于反射。选择那个让坏状态无法被构造的表示，然后在其上暴露你需要的读取方式（`pairs.flat()`、一个 `rangeEnd()` 辅助函数）。

## 最简单全函数类型（Simplest total type）

不要什么都加强。如果对 `T[]` 的每一次操作都是全函数（total），就保留 `T[]`：

```ts
const sum = (xs: number[]) => xs.reduce((a, b) => a + b, 0); // [] is 0, fine
```

当松散类型迫使你在某个使用点撒谎时才加强。信号是 `!`、`arr[0] as T`，以及"这绝不会发生"的抛出：

```ts
// Don't: partiality smuggled past the compiler
function newestSession(sessions: Session[]): Session {
  return sessions.at(0)!;
}

// Do: strengthen the input; the assertion disappears
function newestSession(sessions: NonEmpty<Session>): Session {
  return sessions[0];
}
```

把结果弱化为 `Session | undefined` 是另一种全函数签名。

## `unknown` 优于 `any`

外部数据永远是 `unknown`。使用前先收窄。

```ts
// Don't
function handle(input: any) {
  return input.foo.bar;
}

// Do
function handle(input: unknown) {
  if (typeof input === "object" && input !== null && "foo" in input) {
    // narrowed; compiler verifies access
  }
}
```

外部来源包括 RPC 载荷、`JSON.parse`、`postMessage`、IPC、文件内容、环境变量、数据库结果。

## schema 优于手写守卫

在为外部数据写逐属性的类型守卫之前，先找找仓库的运行时 schema 库和已有 schema。让一个 schema 掌管验证，并从中派生 TypeScript 类型。不要同时维护一个 schema、一个重复的接口和一个可能各自漂移的守卫。

```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().uuid(),
  role: z.enum(["admin", "member"]),
});

type User = z.infer<typeof UserSchema>;

function parseUser(input: unknown): User {
  return UserSchema.parse(input);
}
```

当失败是一个预期分支时使用 `safeParse`。当仓库使用另一个 schema 库时，使用等价的推断辅助函数。不要为一个守卫新增 schema 依赖。这条规则偏好代码库已经信任的 schema 系统。

## 不用 `as` 断言

每一个 `as` 都是潜在的运行时崩溃。只有在类型系统已经验证了该断言之后才断言。

```ts
import { z } from "zod";

// Don't
const user = data as User;

// Don't
function isUser(data: unknown): data is User {
  return typeof data === "object" && data !== null && "id" in data;
}

// Do
const userSchema = z.object({ id: z.string(), name: z.string() });
type User = z.infer<typeof userSchema>;

function parseUser(data: unknown): User {
  return userSchema.parse(data);
}
```

当类型在前时，用它所证明的类型来标注验证器。这样编译器就会拒绝一个证明力度弱于该类型的验证器。把下面对象里的 `name` 去掉，赋值就会编译失败。

```ts
type User = { id: string; name: string };

const userSchema: z.ZodType<User> = z.object({ id: z.string(), name: z.string() });
```

当从已有代码中移除一个 `as` 时，先弄清 TypeScript 为什么无法推断：

- 缺少判别项：加一个，改用可辨识联合。
- 源类型过宽（比如 `Record<string, unknown>`）：收窄它。
- 无类型的边界：用掌管该形状的 schema 解析。只有在没有 schema 的地方才新增。
- 确实无法表达：使用品牌类型或 `satisfies`。

## 收窄层级（Narrowing hierarchy）

从最好到最后手段：

1. **可辨识联合的 switch / if。** 编译器自动收窄。
2. **`in` 操作符。** `"key" in obj` 收窄到包含该键的变体。
3. **`typeof` / `instanceof`。** 用于原始类型和类实例。
4. **用户定义类型守卫。** 当上面这些不够时。
5. **`as` 断言。** 只在验证之后。

```ts
function area(s: Shape): number {
  if ("radius" in s) return Math.PI * s.radius ** 2; // narrowed to circle
  return s.width * s.height; // narrowed to rect
}
```

## 类型守卫

守卫必须真正验证其断言。说谎的守卫比 `as` 更糟。

```ts
function isCircle(s: Shape): s is Shape & { kind: "circle" } {
  return s.kind === "circle";
}
```

可能的话优先用判别项收窄。

## 穷尽性（Exhaustiveness）

在 default 分支里，把判别项赋给一个 `never` 类型的局部变量。

```ts
// Value-returning switch
function area(s: Shape): number {
  switch (s.kind) {
    case "circle":
      return Math.PI * s.radius ** 2;
    case "rect":
      return s.width * s.height;
    default: {
      const _exhaustive: never = s;
      return _exhaustive;
    }
  }
}

// Void switch
function handle(s: Shape): void {
  switch (s.kind) {
    case "circle":
      drawCircle(s);
      break;
    case "rect":
      drawRect(s);
      break;
    default: {
      const _exhaustive: never = s;
      void _exhaustive;
    }
  }
}
```

返回值的 switch 用 return 风格，语句型 switch 用 void 风格。

## `satisfies` 优于 `as`

`satisfies` 在不拓宽字面量类型的情况下做验证。

```ts
// Don't. Widens, loses literal types.
const config = { theme: "dark", cols: 3 } as Config;

// Do. Validates AND preserves literal types.
const config = { theme: "dark", cols: 3 } satisfies Config;
// config.theme is "dark" (literal), not string
```

## 边界验证（Boundary validation）

在数据进入的地方验证一次。内部信任类型。参见 **boundary-discipline** 原则 skill。

- **线路格式（Wire formats）**（proto、JSON-RPC）：用 `ignoreUnknownFields` 解析，让前向兼容的改动不会破坏旧客户端。
- **持久化 JSON：** 带版本的 blob，解析处包一层 try/catch。
- **不要在调用链深处重复验证。**

## 由 schema 派生类型

当 `.proto`、OpenAPI 规范、GraphQL schema 或数据库迁移已经定义了某个形状时，从生成的类型派生，而不是复制它们。

```ts
// Don't. Duplicate shape, drifts when the schema changes.
type CheckSummary = {
  totalCount: number;
  checks: { name: string; status: string }[];
};
function renderChecks(s: CheckSummary) {
  /* ... */
}

// Do. Derive from the generated schema type.
import type { ChecksMessage } from "<generated module>";
function renderChecks(s: Pick<ChecksMessage, "totalCount" | "checks">) {
  /* ... */
}
```

在写新接口之前，先考虑 `Pick`、`Omit`、`Parameters`、`ReturnType`、`Awaited`、`typeof`。

## 对象参数（Object args）

```ts
// Don't. Swap two args, still compiles.
openFile(uri, {
  startLineNumber: 10,
  startColumn: 1,
  endLineNumber: 10,
  endColumn: 1,
});

// Do. Order-independent, self-documenting.
openFile({
  uri,
  selection: {
    startLineNumber: 10,
    startColumn: 1,
    endLineNumber: 10,
    endColumn: 1,
  },
});
```

热点路径上跳过：逐帧渲染、分词器、解析器，以及任何分配成本要紧的紧循环。
