# SakuraFRP Auto Checkin（NVIDIA NIM 版）

SakuraFRP 每日自动签到，GitHub Actions 定时运行，验证码识别走 NVIDIA NIM 的 OpenAI 兼容接口。

> 本仓库 fork 自 [XavierJiezou/SakuraFRP-Daily-Checkin](https://github.com/XavierJiezou/SakuraFRP-Daily-Checkin)，
> 其原始项目为 [RyanStarFox/SakuraFRP_Auto_AI_check](https://github.com/RyanStarFox/SakuraFRP_Auto_AI_check)。
> 本仓库在原版基础上更换了 AI 服务商，并默认关闭邮件通知。

## 与上游的差异

| 项目 | 上游 | 本仓库 |
| --- | --- | --- |
| AI 服务商 | 智谱 [open.bigmodel.cn](https://open.bigmodel.cn/) | NVIDIA NIM `https://integrate.api.nvidia.com/v1` |
| SDK | `zhipuai>=2.0.0` | `openai>=1.0.0` |
| 视觉模型 | `glm-4.6v-flash`，备用 `glm-4v-flash` | `nvidia/nemotron-3.5-super-vl-120b-a12b`，备用 `z-ai/glm-5.3-flash` |
| 文本模型 | `glm-4.7-flash` | `z-ai/glm-5.3-flash` |
| 邮件通知 | 默认 `failure_only` | 默认关闭（`EMAIL_NOTIFY_ENABLED=false`） |
| 逐格并发 | `AI_CELL_WORKERS=3` | `2`（降低免费层触发限流的概率） |

功能本身未改动：Playwright 登录并签到、九宫格与滑块验证码识别、单次执行内多轮重试、失败时返回非零退出码、日志与截图上传为 artifact。

## GitHub Actions 使用

### 必需的 Secrets

`Settings` → `Secrets and variables` → `Actions` → `Secrets` → `New repository secret`

| 名称 | 说明 |
| --- | --- |
| `SAKURA_USERNAME` | 樱花内网穿透账号 |
| `SAKURA_PASSWORD` | 对应密码 |
| `NVIDIA_API_KEY` | [build.nvidia.com/settings](https://build.nvidia.com/settings) 生成，形如 `nvapi-...` |
| `HTTP_PROXY` | 可选，纯 curl 形式的代理地址，如 `http://127.0.0.1:7890` |

邮件相关的 `EMAIL_SMTP_USER` / `EMAIL_SMTP_PASSWORD` / `EMAIL_TO` 无需配置——本仓库已默认关闭邮件通知。

### 可选 Variables

`Settings` → `Secrets and variables` → `Actions` → `Variables` 标签。工作流里写作 `${{ vars.X || '默认值' }}`，**变量优先于代码默认值**，改完下一次运行即生效，无需改代码。

| 变量名 | 作用 | 默认 |
| --- | --- | --- |
| `NVIDIA_MODEL_VISION` | 九宫格 / 滑块的主视觉模型 | `nvidia/nemotron-3.5-super-vl-120b-a12b` |
| `NVIDIA_MODEL_VISION_FALLBACK` | 主模型 429 / 5xx 时自动切换 | `z-ai/glm-5.3-flash` |
| `NVIDIA_MODEL_TEXT` | 只吃文本的语义裁决 | `z-ai/glm-5.3-flash` |
| `EMAIL_NOTIFY_POLICY` | 若重新开启邮件，控制打扰程度 | `failure_only` |

### 触发方式

- **定时**：`cron: '7 4 * * *'`，即北京时间每天 12:07。避开整点可显著降低 GitHub 定时任务延迟或丢触发的概率。
- **手动**：仓库 `Actions` 页 → 左侧 `SakuraFRP Daily Check-in` → `Run workflow`。

一次运行成功后，日志与截图（`logs/`、`captcha_bg.png`、`checkin.png` 等）会作为 artifact 上传，保留 7 天。

## 更换模型

1. 先确认目标 id 真实存在（无需鉴权即可查）：

   ```bash
   curl -s https://integrate.api.nvidia.com/v1/models | grep -o '"id":"[^"]*"'
   ```

2. 在仓库 `Variables` 里新建或修改上表中的变量，值填完整的 model id。
3. 下次运行自动采用。本地运行时则以 `.env` 为准（`.env` 里写了该键就覆盖代码默认值）。

换模型时的四条约束：

- id 必须与 `/v1/models` 返回的完全一致，格式为 `发布方/模型名`，点号与连字符原样保留（如 `z-ai/glm-5.3-flash`）。
- 视觉模型必须支持图像输入（OpenAI 风格的 `image_url` 内容片段），文本模型无此要求。
- 主模型与备用模型不要填成同一个，否则限流时没有兜底。
- 免费试用层按模型有每分钟与每月 token 上限，超限返回 429，会自动切备用模型。

## 本地运行

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
playwright install chromium

cp env.example .env      # 填入 NVIDIA_API_KEY
# account.txt 两行：第一行账号，第二行密码
python main.py --both
```

## 项目结构

```text
.
├── main.py                     # 主程序：登录、验证码、签到
├── ai_service.py               # NVIDIA NIM 调用封装（OpenAI 兼容）
├── logger.py                   # 日志记录器
├── requirements.txt            # Python 依赖
├── env.example                 # 环境变量模板
├── run_checkin.sh              # 本地/服务器执行脚本（含外层重试）
├── run_scheduled.sh            # 定时执行包装脚本
├── generate_random_time.sh     # 随机时刻生成脚本
├── test.py                     # 本地环境自检脚本
└── .github/workflows/checkin.yml
```

## 常见问题

**AI 调用被限流（429）**
主模型报 429 或 5xx 时会立刻切到备用模型，不再空转重试。若频繁触发，把 `NVIDIA_MODEL_VISION` 换成别的可用视觉模型，或调低 `AI_CELL_WORKERS`。

**验证码连续失败**
提高 `CHECKIN_MAX_RETRIES`（外层整轮重试）或 `CAPTCHA_EXTRA_ROUND_ATTEMPTS`（验证码额外轮次）；也可能是网络到目标站点不稳定，检查 `HTTP_PROXY`。

**收不到邮件**
本仓库默认 `EMAIL_NOTIFY_ENABLED=false`，这是有意为之。需要通知就把 workflow 里该行改成 `true`，并补 `EMAIL_SMTP_*` / `EMAIL_TO` 三个 Secrets。

**和上游同步冲突**
本仓库已修改 `ai_service.py`、`requirements.txt`、`checkin.yml`，`Sync fork` 会产生冲突，上游更新需要手动挑取。

## 许可与致谢

- 许可证沿用上游，见 [LICENSE](LICENSE)。
- 感谢 [XavierJiezou](https://github.com/XavierJiezou) 与 [RyanStarFox](https://github.com/RyanStarFox) 的原始实现。

## 免责声明

本项目仅用于个人学习与自动化实践。请遵守目标网站的服务条款与当地法律法规。