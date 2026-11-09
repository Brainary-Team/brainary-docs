# 快速开始

这篇文档展示如何在编写PoA的时候快速接入Working Memory

## 1. 声明集合

一个典型的PoA目录如下：

```text
my-poa/
├── src/
│   ├── memory.mts          # 集合声明
│   └── main.mts            # 入口，注册并装载
├── generated/
│   └── memory.mts          # 生成的 schema
├── dist/
│   └── poa.js              # 打包产物
├── wm.json                 # 声明文件、根类和生成位置
└── tsconfig.json
```
其中`generated/` 和 `dist/` 由后面的生成与打包命令产生。

PoA作者在 `src/memory.mts` 中用 TypeScript 类型标记声明集合字段。根类的字段必须使用 `declare`，不能有初值，也不能带 `private` / `protected`；根类不能继承或实现其他类。值类型写在同一文件。

```typescript
// src/memory.mts
import type { List } from "@brainary/wm-sdk";

export interface Evidence {
  title: string;
  verified?: boolean;
}

export class WorkingMemory {
  declare evidence: List<Evidence>;
}
```

上面的 `WorkingMemory` 只是这篇文档选用的例子。根类叫什么、声明哪些集合、项长什么样，都由作者决定；这里只有一个 List 字段 `evidence`，项的形状是 `Evidence`。

## 2. 生成 schema

PoA作者在 `wm.json` 中指出声明文件、根类和生成位置：

```json
{
  "declarations": [
    {
      "source": "src/memory.mts",
      "class": "WorkingMemory",
      "id": "urn:example:wm-quickstart",
      "output": "memory.mts"
    }
  ],
  "out_dir": "generated",
  "entry": "src/main.mts",
  "bundle": "dist/poa.js",
  "tsconfig": "tsconfig.json"
}
```

在 SDK 仓库根目录先构建 SDK，再把 `wm.json` 的路径传给 `build_cli.js`：

```bash
npm run build
node packages/wm-sdk/dist/build_cli.js wm.json
```

这条命令读取 `src/memory.mts`，检查 TypeScript 项目，写出 `generated/memory.mts`。生成文件提供 `WorkingMemory.to_schema()`。入口文件按 NodeNext 规则用 `.mjs` 扩展名导入它。磁盘上不会先生成一个同名的 `generated/memory.mjs`。

## 3. 注册并装载

作者在入口文件中导入生成模块，再注册并装载：

```typescript
// src/main.mts
import { Kind, PoA } from "@brainary/poa-sdk";
import { WorkingMemory } from "../generated/memory.mjs";

export async function run_wm() {
  const poa = await new PoA({ agent_id: "wm-quickstart" })
    .register_schema(Kind.WM, WorkingMemory.to_schema());

  const memory = await poa.load_data(Kind.WM, {
    evidence: { entries: [] },
  });
}
```

其中重要的概念：

- `PoA` 是入口，构造参数为 `{ agent_id: string }`。`agent_id` 只做作者标签。
- `register_schema` 成功后返回带声明类型的注册对象。
- `load_data` 成功后返回按声明字段生成的 `memory` 句柄。

`Kind` 这里只传 `Kind.WM`。初始数据的顶层字段必须与 schema 完全一致。List 的初始形状是 `{ entries: [] }`。

## 4. 写入并查询

装载得到 `memory` 之后，按字段名调用集合方法：

```typescript
await memory.evidence.append("first", {
  title: "verified source",
  verified: true,
});
const page = await memory.evidence.query({
  filter: { item_id: "first" },
});
```

`append` 返回 `{ position }`，位置从 0 起。`filter` 的多个条件取交集，这里只按 `item_id` 查询。

## 5. 多笔写入

单独一次写入会自己提交。多笔写入放进 `memory.batch`，回调成功后整批最多提交一次。回调必须同步执行并返回 `undefined`，不能 `await`、不能嵌套调用、不能返回 Promise。写操作返回 Promise，在回调里用 `void` 丢掉它。

```typescript
await memory.batch(() => {
  void memory.evidence.append("second", { title: "follow-up" });
  void memory.evidence.append("third", { title: "closed" });
});
```

同一根类还可以声明 `Graph`、`Tree` 和 `KvMap`。字段名就是句柄名，装载后从 `memory` 的同名字段调用。
