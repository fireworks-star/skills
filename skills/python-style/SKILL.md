---
name: "python-style"
description: "Python 编码风格规范：命名、f-string、推导式、with 资源管理、异常分层、类型标注、中文注释。编写或修改任何 Python 代码时必须遵循。"
---

# Python 编码风格规范

Python 个人写法风格。适用于所有 Python 代码，提供命名、惯用法、异常处理、注释等细则。

## 1. 命名规范

| 对象 | 风格 | 示例 |
|------|------|------|
| 变量、函数、方法 | `snake_case` | `batch_size`、`recursive_split_text`、`load_data` |
| 类 | `PascalCase` | `MultiHeadAttention`、`ProductionCrawler` |
| 常量 | `UPPER_CASE` | `DB_URI`、`SEED`、`ALS_FACTORS` |
| 模块/文件 | 小写+下划线 | `app.py`、`data.py`、`hot_topic.py` |

- 函数名动词开头：`create_user`、`get_by_id`、`save`、`delete`、`extract_audio_text`
- 变量名语义明确：`model_name`、`api_key`、`chunk_overlap`，禁止 `a`、`tmp1` 类无意义命名
- 表名用复数（`users`、`orders`），字段名 `snake_case`（`created_at`、`hashed_password`）

## 2. 语言惯用法（Pythonic 写法）

**f-string 优先**，不用 `+` 拼接和 `%` 格式化：

```python
print(f"采集完成，共获取 {len(self.data)} 条数据")
print(f"转换失败: {e}")
```

**推导式替代循环构建**：

```python
squares = [x**2 for x in range(5)]
evens = [x for x in range(20) if x % 2 == 0]
gen = (x**2 for x in range(5))  # 大数据量用生成器省内存
```

**with 语句管理一切资源**（文件、数据库 Session、浏览器上下文）：

```python
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()

with Session(engine) as session:
    session.add(user)
    session.commit()
```

## 3. 异常处理分层

**精确异常优先**：能预期具体异常时捕获具体类型，禁止裸 `except:`。

```python
# 预期明确的异常 → 捕获具体类型
try:
    print(int("hello"))
except ValueError as e:
    print(f"转换失败: {e}")

# 批量数据处理循环 → 捕获后记录并 continue，单条失败不中断整体流程
try:
    item = extract_item(node)
except Exception as e:
    print(f"提取数据时出错: {e}")
    continue
```

事务处理用 `try/except/finally` 完整包裹 `begin/commit/rollback`。

## 4. 结构组织

**导入顺序**：标准库 → 第三方库 → 自定义模块，三段之间空行。

**main 守卫**：可执行脚本必须有：

```python
def main():
    ...

if __name__ == '__main__':
    main()
```

**类型标注**：公开函数、类方法、异步方法签名标注参数和返回类型：

```python
async def scroll_to_load(self, max_scrolls: int = 10, scroll_interval: float = 2.0):
    ...
```

**函数短小**：单函数控制在可读范围内，只做一件事；长流程拆成 Pipeline 分阶段函数，用 `main()` 编排。

## 5. 注释规范（中文）

- 注释语言统一中文
- **解释"为什么"而非"是什么"**——命名已说明的不写注释：

```python
# 好：解释决策原因
# 所有分隔符都已尝试仍未找到时，回退到窗口最大边界
return text[start:start + chunk_size]

# 坏：复述代码
# 返回结果
return result
```

- 公开函数写 docstring，含功能、Args、Returns：

```python
def generate_tts(text: str, output_path: str) -> str:
    """使用 Edge TTS 生成语音配音。

    Args:
        text: 要合成语音的文本
        output_path: 输出 mp3 路径

    Returns:
        音频文件路径，失败返回空字符串
    """
```

- 部分文件头保留 `# -*- coding: utf-8 -*-`（项目已有惯例时跟随）
