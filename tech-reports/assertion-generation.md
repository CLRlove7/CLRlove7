# 🔍 AssertPath-LLM：结构引导的单元测试断言生成框架

> 📌 脱敏说明：本报告已移除敏感项目信息，仅保留技术原理与通用实现方案

## 🎯 核心问题
传统 LLM 生成单元测试断言时存在三大挑战：
1. **幻觉问题**：生成语法正确但语义错误的断言
2. **注意力稀释**：长上下文下关键程序点信息丢失  
3. **目标定位困难**：难以精准识别需要断言覆盖的分支条件

## 💡 技术方案

### 1️⃣ 测试义务标注 & TOG 构建
[脱敏架构图]
源码 → AST/CFG/DDG 多视图融合 → 红边标注测试义务点 → Test-Oriented Graph

### 2️⃣ 最小线性子路径分解（MLSP）
- 将全局断言生成任务分解为**局部子路径覆盖任务**
- 有效缓解长上下文注意力稀释问题
- 算法复杂度：O(n) 线性路径提取

### 3️⃣ 7槽位断言模板机制
| 槽位 | 语义 | 示例 |
|------|------|------|
| AssertType | 断言方法类型 | `assertEquals`, `assertThrows` |
| Expected | 期望值 | `200`, `expectedUser` |
| Actual | 实际执行结果 | `service.calculate(x)` |
| Delta | 精度容差 | `0.001` |
| Message | 失败提示 | `"Calculation mismatch"` |
| Precondition | 前置条件校验 | `assertNotNull(input)` |
| Postcondition | 后置状态验证 | `verify(mock).call()` |

### 4️⃣ 闭环反馈迭代
