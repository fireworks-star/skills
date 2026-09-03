---
name: "data-pipeline"
description: "数据处理与采集规范：pandas 分析流程、可视化标准、爬虫工程（UA/限速/反爬/超时/降级）、Streamlit 状态与缓存。开发数据分析、爬虫、Streamlit 应用时使用。"
---

# 数据处理与采集规范

数据工程模式。适用于数据分析（pandas/matplotlib）、数据采集（requests/Playwright）、Streamlit 应用开发。

## 1. 数据分析标准流程

固定链路：**读入 → 检查缺失 → 清洗 → 转换 → 分组聚合 → 可视化**。

```python
df = pd.read_csv('output.csv', encoding='utf-8')

# 每次读入先检查缺失，不直接上手分析
print(df.isnull().sum())

df_clean = df.dropna()
result = df.groupby('city').agg({
    'score': ['mean', 'max', 'min', 'std'],
    'salary': ['sum', 'mean'],
    'name': 'count',
})
```

- 缺失值显式处理（`dropna` 或按列填充），不留隐式 NaN 带入计算
- 聚合用 `groupby + agg`，禁止手写循环统计
- 工具链分层：Jupyter 探索 → NumPy/Pandas 计算 → Matplotlib/Seaborn 可视化 → Streamlit 应用化

## 2. 可视化规范

图不是画出来就完，五要素齐全：标题、轴标签、图例、网格、导出：

```python
fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(x, y1, label='sin(x)', color='blue')
ax.set_title('三角函数', fontsize=16, fontweight='bold')
ax.set_xlabel('X轴', fontsize=12)
ax.set_ylabel('Y轴', fontsize=12)
ax.grid(True, alpha=0.3)
ax.legend()
plt.savefig('plot.png')
```

## 3. 爬虫工程规范

**工具选型看场景**——静态页面用 requests + BeautifulSoup；以下三种场景必须换 Playwright：JavaScript 动态渲染、滚动加载、需要登录认证。

**生产级爬虫必备项**：

```python
# 1. 真实浏览器 UA，模拟正常请求
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...'
}

# 2. 隐藏自动化特征
browser = await playwright.chromium.launch(
    headless=True,
    args=["--disable-blink-features=AutomationControlled"]
)

# 3. 显式超时
page.set_default_timeout(30000)

# 4. 控制节奏：滚动加载间隔等待，不高频轰炸
await asyncio.sleep(scroll_interval)
```

**结构化提取**：数据统一收集为字典列表，字段名稳定：

```python
results.append({
    "title": title,
    "img_url": img_url,      # 支持懒加载 data-src、data-original
    "like_num": like_num,
})
```

**测试组织**：爬虫逻辑用类封装（浏览器/页面/采集逻辑分离），测试用 `unittest.TestCase` 组织。

**注意**：爬虫仅用于授权范围内的公开数据采集，遵守目标站点服务条款。

## 4. Streamlit 应用规范

**状态管理**：所有会话状态走 `st.session_state`，入口先初始化：

```python
if "count" not in st.session_state:
    st.session_state.count = 0
```

**缓存**：重复加载/计算/查询的数据加缓存，带 TTL：

```python
@st.cache_data(ttl=3600)
def load_data():
    return pd.DataFrame({...})
```

**多页面导航**：用 `st.session_state.page` 管理当前页；数据模型复用 SQLModel；数据加载函数与页面渲染分离。

## 5. 前端基础规范（配合 Streamlit/简单页面）

- 结构、样式、交互三层分离：HTML 骨架、CSS 外置文件、JS 行为
- 语义化标签 + class 组织页面，样式不内联（教学演示除外）
