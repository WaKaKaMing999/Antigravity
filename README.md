# 🤖 Antigravity 量化架构文档 (Level 6: 宏观状态机 Regime Machine 版)

## 📌 核心架构概述
本系统已从“线性规则量化”全面进化为**“基于多资产流动性共振的宏观状态机 (Regime Machine)”**。
后端 (Python/FastAPI) 已完全接管数据获取、持久化存储与极值风控，n8n 端需全面“瘦身”并转变为纯粹的数据管道与消息路由。

### 🏗️ 物理环境与依赖
- **部署环境**: Ubuntu 独立云服务器，Python 3.12 虚拟环境 (`venv`)。
- **并发控制**: FastAPI 全局异步互斥锁 (`asyncio.Lock()`)，防止 n8n 并发穿透。
- **持久化层**: SQLite (`aiosqlite`) 强事务数据库，替代 JSON 文件，根除并发脏写。
- **宏观数据源 (Oracle)**: 引擎内部集成 `yfinance`，自动异步拉取 SPY, TLT, HYG, VIX 动量，**无需 n8n 提供盘面数据**。
- **记忆库**: 挂载 ChromaDB，提供本地向量化防幻觉检索。

---

## 🔌 核心 API 接口路由矩阵

### 1. 核心指挥中枢 `POST /api/final/orchestrator`
- **功能**: 并发触发 Virtual Brain 与 Survival Brain，评估宏观流动性状态 (Regime)，并执行机构级风险资产再分配 (Risk Overlay)。
- **输入 Payload (极简)**: 
  ```json
  { "news": "={{ $json.today_news }}" }
  输出 Response (新版 Schema，Telegram 解析必须匹配此结构):

JSON
{
  "status": "success",
  "risk_score": 85,
  "macro_regime": "LIQUIDITY_CONTRACTION", 
  "confidences": {
    "Virtual_Confidence": 0.52,
    "Survival_Confidence": 0.65
  },
  "raw_stock_intent": 80,
  "overlay_strength": 0.552,
  "final_positions": {
    "stock": 35,
    "bond": 0,
    "cash": 65
  }
}
(注：macro_regime 包含四种状态：LIQUIDITY_CONTRACTION, DEFENSIVE_RECESSION, RATE_REPRICING, NORMAL)

2. 实盘胜率闭环 POST /api/feedback/trade
功能: 接收本地算力（或实盘组件）回传的单次胜负，存入 SQLite，用于每日平滑更新 Virtual Brain 的话语权权重。

输入 Payload: {"is_win": true}

3. 防幻觉记忆库 POST /memory/store & POST /memory/query
功能: 分别用于向量化存储复盘事件，以及查询高度相似的历史事件与市场反应。

🎯 Antigravity (n8n 工程师) 的重构指令
基于上述架构，n8n 的核心工作流必须执行以下重构：

废弃旧数据源: 主干工作流（财经新闻中枢）必须彻底删除用于等待本地盘面数据计算结果的 Wait 节点及相关 Webhook 发送节点。

重塑 API Request: 调用 /api/final/orchestrator 时，剥离所有冗余因子，仅传输 news 上下文。

重构 Telegram 终端美学: 必须将 $json.macro_regime (宏观状态机判定) 提升至战报最高优先级显示；必须清晰列出动态扣减强度 (overlay_strength) 与最终三资产 (stock/bond/cash) 的极简分配比例。

业务域隔离: 国内局势监控 作为实体生存脑，严禁调用 orchestrator，必须维持独立推演。
