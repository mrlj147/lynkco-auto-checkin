# 领克 App 自动签到

本仓库存放 Docker 部署配置，源码在 [lynkco-build](https://github.com/mrlj147/lynkco-build)。

## 功能

- 每日随机时间签到，使用原生签名和 `/up/api/v1/user/sign/upgrade`；已签到时跳过写操作。
- 查询任务前后的积分余额、累计积分和能量体，展示实际变化。
- 从探索广场最新文章流随机选文章，优先避免分享历史中已用过的文章；没有可用文章时跳过，不使用固定旧文章兜底。
- 分享优先采用 getShareCode → shareReporting 两步法，失败后回退 lookup → check → reporting 三步流程。
- Bark 推送签到结果、分享文章标题、任务进度和积分前后对比；保留企业微信、钉钉、飞书通知。
- Token 未过期时复用缓存，过期前自动续期；配置 APPCODE 时优先使用它，失败后回退 AppSecret 原生 HMAC。
- 保留 DeepSeek 自动评论的提示词、选帖、去重、发表请求和评论通知。评论继续使用旧 H5 签名器，新签到使用独立原生签名器。

分享接口返回成功不代表当天一定加分，通知以实际积分和能量体变化为准。完整回退流程也受服务端的内容检查、每日次数和账号规则影响。

## 从旧版本升级

1. 保留原来的 `.env`、`lynkco-data` 数据卷、Bark 和 DeepSeek 配置。
2. 补充 `LYNKCO_DEVICE_ID`、`LYNKCO_NATIVE_APP_KEY`、`LYNKCO_NATIVE_APP_SECRET`。
3. 原来的 `LYNKCO_CA_KEY` / `LYNKCO_CA_SECRET` 属于评论 H5 签名，**不要用新的原生密钥覆盖它们**。
4. 更新镜像并重建容器：

```bash
docker compose pull
docker compose up -d
docker logs -f lynkco-checkin
```

新源码合并到构建仓库 main 并成功发布镜像后，拉取 latest 才能得到更新。

## 部署

```bash
git clone https://github.com/mrlj147/lynkco-auto-checkin.git
cd lynkco-auto-checkin
cp .env.example .env
# 填写个人凭据、应用密钥与通知配置
docker compose up -d
```

也可以直接运行：

```bash
docker run -d --name lynkco-checkin --restart=unless-stopped \
  --env-file .env -e TZ=Asia/Shanghai \
  -v lynkco-data:/data mrlj147/lynkco-auto-checkin:latest
```

模拟器只用于获取应用密钥，日常容器不需要安卓环境。

## 获取个人登录凭据

iPhone 可用 Stream 抓包，筛选 `app-services.lynkco.com.cn`：

| 抓包字段 | 环境变量 |
|---|---|
| 登录响应 `data.centerTokenDto.token` | `LYNKCO_CENTER_TOKEN` |
| 登录响应 `data.centerTokenDto.refreshToken` | `LYNKCO_REFRESH_TOKEN` |
| 同次登录或续期请求参数 `deviceId` | `LYNKCO_DEVICE_ID` |

deviceId 必须与 refreshToken 对应。应用密钥与个人登录凭据是不同的配置；原生 AppKey/AppSecret 的提取方法参考 [LynkCoHelper](https://github.com/shovelshit/LynkCoHelper)。

## 环境变量

| 变量 | 用途 |
|---|---|
| `LYNKCO_CENTER_TOKEN` | 当前登录 Token；配置续期凭据后可自动获取 |
| `LYNKCO_REFRESH_TOKEN` | 续期凭据，建议配置 |
| `LYNKCO_DEVICE_ID` | 同次登录的设备 ID，续期必填 |
| `LYNKCO_NATIVE_APP_KEY` / `LYNKCO_NATIVE_APP_SECRET` | 新签到、分享、积分查询的原生应用密钥 |
| `LYNKCO_NATIVE_APP_CODE` | 可选 APPCODE 续期方案；没有也能使用 HMAC |
| `LYNKCO_NATIVE_GL_DEV_ID` | 可选设备请求头，默认使用 deviceId |
| `LYNKCO_APP_SECRETS` | 可选整合 JSON；字段 nativeAppKey/nativeAppSecret/nativeAppCode/glDevId，独立字段优先 |
| `LYNKCO_DEVICE_HEADERS` | 可选设备请求头 JSON，支持使用抓包中的设备字段 |
| `LYNKCO_APP_VERSION` / `LYNKCO_APP_BUILD` | 可选 App 版本、build 字段 |
| `LYNKCO_TOKEN_CACHE_PATH` | 默认建议 `/data/token_cache.json` |
| `LYNKCO_BARK_KEY` / `LYNKCO_BARK_SERVER` | Bark 推送配置，保留旧值 |
| `LYNKCO_ENERGY_DELAY` | 任务后等积分更新的秒数，默认 5 |
| `LYNKCO_CHECKIN_START` / `LYNKCO_CHECKIN_END` | 每日签到窗口，默认北京时间 1–6 点 |
| `LYNKCO_COMMENT_ENABLED` | 自动评论开关，默认 false |
| `LYNKCO_COMMENT_LIMIT` | 每日目标评论条数，默认 3 |
| `DEEPSEEK_API_KEY` / `DEEPSEEK_MODEL` / `DEEPSEEK_BASE_URL` | 保留现有自动评论模型配置 |
| `LYNKCO_WECOM_WEBHOOK` / `LYNKCO_DINGTALK_WEBHOOK` / `LYNKCO_FEISHU_WEBHOOK` | 保留原有多渠道通知配置 |

## 自动评论

启用评论后，容器仍先执行签到、分享、积分查询和每日通知，等待 5 分钟后运行评论。单次评论命令仍可用：

```bash
docker exec lynkco-checkin python main.pyc --comment --comment-limit 3
```

本次修复了旧入口同时传 `--schedule --comment` 会提前进入单次评论、跳过定时签到的问题。评论正文生成、成功间隔、失败重试和历史去重逻辑保持原样。

## Token 与数据持久化

挂载 `/data` 保留 Token 缓存、分享历史和评论历史。续期成功后保存新的 token、refreshToken 及服务端过期时间，缓存文件只允许文件所有者读写。

refreshToken 可能过期或被服务端撤销，不保证永久有效。重新登录后请更新个人凭据；如仍使用旧缓存，可备份后删除 `/data/token_cache.json` 再重启。不要删除整个数据卷，以免丢失评论历史。

## 通知示例

```text
领克App 自动签到
今日已签到
分享接口成功（奖励以积分变化为准）
分享文章: 周末出行分享（虚构示例）
分享流程: 简化两步
积分余额: 0 → 5（+5）
能量体: 100 → 102（+2）
累计积分: 2005
```

## 常见操作

```bash
# 立即执行一轮签到、分享、积分对比与通知
docker exec lynkco-checkin python main.pyc --once
# 查看签到任务
docker exec lynkco-checkin python main.pyc --info
# 单独执行分享
docker exec lynkco-checkin python main.pyc --share
```

上述手动命令不会额外运行自动评论。Bark 未配置时仅输出日志，签到和分享仍可运行。

## 致谢

原生签名与分享协议参考 [shovelshit/LynkCoHelper](https://github.com/shovelshit/LynkCoHelper)，MIT 许可说明随构建镜像提供。保留原项目对 [四十六](https://github.com/suyunkai) 的致谢。
