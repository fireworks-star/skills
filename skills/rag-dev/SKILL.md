---
name: "rag-dev"
description: "RAG 系统工程规范：递归分块、父子文档分割、多模态解析、数据清洗校验、敏感数据本地化。开发检索增强生成/知识库/文档问答系统时使用。"
---

# RAG 系统工程规范

RAG 工程链路标准。适用于知识库、文档问答、检索增强类系统开发。

## 1. 分块策略

**递归分块为默认方案**：按分隔符优先级切分（段落 > 句子 > 短句 > 空格 > 固定长度），相邻块保留重叠上下文。

```python
def recursive_split_text(text, chunk_size=200, chunk_overlap=50, separators=None):
    """按照分隔符优先级寻找切点，并保留相邻块的重叠内容。"""
    if chunk_size <= 0:
        raise ValueError("chunk_size 必须大于 0")
    if not 0 <= chunk_overlap < chunk_size:
        raise ValueError("chunk_overlap 必须为 0 到 chunk_size 之间")
```

- 常规文档：`chunk_size=200` 附近起步，`chunk_overlap` 约为 `chunk_size` 的 25%
- **参数校验放函数入口**，配置错误立即抛错，不带病运行

**父子文档分割**用于长文档两层切分——先切大父块（800 字符），父块内部再切小子块（500 字符），检索命中子块、返回父块上下文：

```python
parent_documents = split_documents(documents, chunk_size=800, chunk_overlap=100)
for parent_index, parent_document in enumerate(parent_documents):
    child_chunks = recursive_split_text(
        parent_document["content"],
        chunk_size=500,
        chunk_overlap=75,
    )
```

**按数据特点选策略，不盲目套用**：短文本（如票据 OCR 结果，每条不足百字）不使用父子文档分割——策略服务数据形态，不是反过来。

## 2. 多模态处理链路

| 模态 | 工具 | 要点 |
|------|------|------|
| 图片 | 视觉理解/ OCR | 统一转结构化文本 |
| 视频 | 抽帧 + 视频解析 | 先提取再入库 |
| 语音 | FunASR | 转文字后走文本链路 |

## 3. 数据清洗与校验规范

结构化抽取后的数据必须做标准化，核心规则：

```python
# 金额统一换算为分（INT64），避免浮点误差
# 缺失值保存为 null，禁止用 0 冒充——0 是业务含义（金额为零），null 是"未知"
record = {
    "amount": int(yuan * 100) if yuan is not None else None,
    "date": parsed_date if parsed_date else None,
}
```

- 每类文档先定义统一字段结构（如票据 12 字段），再写抽取逻辑
- 抽取结果过校验逻辑（日期合法性、金额范围）后才入库

## 4. 数据安全与部署边界

**敏感数据（财务、票据、个人信息）严禁外传云端 API**——本地化部署处理：

> 财务数据属于敏感信息，严禁外传 → 不使用任何云端 API，改为在本地服务器部署模型处理。

决策顺序：数据安全 > 便利性 > 成本。涉及敏感数据时，模型选型直接排除云 API 方案。

## 5. 检索与评估

- 数据入库链路：解析 → 清洗 → 分块 → 向量化 → 向量库（Milvus）/ ES
- 评估先行：上线前用固定评测集验证检索命中率和答案质量，调参依据评测结果而非感觉
- 典型多模态入库流程：docx/pdf/图片 → python-docx / pyMuPDF / OCR → 文本 → 分割 → 去无意义段落 → 向量存储
