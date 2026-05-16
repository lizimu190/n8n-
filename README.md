# 🚀 「知行」—— 大模型与工作流双轨驱动的智能教务系统 (Pro)

「知行」是一个生产级的智能教务管理系统。项目采用**前端轻量级 UI 遥控器 + 后端 n8n 自动化工作流 + 大语言模型（通义千问 Qwen-Plus）** 的双轨驱动架构，实现了自然语言交互下的成绩一键录入、多维度报表异步生成、以及异常不及格成绩的实时预警拦截。

---

## 🧭 系统全局架构图

整个系统实现了“无状态前端”向“有状态复杂后端”的完美解耦：
[用户前端 (HTML/JS)] -> (REST POST) -> [n8n Webhook] -> [AI Agent 意图分发] -> [智能路由分支] -> [Airtable 数据库] -> (JSON/Text) -> [前端渲染]

---

## ✨ 核心亮点与产品功能

### 1. 💬 AI 语义化成绩流式处理
- **智能去重与热更新**：后端工作流在收到录入指令（如“数学 85分”）时，不会无脑新建。系统会首先在 Airtable 中利用 `={Subject} = '{{ $json.subject }}'` 进行检索。
- **状态分流（If 条件节点）**：
  - 如果科目已存在，自动触发 **Update 节点** 更新分数并返回“成功更新成绩”。
  - 如果科目不存在，则触发 **Create 节点** 新建记录。

### 2. 📊 全量成绩数据多维度看板
- 一键调取数据库中所有的历史成绩，利用后端高性能 JavaScript 隔离沙箱，全自动计算**总科目数、平均分（精确到小数点后两位）、历史最高分**，并格式化输出精美的明细排版列表。

### 3. ⚠️ 动态阈值安全合规拦截 (不及格预警)
- 内置**异常成绩拦截网关**。当录入成绩 $< 60$ 分，或者点击“异常成绩预警”时，后端会秒级捕获安全阈值，将 UI 状态动态升级为 `danger` 警告流，并强制高亮标记异常学科。
- 后端 JavaScript 完美攻克了 Airtable 的“幽灵空白行”与 n8n 的“假空箱子（空 JSON）”漏洞，确保预警 100% 准确不误报。

---

## 🛠️ 后端 n8n 工作流架构解构

项目的后端由极其严密的 `智能管家系统.json` 工作流支撑，包含 **19 个生产级节点**：

### 🧠 1. Agent 核心分类规则（Prompt 级路由）
大模型（`qwen-plus`）被规约成一个极度精准的后台 API 数据解析引擎，拒绝任何废话，严格输出包含 `intent` 和 `entities` 的半角 JSON 串。路由规则如下：
* `score_record`：考试/录入意图（自动提取 `subject` 和 `score`）。
* `data_query`：单科成绩特定查询。
* `report_generation`：全局报表统计生成。
* `query_fail`：不及格异常预警监控。

### 🔀 2. 核心状态机节点对照表
工作流通过一个强大的 `Switch` 节点将大模型的意图完美分流至四大主干链路：

| 路由输出分支 | 触发条件 (Intent) | 绑定的后端业务节点 | 最终响应格式 |
| :--- | :--- | :--- | :--- |
| **Switch Output 1** | `score_record` | Airtable 查重 -> If 分支更新/创建 -> If1 预警 | 严格结构化 JSON |
| **Switch Output 2** | `data_query` | Search records2 节点精准匹配 | 🔍 查询结果文本 JSON |
| **Switch Output 3** | `report_generation` | Search records1 + 报表计算 JS 沙箱 | 📊 智能成绩单文本 JSON |
| **Switch Output 4** | `query_fail` | 过滤算法 `{score} < 60` + 防空壳防幽灵行算法 | ⚠️ 不及格预警文本 JSON |
| **Fallback Output**| * (其他泛化聊天) | Respond to Webhook5 兜底拦截 | “系统管家无法解决其他问题” |

---

## 💻 前端技术栈与核心代码规范

前端设计秉承**“学术克制、清晰至上”**的 Pro 级规范：
- **技术栈**：纯原生 HTML5 + 响应式 Flex 布局 + 玻璃拟态组件（CSS Backdrop Filter）。
- **会话持久化**：采用客户端生成随机 `SessionID` (`session_` + Math.random base36 截取)，确保长上下文的高效会话隔离。
- **UI 状态机动态反馈**：
  ```javascript
  // 前端根据后端吐出的报表文本关键词，秒级闪变气泡状态样式
  let status = (finalReport.includes("⚠️") || finalReport.includes("不及格")) ? 'danger' : '';
  if (finalReport.includes("✅") || finalReport.includes("成功")) status = 'success';
