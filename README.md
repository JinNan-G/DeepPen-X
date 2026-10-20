# DeepPen-X

DeepPen-X（Vehicle AI Security Test Platform）是一套本地部署的车载系统安全与 APP 安全分析平台，围绕自主渗透测试重建车联网安全边界：确定性分析负责刻画攻击面，多 Agent 团队负责自主渗透，独立证据裁决决定什么才算一个被发现的漏洞——每一条结论都附带可审计、可回放、SHA-256 锚定的证据。

平台默认后端运行在 `127.0.0.1:8000`，前端运行在 `127.0.0.1:5173`，数据库使用本地 SQLite。

本文档同时覆盖 macOS/Linux 和 Windows。所有命令都在项目根目录执行，除非单独说明。

## 平台能力总览
### 动态验证会话 Timeline（计划 / 工具 / 裁决节点）

（此处插入截图：动态验证会话 Timeline V2 全览）

<!-- ![动态验证会话 Timeline](docs/screenshots/dynamic-timeline.png) -->

### Planner 版本化攻击计划

（此处插入截图：Planner 生成的假设、证明义务、预期可观测与停止条件）

<!-- ![Planner 攻击计划](docs/screenshots/planner-plan.png) -->

### Executor 工具执行与 ToolRun 账本

（此处插入截图：Executor 经 ADB/SSH 对台架设备执行攻击动作及证据采集）

<!-- ![Executor 工具执行](docs/screenshots/executor-toolrun.png) -->

### Evidence Judge 独立裁决

（此处插入截图：Judge 对候选结论的裁决与定向补证指令）

<!-- ![Evidence Judge 裁决](docs/screenshots/judge-ruling.png) -->

### 断点恢复与事务化续跑

（此处插入截图：进程中断后的 unknown_side_effect 处理与恢复续跑）

<!-- ![事务化恢复](docs/screenshots/transactional-recovery.png) -->

### 漏洞回写与证据链

（此处插入截图：accepted 发现回写漏洞管理及完整证据链）

<!-- ![漏洞回写](docs/screenshots/discovery-writeback.png) -->

### 系统安全扫描与 23 类安全域攻击面

（此处插入截图：系统扫描攻击面覆盖 / 攻击图）

<!-- ![攻击面覆盖](docs/screenshots/attack-surface.png) -->

### APP 扫描与逐文件 AI 复核

（此处插入截图：APP 扫描三态 AI 复核与证据定位）

<!-- ![APP 扫描](docs/screenshots/app-scan.png) -->


### 三个平面：侦察 / 渗透 / 裁决

DeepPen-X 围绕一条原则构建严格分工：

- **确定性攻击面侦察（Recon）**：系统安全扫描、APP 扫描与二进制深度分析共同完成固件与 companion APP 的攻击面建模。攻击面按 23 类安全域注册表组织（引导信任链、凭据、OTA 完整性、UDS/DoIP/SOME-IP 协议栈、平台安全、无线等），保证没有安全域被静默挤掉。离线规则只负责提名候选；每个候选经三态 AI 复核（`confirmed` / `not_a_vulnerability` / `needs_more_context`）过滤可利用性闸门；存活事实以绑定产物 SHA-256、附带显式 `confirm_requires` 确认清单的渗透假设形式产出。
- **多 Agent 自主渗透（Penetration）**：动态验证是一个持久化的循环状态机，而不是一次性模型循环。Planner（无工具权限）输出不可变版本化攻击计划，包含假设、证明义务、替代路线、预期可观测现象和停止条件；Executor 是唯一持有工具的角色（ADB、SSH、串口、Frida、宿主命令、工作区文件），负责执行计划、回放凭据、驱动诊断会话、探测 IPC 面，并在设备暴露新信息时超出计划探索；一次渗透活动可绑定多个发现，并支持将多个孤立中危弱点组合为一条高危攻击链结论。
- **独立证据裁决（Adjudication）**：Evidence Judge 依据原始工具结果对每一个"已发现"声明独立裁决，路由只有四种：`accepted` / `repair_required`（定向补证）/ `replan_required` / `blocked`（人工介入）。按设计，`inconclusive` 不是可接受的结论。被接受的发现事务化回写漏洞管理，完整证据链（攻击计划、ToolRun 账本、Judge 裁决）随结论保留，可审计、可回放。

### 24 小时自主运行的事务化可靠性

平台以数据库事务的严谨性对待 Agent 行为：

- 每个攻击动作在执行前写入 ToolRun 账本；崩溃恢复按副作用类别（`read_only` / `idempotent_write` / `non_idempotent_write`）处理，写操作中途中断会标记 `unknown_side_effect`——禁止自动重放，升级 Judge 与人工对账。
- 双层记忆模型：权威状态由数据库事件重新生成，非权威工作记忆按 85%–95% 渐进压缩，内容寻址 blob 保存原始证据，压缩边界可校验，Compactor 带熔断，并支持手动 Compact/Snip。
- 自动上下文压缩后为任务持久化 Workspace Resume Checkpoint（记录接触过的文件路径、SHA-256、大小、计划版本、最近 ToolRun 与压缩边界）；续写必须经过 `workspace_status` 对比、重读、`expected_sha256` 校验，状态不符返回 `stale_file_state`，文本写入使用临时文件 + `fsync` + 原子替换。
- 支持 24 小时活跃周期、设备断线自动恢复、审批与工具事务恢复、同一任务多 cycle 续跑。

### 动态验证会话（Timeline V2）

动态会话只提供 Timeline V2，以类型化节点展示对话、计划、工具、审批、证据复核、裁决、公开状态和错误。原始诊断保留在后端审计数据中；对话与过程正文按清洗后的 GitHub Flavored Markdown 渲染；失败工具展示经凭据脱敏、长度受限的真实错误原因。升级已有数据库需先执行幂等迁移和只读检查（见下文"Dynamic Timeline V2 部署门禁"）。

### 多 Agent 协作与并发调度

限定漏洞验证按 Planner / Executor / Evidence Judge 三角色工单流程执行，自由探索保持独立对话入口。同一设备任务串行、不同设备并行，无设备自由对话使用独立队列；所有角色共享进程级和用户级 AI 并发预算。Agent Teams v2 对新建 Cycle 默认启用持久化专家队列。

## 功能截图



## 目录

项目根目录示例：

```text
fs-sensitive-ai-platform
```

macOS/Linux 进入目录：

```bash
cd /path/to/fs-sensitive-ai-platform
```

Windows PowerShell 进入目录：

```powershell
cd D:\path\to\fs-sensitive-ai-platform
```

## 依赖要求

- Python 3.11 或更新版本
- Node.js 20 或更新版本
- npm
- APP 扫描需要本机可执行的 `apktool`、`jadx`、`zip/unzip` 等工具；未安装时平台会提示工具缺失
- 动态验证与设备侧渗透依赖本机 `adb`、`ssh`、`sftp`，可选依赖串口工具和 Frida

## 首次初始化

### macOS/Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt

cd frontend
npm install
cd ..
```

创建 `.env.local`：

```bash
cat > .env.local <<'EOF'
APP_SECRET_KEY=change-this-local-secret
FIRST_ADMIN_USERNAME=admin
FIRST_ADMIN_PASSWORD=Admin@Fireware2026!
FS_SAMPLE_ASSET_PATH=/path/to/rootfs-or-sample-dir
LOCAL_AI_API_KEY=replace-with-local-ai-key
EOF
```

初始化数据库：

```bash
source .venv/bin/activate
python scripts/init_db.py
```

### Windows PowerShell

如果 PowerShell 不允许激活虚拟环境，可先在当前窗口临时放开执行策略：

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

创建虚拟环境并安装依赖：

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt

cd frontend
npm install
cd ..
```

创建 `.env.local`：

```powershell
@'
APP_SECRET_KEY=change-this-local-secret
FIRST_ADMIN_USERNAME=admin
FIRST_ADMIN_PASSWORD=Admin@Fireware2026!
FS_SAMPLE_ASSET_PATH=D:\samples\rootfs-or-sample-dir
LOCAL_AI_API_KEY=replace-with-local-ai-key
'@ | Set-Content -Encoding UTF8 .env.local
```

初始化数据库：

```powershell
.\.venv\Scripts\Activate.ps1
python scripts\init_db.py
```

`.env.local` 已加入 `.gitignore`，不要提交到代码仓库。

### Dynamic Timeline V2 部署门禁

动态会话只提供 Timeline V2。升级已有数据库时，先备份主数据库，再执行一次幂等迁移和只读检查；`check` 失败时后端会拒绝启动，不能跳过门禁部署：

```bash
.venv/bin/python -m dynamic_verify.timeline_migrate apply
.venv/bin/python -m dynamic_verify.timeline_migrate check
```

Windows PowerShell 使用：

```powershell
.\.venv\Scripts\python.exe -m dynamic_verify.timeline_migrate apply
.\.venv\Scripts\python.exe -m dynamic_verify.timeline_migrate check
```

迁移按任务独立事务执行，可安全重跑；原始 `dynamic_verify_step` 审计记录不会被改写。Timeline 通过类型化节点展示对话、计划、工具、审批、证据复核、裁决、公开状态和错误；原始诊断只保留在后端审计数据中，不向会话页面预加载。对话及过程正文统一按经过清洗的 GitHub Flavored Markdown 渲染；若 Provider 把 DSML 工具协议错误地放进文本字段，后端会恢复为真实工具调用，历史内容在重投影时转换为可读 Markdown。失败工具直接展示经过凭据脱敏、长度受限的实际错误原因，管理员占位文案和内部错误类型均不进入公共载荷；失败流中只有"接下来执行"等承诺而没有结果的短暂进度片段不会进入公开 Timeline。目标契约、验收账本、运行时错误控制和重试状态仅属于后端编排数据，不出现在普通任务响应或页面中。迁移检查会阻止包含上述占位文案、内部错误类型、原始 DSML 或无效失败进度的投影通过部署门禁。旧 `/steps`、`/transcript` 和 `/ws/dynamic/{task_id}` 不再提供服务。

动态验证的"完全访问权限"（`autonomous`）会直接执行宿主和设备命令：`adb_shell` 不使用只读命令白名单，也不会拒绝 `getenforce`、`uname`、管道、命令拼接或设备写命令；`host_exec`、`device_exec` 同样不经过命令黑名单、目标包检查或人工审批。较低权限仍会返回审批请求而不是伪装成命令执行失败。任意 ADB shell 均按非幂等操作串行记录，避免断线恢复时自动重放可能产生副作用的命令。

## 前台启动

适合开发调试，终端关闭后服务也会停止。

### macOS/Linux

启动后端：

```bash
source .venv/bin/activate
python -m uvicorn main:app --host 127.0.0.1 --port 8000
```

启动前端，另开一个终端：

```bash
cd frontend
npm run dev -- --host 127.0.0.1 --port 5173
```

### Windows PowerShell

启动后端：

```powershell
.\.venv\Scripts\Activate.ps1
python -m uvicorn main:app --host 127.0.0.1 --port 8000
```

启动前端，另开一个 PowerShell：

```powershell
cd frontend
npm run dev -- --host 127.0.0.1 --port 5173
```

浏览器访问：

```text
http://127.0.0.1:5173
```

后端接口文档：

```text
http://127.0.0.1:8000/docs
```

## 后台启动

### macOS/Linux

```bash
mkdir -p logs
nohup .venv/bin/python -m uvicorn main:app --host 127.0.0.1 --port 8000 > logs/backend.log 2>&1 &

cd frontend
mkdir -p ../logs
nohup npm run dev -- --host 127.0.0.1 --port 5173 > ../logs/frontend.log 2>&1 &
cd ..
```

### Windows PowerShell

```powershell
New-Item -ItemType Directory -Force logs | Out-Null

Start-Process -FilePath ".\.venv\Scripts\python.exe" `
  -ArgumentList "-m","uvicorn","main:app","--host","127.0.0.1","--port","8000" `
  -RedirectStandardOutput "logs\backend.log" `
  -RedirectStandardError "logs\backend.err.log" `
  -WindowStyle Hidden

Start-Process -FilePath "npm.cmd" `
  -ArgumentList "run","dev","--","--host","127.0.0.1","--port","5173" `
  -WorkingDirectory ".\frontend" `
  -RedirectStandardOutput "logs\frontend.log" `
  -RedirectStandardError "logs\frontend.err.log" `
  -WindowStyle Hidden
```

## 查看服务状态

### macOS/Linux

```bash
lsof -nP -iTCP:8000 -sTCP:LISTEN
lsof -nP -iTCP:5173 -sTCP:LISTEN
tail -n 80 logs/backend.log
tail -n 80 logs/frontend.log
```

### Windows PowerShell

```powershell
Get-NetTCPConnection -LocalPort 8000 -State Listen | Select-Object LocalAddress,LocalPort,OwningProcess
Get-NetTCPConnection -LocalPort 5173 -State Listen | Select-Object LocalAddress,LocalPort,OwningProcess
Get-Content logs\backend.log -Tail 80
Get-Content logs\frontend.log -Tail 80
```

## 停止服务

### macOS/Linux

```bash
BACKEND_PID=$(lsof -tiTCP:8000 -sTCP:LISTEN)
[ -n "$BACKEND_PID" ] && kill "$BACKEND_PID"

FRONTEND_PID=$(lsof -tiTCP:5173 -sTCP:LISTEN)
[ -n "$FRONTEND_PID" ] && kill "$FRONTEND_PID"
```

如果进程没有退出，可把 `kill` 换成 `kill -9`。

### Windows PowerShell

```powershell
$backend = Get-NetTCPConnection -LocalPort 8000 -State Listen -ErrorAction SilentlyContinue
if ($backend) { Stop-Process -Id $backend.OwningProcess -Force }

$frontend = Get-NetTCPConnection -LocalPort 5173 -State Listen -ErrorAction SilentlyContinue
if ($frontend) { Stop-Process -Id $frontend.OwningProcess -Force }
```

## 重启服务

更新代码后必须同时重启前、后端。这里的 `uvicorn` 默认没有启用
`--reload`；如果 8000 端口仍由旧进程占用，新前端会连接到旧后端，典型现象是
动态测试会话无法加载、Timeline 接口返回 404，或模型配置提交返回 422。不要只刷新
浏览器或只重启前端。

### macOS/Linux

```bash
BACKEND_PID=$(lsof -tiTCP:8000 -sTCP:LISTEN)
[ -n "$BACKEND_PID" ] && kill "$BACKEND_PID"

FRONTEND_PID=$(lsof -tiTCP:5173 -sTCP:LISTEN)
[ -n "$FRONTEND_PID" ] && kill "$FRONTEND_PID"

sleep 1
mkdir -p logs
nohup .venv/bin/python -m uvicorn main:app --host 127.0.0.1 --port 8000 > logs/backend.log 2>&1 &

cd frontend
mkdir -p ../logs
nohup npm run dev -- --host 127.0.0.1 --port 5173 > ../logs/frontend.log 2>&1 &
cd ..
```

### Windows PowerShell

```powershell
$backend = Get-NetTCPConnection -LocalPort 8000 -State Listen -ErrorAction SilentlyContinue
if ($backend) { Stop-Process -Id $backend.OwningProcess -Force }

$frontend = Get-NetTCPConnection -LocalPort 5173 -State Listen -ErrorAction SilentlyContinue
if ($frontend) { Stop-Process -Id $frontend.OwningProcess -Force }

Start-Sleep -Seconds 1
New-Item -ItemType Directory -Force logs | Out-Null

Start-Process -FilePath ".\.venv\Scripts\python.exe" `
  -ArgumentList "-m","uvicorn","main:app","--host","127.0.0.1","--port","8000" `
  -RedirectStandardOutput "logs\backend.log" `
  -RedirectStandardError "logs\backend.err.log" `
  -WindowStyle Hidden

Start-Process -FilePath "npm.cmd" `
  -ArgumentList "run","dev","--","--host","127.0.0.1","--port","5173" `
  -WorkingDirectory ".\frontend" `
  -RedirectStandardOutput "logs\frontend.log" `
  -RedirectStandardError "logs\frontend.err.log" `
  -WindowStyle Hidden
```

重启后可确认当前后端已经提供动态会话接口：

```bash
curl -fsS http://127.0.0.1:8000/openapi.json | python -c "import json,sys; p=json.load(sys.stdin)['paths']; assert '/api/dynamic/tasks/{task_id}/timeline' in p; print('Dynamic Timeline V2 ready')"
```

Windows PowerShell：

```powershell
(Invoke-RestMethod http://127.0.0.1:8000/openapi.json).paths.PSObject.Properties.Name |
  Select-String '/api/dynamic/tasks/\{task_id\}/timeline'
```

## 常用检查

后端测试：

```bash
pytest -q
```

ELF 深度分析专项测试：

```bash
pytest tests/test_binary_analysis.py -q
```

前端构建：

```bash
cd frontend
npm run build
```

## 关键路径

- 数据库：`data/platform.db`
- 已完成任务归档库：`data/tasks/fs-<task_uuid>.db`、`data/tasks/app-<task_uuid>.db`
- 后端日志：`logs/backend.log`
- 前端日志：`logs/frontend.log`
- 扫描工作目录：`workspace/`
- 报告输出目录：`workspace/reports/`
- 本地密钥配置：`.env.local`

已完成的系统安全或 APP 扫描可在任务列表中主动归档。平台会先生成独立 SQLite 数据库、校验完整性和行数，再释放主库中的完整证据、文件级 AI 结果、检查点及大体积审计上下文；任务和漏洞均保留稳定 UUID，现有数字 ID/API 保持兼容，任务列表、漏洞评审、攻击图和单任务报告仍可透明读取归档结果。归档库不包含用户密码、AI API Key 或其它任务数据，可在任务列表中下载成 `fs-<task_uuid>.db` 或 `app-<task_uuid>.db` 后单独分享。

平台启动和任务列表刷新时会自动扫描 `data/tasks/`。将 `fs-<task_uuid>.db` 或 `app-<task_uuid>.db` 放入该目录后，平台会自动把任务、资产占位、漏洞索引和轻量 AI 结论注册到主库；若本机已有相同数字 ID，会自动重映射本地主库和归档库中的任务/漏洞 ID。也可以在系统扫描或 APP 扫描页面点击"导入归档"，选择别人分享的 `.db` 文件，平台会先校验归档类型和 SQLite 完整性，再复制到 `data/tasks/` 并立即加载。移除对应归档文件后，平台会自动清理主库中的挂载任务、漏洞索引和自动创建的归档资产占位。

删除任务会同步清理该任务在受管报告目录中的单任务报告和对应归档数据库。SQLite 空间整理在后台空闲时执行；若仍有扫描运行，会记录待整理状态并在最后一个扫描任务结束后继续。备份平台时应同时备份 `data/platform.db` 和 `data/tasks/`。

Windows 上扫描大型或深层 APK 时，建议把 `config/app_config.yaml` 里的
`workspace` 配置为短路径，例如 `C:\fs_ws`，并启用 Windows
`LongPathsEnabled`。如果后端运行在 macOS/Linux，请不要直接填写
`D:\...` 这类 Windows 盘符路径；请改用后端机器可访问的挂载路径，例如
WSL/Linux 下的 `/mnt/d/...`。

### 动态验证宿主命令的跨平台行为

动态验证会把当前宿主系统、CPU 架构、路径分隔符、默认 Shell 和实际可用
Shell 注入 Executor 上下文。`host_exec` 支持显式选择 `auto`、`cmd`、
`powershell` 或 `posix`：

- Windows 的 `auto` 保持兼容地使用 `cmd.exe`；复杂脚本应选择
  `powershell`。平台通过显式 argv 启动 Shell，不再依赖 Python
  `shell=True` 的隐式解释行为。
- macOS/Linux 的 `auto` 使用 POSIX Shell。只有机器上实际发现
  `PowerShell/pwsh` 时才允许选择 `powershell`。
- Agent 不应在 Windows 混用 `ls/grep/sed`、`$VAR`、POSIX 单引号或
  `/tmp/...`，也不应在 macOS/Linux 使用 `%VAR%`、`dir/findstr` 或盘符路径。
- 跨平台文件读写优先使用 `read_file`、`write_file`、`edit_file`、
  `list_files`；确实需要脚本时优先使用已发现的 Python 和工作区相对路径。
- Frida 使用参数数组直接启动，因此 Windows 工具路径或脚本路径包含空格时
  不再经过 POSIX 引号拼接。

可通过动态设备接口检查后端实际识别的执行环境：

```text
GET /api/dynamic/devices
```

响应中的 `host` 字段会给出 `system`、`family`、`machine`、
`default_shell` 和 `shells`。Windows 部署完成后应至少确认 `cmd` 可用；
需要 PowerShell 动态脚本时还应确认 `powershell` 出现在 `shells` 中。

### 自动压缩后的代码/文本安全续写

Executor 或自由会话触发自动上下文压缩时，平台会为当前任务建立持久化的
Workspace Resume Checkpoint。检查点记录本周期通过 `read_file`、
`write_file`、`edit_file` 接触过的文件路径、SHA-256、大小、计划版本、
最近 ToolRun 和压缩边界；原始步骤和工作区文件不会被摘要替换。

压缩后继续写入时执行以下门禁：

1. 必须先调用 `workspace_status` 对比检查点和当前工作区。
2. 修改检查点中的已有文件前必须重新调用 `read_file`。
3. `write_file` 和 `edit_file` 必须传入读取结果中的 `expected_sha256`；
   创建新文件时传空字符串。
4. 文件状态不匹配时返回 `stale_file_state`，要求重新读取，禁止盲目覆盖。
5. 文本写入使用同目录临时文件、`fsync` 和原子替换；中断不会留下半个目标文件。
6. `host_exec` 无法静态确定写集合，因此压缩后必须先恢复检查点中的全部已有文件，
   才允许继续执行。

## 有效漏洞与误报控制

- 平台使用统一漏洞评级标准 `unified-risk-scoring-v1`。离线规则只负责识别候选文件和候选证据、划分文件风险、提高 AI 送审优先级；规则命中不能直接确认漏洞，也不能直接创建 `needs_confirmation/open/confirmed` 漏洞。系统安全扫描的公开漏洞必须由 AI 文件级复核输出 `confirmed` 并通过平台证据闸门后，才能进入漏洞简介、漏洞清单、评审、报表和看板。最终入库的 `severity`、`risk_score` 由后端统一评分函数计算：规则候选提供本地风险分量，AI 返回的 `risk_level`、`risk_score` 提供模型分量，再按统一等级区间定级。
- 等级区间固定为：严重 90-100，高危 70-89，中危 40-69，低危 10-39，信息 0-9。同一个任务、同一漏洞、同一证据在重复扫描时必须得到相同等级和分数。
- 基础分由漏洞类型决定：外部无鉴权命令执行、APK/rootfs 内置真实私钥为严重基线；TLS 证书校验绕过、硬编码可复用密码/Token/API Key、WebView 高危配置为高危基线；组件暴露、明文通信、弱加密、敏感日志、车联网功能攻击面按证据链完整度定级。
- 加分项固定来自结构化证据：外部可达、鉴权缺失或无效、外部输入到敏感操作的数据流、影响车控/诊断/OTA/系统权限、真实生产凭据或私钥。模型置信度不能替代这些证据。
- 降级和封顶规则固定：普通 DeepLink、Binder/IPC 或未证明导出缺权限的组件入口仍属于攻击面事实，不进入正式漏洞；证据不完整或量产构建未知进入待补充上下文；公共证书、默认 CA、示例/占位值、debug/demo 专用配置不提升为高风险漏洞。导出组件缺权限的已确认结论缺少完整利用链时最高按中危，不再被压到信息级。
- APP 扫描会先识别 AI、语音交互、调试测试、工程/诊断和特权车控等高危敏感功能。普通第三方应用的 `exported=true`、隐式导出且未声明 `permission`（或仅 `normal`/`dangerous` 自定义权限）的组件属于可确认的 CWE-926 组件暴露/权限控制缺陷，任意应用都能调用该入口；完整调用链只影响定级，不决定是否成立。Activity/Receiver 通常按低危到中危、Service/Provider 按中危到高危，能证明外部输入到达文件、命令、车控、诊断、录屏、凭据等敏感能力时升级。
- 平台/厂商系统包（如 `com.qualcomm.qti.*`、`com.android.*`、`sharedUserId=android.uid.system` 等）的 exported 组件仍遵循平台权限模型例外：其访问通常由平台签名权限、特权授权和 SEPolicy 保护，仅凭 Manifest 声明确认前必须基于组件代码证明平台级保护缺失/无效。
- `sharedUserId` 是车机应用身份上下文，不单独归为漏洞；若其它漏洞成立，只用于评估其影响范围。
- 系统安全扫描会从 `build.prop` 等属性文件识别 `user`、`userdebug`、`eng`。只有确认是 `user` 量产构建时，才把 ADB/root、`ro.debuggable`、`ro.secure`、SELinux permissive、verity/AVB、test-keys 等构建类型相关安全配置归为漏洞。该策略不豁免硬编码凭据、私钥、弱 TLS、未授权接口或命令执行等独立风险。
- 系统安全扫描默认不限制 AI 文件数量：所有已枚举且可读取的系统代码、脚本、配置、数据库、日志、证书/密钥和 ELF 结构化证据都会进入 AI 文件级安全分析；攻击面注册表只作为分类、排序、进度和攻击图标签，不作为送审准入、漏洞类型白名单或逐文件检查上限。`.js/.html/.lua` 按当前实测策略默认剔除，仍会在枚举事件中计数说明。
- 系统与 APP 使用独立提示词；模型结论采用 `confirmed`、`not_a_vulnerability`、`needs_more_context` 三态，并经过原始证据定位、关联文件和平台确定性策略复核。
- 每个 APP 主文件使用独立、可持久化的 `review_session_id`。同一文件的多轮分析严格串行，不同文件可以并发；下一轮会携带上一轮结构化结论，后端按稳定 `finding_id` 合并，模型后续省略或返回空列表不会覆盖已确认且证据落地的漏洞。
- 主文件按带重叠和原始行号/offset 的源码块逐段分析，不再在上下文超限时只保留文件头。关联文件优先提取请求 symbol 所在完整方法；提示词再次超限时保留源码首尾、规则/历史证据锚点和历史 finding 账本，完整未压缩状态保存在 APP 检查点中。
- 每轮结束会持久化结论、完整证据字段、轮次历史、已完成源码块和关联 Artifact 哈希。服务重启或外层重试会恢复同一会话；主文件或任一关联文件哈希变化时旧会话自动失效并重新分析。`error` 检查点保持可重试，只有哈希一致的 `completed` 文件才会断点跳过。
- 关联文件解析只允许访问当前 APP 任务的 `jadx`、`apktool`、`unzip` 根目录，拒绝模型提供的绝对路径；每条最终 AI 证据绑定真实 `artifact_id`、相对路径、SHA-256 和原始行号，不能把关联类证据错误记到主文件。
- APP 不再执行扫描前的大模型 Agent 规划，解包和静态候选完成后直接进入逐文件代码分析。文件分析共享进程级全局 AI 并发上限；单任务 worker 只是第二级限制。单次扫描复用网络连接池，超时或取消会等待在途调用结束清理，避免多任务并发放大连接和限流故障。
- 蓝牙/Wi-Fi/UDS/DoIP 诊断、凭据/密钥、Native/JNI 新增专项候选规则；这些规则只负责定位高风险上下文，仍需完整源码证据、关联文件和 AI 三态结论才能进入最终漏洞报告。
- 登录、工程模式、工厂模式和诊断入口会确定性检查固定口令直接比较、固定口令哈希比较，以及 PKI/动态密钥失败后的后备认证分支；模型旧格式中明确的 `has_vulnerability=true` 结论仍会进入原始证据定位流程，不再因缺少逐项 `decision` 字段直接丢弃。
- HTTPS/MQTT/UDS/DoIP/OTA 等协议名称、公共 URL、公共证书和 API 名称只作为上下文候选，不能单独进入漏洞报告。

## 二进制深度分析

- 系统安全扫描对 ELF 文件执行结构化静态分析，不再只做字符串提取。分析内容包括：ELF 头/节/段、动态表、导入导出符号、重定位与 PLT 解析、函数边界恢复、危险 API 调用点反汇编、参数回溯、带地址归属的字符串与代码交叉引用、加固姿态（RELRO/NX/PIE/canary/FORTIFY/RPATH）。
- 全部基于 Python 标准库实现，不需要 IDA、Ghidra、JDK 或任何额外 wheel，macOS/Windows/Linux 行为一致。安装 `capstone` 可让反汇编切片的助记符更完整，但语义事实（调用目标、恢复出的参数、解析出的常量）始终由内置解码器给出，不受可选依赖影响。
- 支持架构：AArch64、ARM（A32/Thumb-2）、x86-64、i386、MIPS32/64，含大端序与 MIPS 分支延迟槽。
- 漏洞挖掘覆盖四类：内存安全（栈/堆溢出、格式化字符串、整数溢出、UAF 模式）、命令注入与危险执行、硬编码凭据与密码学误用、车载协议实现安全（UDS/DoIP/SOME-IP/MQTT/CAN）。
- 分析分阶段执行以控制成本：先解析结构与符号表，仅凭导入表判断是否存在风险类别，只有存在时才扫描代码段。不含危险 API、弱算法、凭据特征和协议符号的二进制自动跳过反汇编，仍保留完整结构证据。
- 深度 ELF 分析具备安全降级机制。正常情况下 `scan_one_file` 会写入结构化 `binary_analysis` 并把带地址归属的字符串继续送入原有规则引擎；如果未来某个畸形 ELF 或解码器边角触发异常，系统不会让该文件扫描失败，也不会中断任务，而是自动回退到原有 `extract_strings` 路径，并在 `binary_analysis.errors` 中记录降级原因。
- 调用点只产生"候选点"，不产生漏洞。每个候选点同时携带确认所需的证据链（`confirm_requires`）和判定为非漏洞的常见理由（`benign_if`）。仅出现 `strcpy`、`system`、`MD5` 等符号或调用点，一律不构成漏洞。
- 内存安全确认必须同时给出目标缓冲区边界依据、写入长度不受该边界限制的依据、以及长度可由外部输入影响的入口，三段缺一即 `needs_more_context`。命令执行必须区分固定常量命令、含 `%s`/`$` 的模板和运行期构造三种情况。凭据常量必须给出引用它的函数与用途。协议识别只作为攻击面事实。
- 参数回溯跨过跳转目标时标记 `argument_confidence=path_sensitive`，表示所列取值只在其中一条控制流路径上成立；反汇编或字符串提取被预算截断时标记 `truncated`，模型不得声称已覆盖全文件。
- `_FORTIFY_SOURCE` 包装函数（`__strcpy_chk`、`__sprintf_chk` 等）单独建模：参数位置与原函数不同，且越界后果是进程主动中止而非内存破坏，不按栈溢出定级。
- 加固属性只影响定级不影响成立。缺少 PIE、RELRO 不完整、无栈保护属于攻击面事实，只有在同一文件已证实可达内存破坏时才作为危害升级依据。
- 可通过 `config/app_config.yaml` 的 `app.elf_deep_analysis_enabled` 关闭，关闭后回退为原有字符串提取。
- 可用性回归由 `tests/test_binary_analysis.py` 覆盖，包括结构解析、架构解码、候选点证据、误报闸门、关闭深度分析后的旧路径，以及深度分析异常时自动回退到字符串扫描，确保 ELF 能力不会影响系统安全扫描主流程。

实现说明、误报闸门清单和测试策略见 [二进制深度分析与 AI 漏洞挖掘](docs/binary-deep-analysis.md)。

## 动态验证 Agent 角色与长期闭环

限定漏洞验证按三个逻辑角色执行，角色共享同一任务、模型配置和 AI 并发预算：

1. **Planner / Coordinator**：无工具权限；根据漏洞上下文生成结构化假设、证明义务、依赖、替代路线、预期现象和停止条件。计划以不可变版本持久化，重规划时创建新版本。
2. **Executor**：唯一拥有工具权限的角色；按计划收集证据并提交候选结论。APP 与固件/系统分别加载 `android_app`、`firmware_system` playbook。
3. **Evidence Judge**：无工具权限；独立检查候选结论、计划覆盖和真实工具结果，路由到接受、定向补证、重规划或人工阻塞。最终回写建议只来自 accepted 裁决。

限定漏洞验证是数据库驱动的循环状态机，而不是一次性模型循环：

```text
Planner(vN) -> Executor -> Evidence Judge
                         |- accepted -> completed (verified/not_verified)
                         |- repair_required -> Executor 定向补证
                         |- replan_required -> Planner(vN+1)
                         `- blocked -> manual_required，可新建 cycle 续跑
```

正常流程不再以 `inconclusive` 完成。Executor 可以把它作为检查点候选交给 Judge，但 Judge 必须路由到补证、重规划或人工阻塞。`failed` 仅表示平台不变量或内部编排故障。

动态验证支持持久化 `Planner → Executor → Evidence Judge` 循环、Judge 定向补证/重规划、24 小时活跃周期、设备断线自动恢复、审批与工具事务恢复，以及同一任务多 cycle 续跑。上下文管理采用权威状态/非权威工作记忆双层模型、85%–95% 渐进压缩、内容寻址 blob、可校验压缩边界、Compactor 熔断和手动 Compact/Snip。

多 Agent 团队（Agent Teams v2）默认对新建 Cycle 启用持久化专家队列，简单任务无需创建专家，已有 Cycle 的引擎不变。所有角色输入中的漏洞文本、反编译内容、日志和工具输出均按不可信数据处理，不能改变角色职责、执行边界或输出契约。

详细问题、整改项和后续质量门禁见 [系统安全与 APP 安全提示词/Agent 审计报告](docs/security-scan-agent-audit.md) 和 [APP 扫描 AI 并发、多轮上下文整改验收](docs/app-scan-ai-concurrency-audit.md)。

## 安全说明

- `.env.local` 用于保存本地管理员密码、AI API key 和运行配置，禁止提交。
- 完整证据查看、报告导出、AI 配置、规则修改和用户管理均需要权限。
- AI 不可用时不会把离线规则命中上报为漏洞；离线规则只作为 AI 候选线索，需恢复 AI 后重新确认。
- 普通系统安全扫描不会解包 APK；APK 解包和代码分析只在 APP 扫描模块执行。
- 动态验证的 `autonomous` 权限会直接执行宿主和设备命令，仅面向隔离的测试台架与授权设备使用；所有命令均按非幂等操作串行记入 ToolRun 账本，可审计、可回放。
