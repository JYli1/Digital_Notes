> 关联：[[../Java安全/Java安全总览|Java安全总览]]、[[CC1]]

# CommonsCollections CC6

CC6 是 CommonsCollections 反序列化链里常见的一条链，适合和 [[CC1]] 对照学习。整理时重点关注触发入口、`LazyMap` / `TiedMapEntry` 的触发方式，以及最终如何走到 `ChainedTransformer`。

## 核心思路

1. 利用反序列化触发某个容器对象的 `readObject` 或哈希计算流程。
2. 通过 `TiedMapEntry` 触发 `LazyMap#get`。
3. `LazyMap` 调用恶意 `Transformer` 链。
4. `ChainedTransformer` 逐步反射调用到命令执行 sink。

## 和 CC1 的关系

- [[CC1]] 更适合先理解 `Transformer` 链和反射调用。
- CC6 更适合理解 `HashMap`、`TiedMapEntry`、`LazyMap` 如何组合触发。

## 复盘检查点

- CommonsCollections 版本是否满足链条条件。
- 目标 JDK 版本是否有限制。
- payload 生成后是否需要先清理 map 里的触发痕迹。
- 反序列化入口是否真的会读取可控对象。
