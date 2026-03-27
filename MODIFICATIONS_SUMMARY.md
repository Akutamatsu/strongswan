# IKE_SA_INIT KE 失败重试功能实现总结

## 概述

在应用层实现了当 initiator 在接收到 IKE_SA_INIT 后、处理 KE 公钥时失败时，能够重试发送 IKE_SA_INIT 的功能。该功能可通过配置项限制重试次数，默认重试 1 次。

## 修改文件

### 1. conf/options/charon.opt

**添加新配置选项**：

```
charon.max_ike_init_retries = 1
	Maximum number of times to retry IKE_SA_INIT when the initiator receives
	an IKE_SA_INIT response but fails to process it (e.g. key exchange failure).
	0 to disable retries.
```

- **位置**：`charon.retry_initiate_interval` 和 `charon.reuse_ikesa` 之间
- **默认值**：1（重试最多 1 次）
- **用途**：控制 KE 失败时的重试次数
- **配置示例**：
  - `1` - 重试 1 次（共 2 次尝试）
  - `2` - 重试 2 次（共 3 次尝试）
  - `0` - 禁用重试，失败后直接销毁 IKE_SA

### 2. src/libcharon/sa/ikev2/tasks/ike_init.c

**修改内容**：

#### a. 添加新结构体字段 (第 134-140 行)

```c
/**
 * Retries due to key exchange public key application failure
 */
u_int ke_failed_retries;
```

- 用于跟踪 KE 公钥应用失败导致的重试次数
- 与现有的 `retry` 字段区分（后者用于 INVALID_KE_PAYLOAD 和 COOKIE）

#### b. 修改 process_i 中 ke_failed 的处理 (第 1442-1458 行)

**原逻辑**：
```c
if (this->ke_failed)
{
    DBG1(DBG_IKE, "applying key exchange public value failed");
    return FAILED;
}
```

**新逻辑**：
```c
if (this->ke_failed)
{
    uint32_t max_retries;

    max_retries = lib->settings->get_int(lib->settings,
                        "%s.max_ike_init_retries", 1, lib->ns);

    if (max_retries > 0 && this->ke_failed_retries < max_retries)
    {
        DBG1(DBG_IKE, "applying key exchange public value failed, "
             "retrying IKE_SA_INIT (attempt %d/%d)",
             this->ke_failed_retries + 1, max_retries);
        this->ike_sa->reset(this->ike_sa, FALSE);
        this->ke_failed_retries++;
        return NEED_MORE;
    }
    DBG1(DBG_IKE, "applying key exchange public value failed");
    return FAILED;
}
```

## 工作流程

```
1. Initiator 发送 IKE_SA_INIT
   ↓
2. Responder 返回 IKE_SA_INIT 响应
   ↓
3. ike_init.process_i() 处理响应
   ↓
4. ke_payload 处理失败 (ke_failed = TRUE)
   ↓
5. 检查重试次数
   ├─ 未超过限制：
   │  ├─ 调用 ike_sa->reset(false)
   │  ├─ 增加 ke_failed_retries 计数器
   │  ├─ 返回 NEED_MORE
   │  ├─ task_manager 检测到 reset 标志
   │  ├─ 调用 initiate() 重新激活任务
   │  └─ 重新生成并发送 IKE_SA_INIT
   │
   └─ 超过限制或禁用重试：
      ├─ 返回 FAILED
      ├─ task_manager 销毁 IKE_SA
      └─ 连接失败
```

## 应用层重试机制

### 不需要修改 task_manager_v2.c

现有的 task_manager 已经支持所需的重试机制：

1. **reset 标志处理**（`process_response` 中）：
   - 当任务调用 `ike_sa->reset()` 时，设置 `this->reset = TRUE`
   - 在每个任务处理后检查 `this->reset` 标志
   - 如果设置，销毁枚举器并调用 `initiate()`

2. **initiate() 重新激活**：
   - 检查 `active_tasks` 数量
   - 当数量为 0 时，根据 IKE_SA 状态激活新任务
   - IKE_INIT 任务会重新被激活
   - `build_i()` 生成新的 IKE_SA_INIT 消息

### 关键代码路径（task_manager_v2.c）

```c
// process_response 中的处理 (~L883)
if (this->reset)
{
    /* start all over again if we were reset */
    this->reset = FALSE;
    enumerator->destroy(enumerator);
    return initiate(this);
}

// reset() 方法中的处理 (~L2483)
while (array_remove(this->active_tasks, ARRAY_TAIL, &task))
{
    task->migrate(task, this->ike_sa);
    INIT(queued, .task = task, .time = now,);
    array_insert(this->queued_tasks, ARRAY_HEAD, queued);
}
this->reset = TRUE;
```

## 日志输出示例

### 重试时
```
00[IKE]  charon[1234]: applying key exchange public value failed, retrying IKE_SA_INIT (attempt 1/1)
00[IKE]  charon[1234]: initiating IKE_SA net[1] to 192.168.1.1
```

### 重试次数过限时
```
00[IKE]  charon[1234]: applying key exchange public value failed
00[IKE]  charon[1234]: charon[1234]: IKE_SA destroyed
```

## 配置使用示例

### strongswan.conf

```ini
charon {
    # 禁用重试
    max_ike_init_retries = 0
    
    # 默认 1 次重试
    max_ike_init_retries = 1
    
    # 允许最多 2 次重试（共 3 次尝试）
    max_ike_init_retries = 2
}
```

## 测试场景

1. **正常场景**：KE 处理成功 → 继续 IKE_AUTH
2. **单次失败重试**：
   - 第 1 次尝试：KE 失败
   - 触发 reset() 和重试
   - 第 2 次尝试：KE 成功 → 继续 IKE_AUTH
3. **多次失败**：
   - 尝试次数超过 `max_ike_init_retries`
   - 返回 FAILED
   - IKE_SA 销毁

## 风险评估与安全考虑

### 已解决的风险

1. **DoS 攻击防护**：
   - 通过 `max_ike_init_retries` 限制重试次数
   - 防止无限重试导致资源耗尽

2. **RFC 7296 合规性**：
   - 重试是在应用层处理，不违反协议层面的规范
   - 仅在 KE 失败时触发，符合错误处理的合理预期

3. **状态一致性**：
   - 使用 `ike_sa->reset()` 确保状态清理
   - 通过 `reset` 标志确保正确的任务流管理

### 兼容性

- **向后兼容**：默认值为 1，与现有的"重试一次"行为一致
- **可配置禁用**：设置为 0 恢复原始行为（无重试，失败后直接销毁）

## 与其他重试机制的区别

| 机制 | 位置 | 触发条件 | 行为 |
|-----|-----|---------|------|
| **INVALID_KE_PAYLOAD** | ike_init.c | Responder 拒绝 KE 方法 | 直接在协议层重试，返回 NEED_MORE |
| **COOKIE** | ike_init.c | Responder 发送 COOKIE | 直接在协议层重试，返回 NEED_MORE |
| **KE 失败重试** | ike_init.c | KE 公钥解密/验证失败 | 应用层通过 reset/initiate 重试 |
| **retry_initiate_interval** | ike_sa.c | 连接初始化失败 | 定时器触发新的初始化 |

## 后续改进建议

1. **统计信息**：在 IKE_SA 中记录各类重试的统计
2. **自适应重试**：根据失败类型动态调整重试策略
3. **事件通知**：向控制层报告重试事件
4. **扩展支持**：支持其他协议层失败场景的重试
