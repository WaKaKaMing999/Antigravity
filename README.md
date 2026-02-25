# V9.2 云端自动化情报系统架构说明书

## 1. 基础设施与底层环境
- **物理节点**：部署于腾讯云 (Tencent Cloud) 新加坡节点。
- **硬件资源**：配置为 2核 CPU / 4GB 内存 / 30Mbps 带宽。
- **流量配额**：每月限额 1536GB，因全站流量经过隧道代理，需严格监控出站流量。
- **部署架构**：采用 Docker Compose 进行全容器化编排部署。
- **宿主机安全**：防火墙遵循最小权限原则，仅允许开放 22 端口 (SSH)。业务端口（如 5678 等）在宿主机层面被强制关闭。

## 2. 网络拓扑与零信任访问策略
- **全局统一网关**：系统废弃直接 IP 访问，所有外部请求必须通过全局唯一出入口 `https://n8n.ai-worker.top` 访问。
- **隧道加密**：服务器作为 Cloudflare 隧道的末端接入点，实现物理级别的网络隐身。
- **内部通信网络**：所有容器（包含 n8n、RSSHub、Tunnel 及自定义微服务）均运行在名为 `n8n_bridge` 的 Docker 内部自建网络中。
- **内部寻址规范**：内部服务间的调用必须使用内部 DNS（例：`http://rsshub:1200` 或 `http://antigravity:8000`），严禁绕行公网，以实现零延迟与零流量消耗。
- **第三方授权 (OAuth)**：Google API 等外部授权的重定向 URI 被严格锁定为 `https://n8n.ai-worker.top/rest/oauth2-credential/callback`。Google Cloud 项目状态必须永久保持在 [In production] 模式。

## 3. 核心微服务矩阵
当前内部网络 (`n8n_bridge`) 运行着四大核心容器：
1. **`n8n_main` (业务流转引擎)**：系统的核心枢纽，负责调度自动化工作流。其环境变量被强制约束，`WEBHOOK_URL` 和 `N8N_EDITOR_BASE_URL` 均指向全局唯一网关。
2. **`rsshub` (高频数据采集节点)**：运行于内部 `1200` 端口。主要负责广域情报的初步探测与 RSS 订阅源生成。
3. **`antigravity_agent` (AI 深度解析微服务)**：基于 Python/FastAPI 自研的内部处理节点，运行于内部 `8000` 端口。
    - **防壁垒抓取**：集成 ScraperAPI (`premium=true` 模式) 突破目标网站的反爬/WAF 限制（如高防御的政务网或商业终端），抓取动态渲染的完整 DOM 树。
    - **AI 融合提纯**：在获取正文后，直接在微服务层调用 OpenRouter 大模型接口进行数据提纯与结构化提取，返回标准 JSON。
    - **安全解耦**：所有 API 密钥（ScraperAPI、OpenRouter）均剥离至 `.env` 文件独立管理，杜绝硬编码。
4. **`cloudflared-tunnel` (安全接入守护进程)**：负责维持与 Cloudflare 边缘节点的加密长连接。

## 4. 数据持久化标准
- **挂载策略**：采用 Docker External Volumes (`n8n_data`) 进行数据持久化。
- **物理映射**：宿主机数据存放路径为 `/var/lib/docker/volumes/n8n_data/_data`。
- **文件操作红线**：工作流中所有产生物理文件落地（如报表生成、媒体下载）的节点，其写入路径必须且只能是 `/home/node/.n8n` 及其子目录，以规避容器生命周期重置导致的数据丢失。

## 5. 系统容错与资源熔断机制
- **内存溢出 (OOM) 防护**：鉴于 2C4G 的硬件瓶颈，处理超过 50 个数据对象的批量任务时，必须启用 `Split In Batches` (Loop) 节点，批次处理上限设定为 10-20 条，并强制在批次间插入 `Wait` 节点进行节流缓冲。
- **外部 API 熔断**：针对大模型 API 及第三方推送，工作流节点必须配置 `On Error -> Continue` 或挂载专属的 Error Trigger。禁止因单一外部依赖超时导致全局宏观分析流中断。
- **微服务容错**：`antigravity_agent` 内置 Titanium 防崩溃装甲，遇到目标站拦截或 API 欠费时，强制返回标准报错 JSON（如 `scraper_error`），确保 n8n 主流程不报 500 崩溃错误。
