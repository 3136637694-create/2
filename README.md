# EdgeVision 边缘端实时视觉分析系统

基于 RTSP / 本机摄像头视频流的边缘端实时视觉分析平台：视频采集 → 目标检测（YOLO）→ 多目标跟踪（SORT）→ 事件分析 → 实时告警，提供 REST 接口与 WebSocket 实时数据通道，配套暗色系监控台 Web 前端。

适用于校园安防、工地安全、客流统计、交通卡口等场景，把人力从持续盯屏中解放出来。

---

## 1. 功能特性

- **视频采集**：支持本机摄像头（`0`、`1`…）与 RTSP 网络流（`rtsp://…`）；断线自动指数退避重连
- **AI 流水线**：YOLO 目标检测 + SORT 多目标跟踪（稳定目标 ID）+ 规则状态机事件识别（区域闯入 / 人员聚集 / 遗留物）
- **实时画面**：检测框、目标 ID、运动轨迹叠加；MJPEG 视频流（`/stream/{id}`）与 Canvas 监控画面
- **可视化与告警**：监控台四页面——实时监控、摄像头管理、告警事件、系统指标；WebSocket 实时推送检测结果与告警
- **对外接口**：REST（摄像头 CRUD / 事件分页查询 / 系统指标）+ WebSocket（`/ws/monitor`）+ Swagger 自动接口文档（`/docs`）
- **数据持久化**：SQLite 存储摄像头配置、告警事件、性能指标（丢帧、队列占用、端到端延迟）；全部参数化 SQL 防注入
- **双模式**：前端 Mock / API 一键切换，无后端也可预览演示效果

## 2. 技术栈

| 层 | 组件 | 说明 |
| --- | --- | --- |
| 后端 | FastAPI + Uvicorn | REST / WebSocket / 自动接口文档 |
| 采集 | OpenCV (cv2) | 摄像头 / RTSP 读取，DSHOW 后端（Windows 本机摄像头） |
| 推理 | ONNX Runtime | YOLO 模型推理（模型缺失自动回退演示检测） |
| 跟踪 | 自研 SORT | 匈牙利匹配多目标跟踪 |
| 存储 | SQLite | 本地文件库（WAL 模式） |
| 前端 | Vue3 + TypeScript + Element Plus + ECharts | 暗色系监控台，Vite 构建 |

## 3. 目录结构

```
EdgeVision/
├── README.md            # 本文档
├── 需求.md              # 需求规格说明
├── frontend/            # 前端（Vue3 + Vite）
│   ├── src/             # 源码（布局 / 四页面 / API 桩层 / Mock 引擎 / WS 封装）
│   └── dist/            # 构建产物（npm run build 生成）
└── backend/             # 后端（FastAPI）
    ├── app/
    │   ├── main.py          # 应用入口（lifespan 接线 / CORS / 路由）
    │   ├── config.py        # 全局配置
    │   ├── db.py            # SQLite 初始化（建表 / WAL）
    │   ├── models.py        # Pydantic 数据契约（snake→camel）
    │   ├── repositories.py  # 参数化 SQL 仓储层
    │   ├── manager.py       # 流水线管理器（多路摄像头）
    │   ├── pipeline.py      # 采集→推理→跟踪→事件 流水线
    │   ├── metrics.py       # 系统指标采样器
    │   ├── sources/         # 采集源抽象（RTSP / 本机摄像头 / 模拟源）
    │   ├── ai/              # 检测器 / SORT / 事件状态机
    │   └── api/             # REST / WebSocket / MJPEG 路由
    ├── data/                # SQLite 数据文件（运行时生成）
    ├── models/              # 模型目录（放入 yolov8n.onnx 启用真实推理）
    ├── venv/                # Python 虚拟环境
    └── pyproject.toml       # 依赖清单
```

## 4. 快速部署

### 4.1 环境要求

- Python ≥ 3.10
- Node.js ≥ 18
- （可选）本机摄像头 / RTSP 视频源

### 4.2 启动后端（端口 8000）

```bash
cd backend

# 首次运行：创建虚拟环境并安装依赖
python -m venv venv
venv\Scripts\activate           # Windows；macOS/Linux: source venv/bin/activate
pip install -e .

# 启动服务
venv\Scripts\python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

启动后访问：

- 接口文档（Swagger UI）：http://localhost:8000/docs
- 健康检查：http://localhost:8000/api/health

### 4.3 启动前端（端口 5173）

```bash
cd frontend
npm install
npm run dev        # 开发模式
# npm run build && npm run preview   # 生产构建预览
```

浏览器打开 http://localhost:5173 ，顶部开关可切换 **Mock / API** 模式：

- **Mock 模式**：内置模拟引擎生成数据，无需后端即可演示
- **API 模式**：对接后端 FastAPI（`/api`、`/ws` 已由 Vite 代理转发到 8000）

## 5. 接入本机摄像头

在「摄像头管理」页新增 / 编辑摄像头，`source` 字段按以下规则填写：

| source | 含义 |
| --- | --- |
| `0`（或 `1`、`2`…） | 本机摄像头索引（Windows 使用 DirectShow + MJPG，640×480@15fps） |
| `rtsp://user:pass@ip:554/stream` | RTSP 网络视频流 |
| `sim:parking`（`sim:xxx`） | 模拟视频源（演示 / 无摄像头环境） |

示例：`source = "0"` 即打开本机默认摄像头，保存后「实时监控」页即可看到真实画面。

> 说明：本机摄像头在采集子线程中打开，已在 Windows 下完成 COM 初始化（STA / DirectShow 后端适配），实测稳定出帧；画面断线时系统自动进入退避重连。

## 6. 对外接口

### REST（前缀 `/api`）

| 方法与路径 | 说明 |
| --- | --- |
| `GET /api/health` | 健康检查 |
| `GET /api/cameras` | 摄像头列表 |
| `POST /api/cameras` | 新增摄像头（自动启动流水线） |
| `PUT /api/cameras/{id}` | 更新摄像头（源 / 帧率变化自动重启） |
| `DELETE /api/cameras/{id}` | 删除摄像头（自动停止流水线） |
| `GET /api/events` | 告警事件分页查询（`type` / `level` / `camera_id` / `page` / `page_size`） |
| `GET /api/metrics` | 最近 60 条系统指标（按时间升序） |
| `GET /api/system/stats` | 系统概览统计 |
| `GET /stream/{id}` | MJPEG 实时视频流 |
| `WS /ws/monitor` | WebSocket 实时推送（检测帧 / 告警事件 / 系统指标） |

完整接口定义与联调调试请访问 Swagger UI：http://localhost:8000/docs

### 数据契约（camelCase）

- **CameraInfo**：`{ id, name, location, source, status, fps, resolution, enabled, lastSeen }`
- **AlertEvent**：`{ id, cameraId, cameraName, type, level, message, targets, ts }`
- **Metric**：`{ id, ts, e2eLatencyMs, droppedFrames, queueCapacity, queueUsage, fps, detectionMs }`
- **Detection**：`{ id, cameraId, ts, trackId, cls, confidence, bbox }`

## 7. 模型接入

默认使用演示检测器（可完整跑通界面与流水线）。启用真实 YOLO 推理：

1. 下载 YOLOv8n ONNX 模型，命名为 `yolov8n.onnx`
2. 放入 `backend/models/` 目录
3. 重启后端，`GET /api/system/stats` 中 `model` 字段即显示真实推理

采集源与检测器均基于抽象接口实现（`FrameSource` / `Detector`），可替换为任意实现。

## 8. 可观测性

- **端到端延迟**：帧入队 → 处理完成的毫秒耗时
- **丢帧保护**：采集队列满时丢弃并计数，防止内存上涨
- **队列占用**：当前队列 / 容量（`queueUsage`）
- **帧率**：最近 1 秒实际处理帧数
- 以上指标每 5 秒采样入库，前端「系统指标」页折线图实时展示

## 9. 常见问题

| 问题 | 处理 |
| --- | --- |
| 摄像头状态一直是 `reconnecting` | 检查 source 是否为本机摄像头索引（纯数字）或有效 RTSP 地址；本机摄像头被其他程序占用时需先释放 |
| 画面显示 SIMULATED | 该路摄像头打开失败已自动回退模拟源，检查摄像头是否可用 |
| 前端画面空白 | 确认顶部已切换到 API 模式，且后端在 8000 端口运行 |
| 修改后端代码不生效 | uvicorn 未开 `--reload`，需重启进程 |
| 想用真实检测模型 | 参照「模型接入」章节放入 onnx 模型 |
