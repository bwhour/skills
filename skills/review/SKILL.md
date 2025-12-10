
---
name: review
description: "专业代码审查能力。当用户要求 review、审查、检查代码、或贴代码询问意见时自动触发。支持安全审计、性能分析、Web3/智能合约专项。"
---

# 🔍 Code Review Skill

## 触发场景

当用户：
- 说 "review"、"审查"、"帮我看看这段代码"、"检查一下"
- 贴了代码并询问意见
- 使用 `/review` 指令

## 输入期待

- 链/合约类型（ERC20/721/1155/自定义），是否可升级（proxy/UUPS/diamond）
- 关键资产与角色：国库/LP/抵押仓/路由/守护人/owner/roles
- 外部依赖：预言机来源、路由/桥、签名来源或其他合约
- 变更规模（行/文件）与主要变更点；若缺失则按高风险路径扫描并在报告中注明假设

## 审查流程

### Step 1: 快速定位高风险
```yaml
30秒内标记:
  - 链/合约类型、可升级与否、关键资产与角色
  - Diff 热点: 外部调用、状态写入、权限/role/owner 变更、资金流/转账、存储布局变更
  - 缺失信息的假设写入报告

额外初印象:
  - 文件大小/复杂度、命名一致性、明显反模式、测试/文档入口

按规模分档:
  小(<200行): 全量细查
  中(200–800行): 高风险优先(权限/资金/状态机/外部交互)，低风险抽查
  大(>800行): 先结构性风险(API/存储/权限/并发/迁移/升级)，必要时要求拆分/补设计与测试
```

### Step 2: 安全优先深查（区块链）
```yaml
必须逐行检查:
  - AccessControl/owner/roles/guardian/pausable
  - 资金流与数学: fee/rounding/slippage/bounds，余额一致性/双花
  - 外部交互: CEI、ReentrancyGuard、delegatecall/call.value 使用
  - 预言机/价格: 来源、多喂价/阈值、陈旧/操纵窗口
  - MEV/抢跑: slippage、deadline、commit-reveal、partial fill 限制
  - 升级性: storage layout 变更、initializer/reinitializer、gap
  - 签名: EIP-712 域/chainid/nonce/replay、防重放/吊销/过期
  - Fail-safe: pause/circuit breaker、限额、紧急退出/提款
  - 事件与不变量: 状态变更 emit；总量/抵押/债务/余额守恒
```

### Step 2.1: 通用六维补充
```yaml
正确性: 逻辑/边界/错误恢复？
安全性: 输入校验/数据泄漏/授权链路（非链侧）？
性能: 复杂度/热点路径/缓存或限流？
可读性: 命名/意图暴露/复杂分支可否拆解？
可维护性: 模块边界/依赖方向/配置与特性开关默认值？
可测试性: 可否注入依赖、Mock 外部、覆盖关键分支？
```

### Step 3: 输出报告

使用以下模板：

```markdown
## 🔍 Code Review Report

**文件**: `path/to/file`
**范围**: [已审查模块]；未查/抽查及理由
**评级**: ⭐⭐⭐⭐ (4/5)

### 📌 摘要
[核心风险与结论 1-2 句]

### 🔴 Critical (P0) - 必须修复
[无 / 或列出]

### 🟠 High (P1) - 应当修复
[问题描述 + 建议修复]

### 🟡 Medium (P2) - 建议改进
[问题描述]

### 🔵 Nit (P3-P4)
- [ ] 小问题列表

### 🌟 亮点
- ✅ 做得好的地方

### 🧪 测试建议
- `forge test ...` / `forge snapshot` / `slither .` / `echidna-test ...`（未运行需注明）

### ✅ 结论
[可合并 / 需修改 / 阻塞+原因]
```

---

## 🔐 安全检查清单

### 通用
```yaml
输入验证:
  □ 外部输入都验证了？
  □ SQL/NoSQL 注入防护？
  □ XSS 防护？

认证授权:
  □ 权限检查正确位置？
  □ Session/Token 安全？

敏感数据:
  □ 无硬编码密钥？
  □ 日志不打印敏感信息？
```

### Solidity / Web3 专项
```yaml
智能合约:
  □ ReentrancyGuard？
  □ Checks-Effects-Interactions？
  □ AccessControl 正确？
  □ 整数溢出保护？
  □ 预言机操纵防护？
  □ 闪电贷攻击向量？
  □ 前端抢跑防护？
  □ 签名域/nonce/重放防护？
  □ MEV 缓解(截止块/滑点/commit-reveal/partial fill 限制)？
  □ 升级/代理: storage layout、初始化、权限锁
  □ 事件: 关键状态变更均有 event
  □ 不变量: 供应/抵押/债务/余额守恒
```

### 公链安全实践 (Ethereum / Cosmos)
```yaml
Ethereum:
  □ chainid/fork 兼容；block.timestamp 使用安全？
  □ Gas griefing 评估（批量循环/回滚成本）
  □ 存储槽/代理对齐；selfdestruct 影响 (EIP-6780 后行为)
  □ Mempool/MEV: 抢跑、夹击、延迟填充风险

Cosmos (SDK/IBC):
  □ AnteHandler/feegrant/gov 参数是否变更
  □ Msg 权限/模块账户校验，bank/ibc 发送限额/速率
  □ IBC channel/sequence/timeout/replay 检查
  □ Invariants 保持 (supply/bank/staking/distribution)
```

### Go 专项
```yaml
并发:
  □ 共享状态加锁/无竞态？
  □ Channel 关闭顺序正确？
  □ Context 传递/超时/取消？

错误:
  □ error 全检查且带上下文？
  □ 边界/零值/空切片处理？
```

### TypeScript 专项
```yaml
类型/安全:
  □ strict 启用，避免 any？
  □ 输入校验(schema) 防注入/XSS？
异步:
  □ Promise 有 catch/错误上报？
  □ 资源释放/超时/重试退避？
```

---

## ⚡ 性能检查

```yaml
Gas/性能快速核对:
  □ 循环/批量操作是否触碰 storage？可否缓存到内存？
  □ 读路径能否改为 view/staticcall？
  □ 关键路径有粗略 gas 估算或 snapshot
  □ Events 替代冗余 storage 记录
```

---

## 🎨 Review 风格

```yaml
态度:
  - 建设性，解释"为什么"
  - 提供具体改进建议
  - 承认做得好的地方

优先级:
  安全 > 正确性 > 性能 > 可读性 > 风格
```

---

## 📊 问题严重程度

| 级别 | 标签 | 含义 |
|------|------|------|
| P0 | 🔴 CRITICAL | 阻塞合并 |
| P1 | 🟠 HIGH | 应修复 |
| P2 | 🟡 MEDIUM | 建议修复 |
| P3 | 🔵 LOW | 可选 |
| P4 | 🟢 NIT | 纯偏好 |

---

## 🔧 可用脚本

常用核查命令（未运行需注明）：
```bash
rg "<pattern>" <path>          # 快速定位风险点
forge test                     # Solidity 单元/属性
forge snapshot                 # Gas 快照
slither .                      # 静态分析
echidna-test ...               # Fuzz/不变量
```

---

## ⏱️ 时间盒与透明度

- 未全覆盖时需注明覆盖范围、剩余风险与假设，避免假装全量审查
