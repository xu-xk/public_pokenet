# 首通登记：事务、快照与中断恢复

## 问题

一场剧情战斗结束后，玩家要进入登记仪式、保存当时的队伍并返回游戏世界。若结果重复投递、玩家切换队伍，或流程中途断线，系统仍需要保持记录一致，并从已完成的阶段继续。

## 设计

1. 从已验证的战斗结果读取获胜及队伍快照，而不是临时读取玩家当前队伍。
2. 以角色为单位保存来源战斗、完成标识、队伍快照和流程阶段；登记时核对账号与角色归属，并锁定相关记录。
3. 把首通标记、返回检查点与阶段推进放入同一数据库事务，重复执行时保持幂等。
4. 用 `CEREMONY_PENDING → RETURN_PENDING → RETURNED` 表达可恢复的流程；重新进入对应地图时依据已持久化阶段续接。

下面是服务端中的一段精简代码，展示事务内检查归属、读取当前阶段和提交首通标记的思路；省略的依赖和调用链不在展示仓库中。

```java
return database.ctx().transactionResult(configuration -> {
    DSLContext tx = DSL.using(configuration);
    CharacterRecord owner = lockOwner(tx, accountId, characterId);
    if (owner == null) return false;
    Entry current = read(tx, accountId, characterId);
    if (current == null) return false;
    if (current.stage() != Stage.CEREMONY_PENDING) return true;
    CharacterRespawnCheckpointStore.upsert(tx, owner, accountId, characterId,
            CharacterRespawnCheckpointStore.initialKantoCheckpoint());
    // 首通标志与阶段更新继续在同一事务内提交。
});
```

代码节选自项目服务端 `KantoHallOfFameStore`；为便于阅读略去后续 SQL，示例不可独立编译。

## 验证与边界

历史候选的定向 PostgreSQL 测试 4/4、队伍快照测试 3/3 通过；覆盖登记、快照与恢复相关场景。客户端真实 GUI 流程未运行。另有全量测试源码编译失败记录，不能把定向测试通过说成全量通过。
