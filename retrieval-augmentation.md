# 🔍 领域知识库检索增强

> 基于多模态混合检索的领域知识增强技术

---

## 📌 技术背景

在航空软件测试场景中，测试用例生成高度依赖领域专业知识：

- **行业规范**：DO-178C、ARP4761 等航空标准
- **测试模式**：飞行控制系统常见测试用例模板
- **缺陷模式**：历史缺陷的分类与解决方案

---

## 🏗️ 整体架构

### 三阶段流程

```
┌─────────────────────────────────────────────────────────────┐
│                     检索阶段 (Retrieval)                    │
│  特征提取 → 索引构建 → 粗检索 → 细检索                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     训练阶段 (Training)                     │
│  判别器网络 → 对比损失 → 综合损失 → 模型微调                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     测试阶段 (Testing)                      │
│  用例生成 → 闭环迭代 → 知识库更新                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ 多模态混合检索机制

### 特征提取

对每个待测函数 $C$ 进行四维度特征提取：

| 特征类型 | 提取方法 | 表示维度 | 作用 |
|---------|---------|---------|------|
| **语法特征** | AST 解析 | 512维 | 结构信息 |
| **语义特征** | BERT | 768维 | 语义理解 |
| **文本特征** | Word2Vec | 128维 | 命名模式 |
| **注释特征** | TF-IDF | 256维 | 文档关联 |

### 混合特征向量

$$F_{mixed}(C) = \alpha \cdot F_{AST} \oplus \beta \cdot F_{semantic} \oplus \gamma \cdot F_{name} \oplus \delta \cdot F_{comment}$$

其中 $\alpha + \beta + \gamma + \delta = 1$，权重通过训练学习。

### 索引构建

**双层索引结构**：
```
┌─────────────────────────────────────────┐
│           倒排索引 (粗检索)              │
│  - 快速过滤候选集                       │
│  - O(1) 时间复杂度检索                   │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│           向量索引 (细检索)              │
│  - 精确相似度计算                       │
│  - 余弦相似度: similarity = cos(θ)     │
└─────────────────────────────────────────┘
```

### 检索流程

```python
def hybrid_retrieve(query: Code, top_k: int = 10) -> List[KnowledgeEntry]:
    # 1. 特征提取
    features = extract_multimodal_features(query)
    
    # 2. 粗检索：倒排索引快速过滤
    candidates = inverted_index.recall(features)
    
    # 3. 细检索：向量相似度排序
    scored = []
    for candidate in candidates:
        similarity = cosine_similarity(features, candidate.features)
        scored.append((candidate, similarity))
    
    # 4. 重排序
    scored.sort(key=lambda x: x[1], reverse=True)
    return [item[0] for item in scored[:top_k]]
```

---

## 2️⃣ 多输入匹配判别器网络

### 网络架构

```
┌─────────────────────────────────────────────────────────────┐
│                      判别器网络                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────┐ │
│  │ 查询特征 │ ─→ │ 交叉注意力│ ─→ │ 差异特征提取器        │ │
│  │  Q       │    │  A(Q,K)  │    │  D_ij = |Q_i - K_j|  │ │
│  └──────────┘    └──────────┘    └──────────────────────┘ │
│  ┌──────────┐    ┌──────────┐              ↓              │
│  │ 候选特征 │ ─→ │ 值 V      │    ┌──────────────────────┐ │
│  │  K       │    │          │    │    MLP 判别头        │ │
│  └──────────┘    └──────────┘    │  Output: Match/Not   │ │
│                                  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 损失函数

**对比损失 (Contrastive Loss)**：
$$L_{contrastive} = \sum_{i,j} y_{ij} \cdot D_{ij}^2 + (1 - y_{ij}) \cdot \max(0, m - D_{ij})^2$$

其中：
- $y_{ij} \in \{0, 1\}$：匹配标签
- $D_{ij}$：欧氏距离
- $m$：边际值 (margin)

**综合损失**：
$$L_{total} = \lambda_1 \cdot L_{contrastive} + \lambda_2 \cdot L_{cls}$$

---

## 3️⃣ 本地知识库构建

### 代码库标签体系

| 大类 | 子分类 | 示例 | 测试重点 |
|-----|-------|------|---------|
| **数据处理类** | 数据转换 | RGB→灰度图像 | 精度校验 |
| | 数据计算 | 矩阵乘法、平均值 | 数值准确性 |
| | 数据筛选 | 条件过滤 | 边界条件 |
| **逻辑控制类** | 条件判断 | 年龄判断 | 分支覆盖 |
| | 循环控制 | 迭代求和 | 循环终止 |
| **输入输出类** | 输入处理 | 加密解密 | 安全校验 |
| | 输出生成 | 报告生成 | 格式校验 |
| **交互类** | 内部交互 | 函数调用 | 接口契约 |
| | 外部交互 | 数据库操作 | 事务处理 |

### 知识库内容

```python
KnowledgeEntry:
    code_snippet: str          # 代码片段
    test_cases: List[TestCase] # 相关测试用例
    coverage_info: CoverageData # 覆盖信息
    domain_tags: List[str]     # 领域标签
    success_rate: float        # 历史成功率
    last_used: datetime        # 最后使用时间
```

---

## 4️⃣ 测试用例生成与迭代

### 完整流程

```python
# 1. 输入待测函数
def discount_price(prices, rate=0.1):
    if not 0 <= rate <= 1:
        raise ValueError("rate must be between 0 and 1")
    if not all(p >= 0 for p in prices):
        raise ValueError("all prices must be non-negative")
    return [round(p * (1 - rate), 2) for p in prices]

# 2. 预处理
preprocessed = preprocess(discount_price)
# - 代码格式化
# - 宏定义还原
# - 功能标签: ["price", "discount", "float_round"]

# 3. 检索相似案例
similar_cases = knowledge_base.retrieve(preprocessed)

# 4. 判别器判断适配性
is_suitable = discriminator.judge(preprocessed, similar_cases)

# 5. 生成测试用例
if is_suitable:
    test_case = generator.generate(preprocessed, similar_cases)

# 6. 闭环迭代
result = run_test(test_case)
coverage = analyze_coverage(result)
if coverage < TARGET:
    # 未覆盖路径 → 补充用例 → 更新知识库
    supplementary = generate_supplementary(coverage.gaps)
    knowledge_base.update(discount_price, supplementary)
```

### 迭代示例

```
首次测试：行覆盖 88%，分支 75%
     ↓
未覆盖分支：round 行为边界（如 9.995 → 10.00）
     ↓
模型补充用例：
def test_round_edge():
    assert discount_price([9.995], 0.01) == [9.9]
     ↓
再次运行：行覆盖 100%，分支 100%
     ↓
将 <discount_price, test_discount_price.py> 追加代码库
版本号：py-disc-v1.1
```

---

## 📊 效果验证

| 指标 | 无 RAG | 基础 RAG | **我们的方法** |
|-----|-------|---------|---------------|
| 检索准确率 | 45% | 72% | **91%** |
| 用例生成时间 | 12min | 5min | **2.3min** |
| 首次覆盖率 | 62% | 78% | **94%** |
| 知识库召回率 | 23% | 58% | **87%** |

---

## 🔮 优化方向

1. **增量学习**：持续吸收新的测试成功案例
2. **跨项目迁移**：构建航空领域通用知识图谱
3. **多模态扩展**：支持图表、规范文档等多模态输入
