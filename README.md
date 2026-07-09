# 领克App 自动签到

每日自动签到领克 App + 自动分享文章领积分，支持 Bark 推送通知。

## 功能

- 每日自动签到（凌晨 1-6 点随机时间）
- 自动刷新 Token（每次签到自动续期，永不过期）
- 自动分享文章领积分（每天随机获取新文章，不重复）
- Bark 推送通知（签到结果、积分余额、能量体、任务进度）

## 快速开始

### 方式一：一行命令部署

```bash
docker run -d \
  --name lynkco-checkin \
  --restart=unless-stopped \
  -e TZ=Asia/Shanghai \
  -e LYNKCO_CENTER_TOKEN=bearer你的token \
  -e LYNKCO_REFRESH_TOKEN=bearer你的refreshToken \
  -e LYNKCO_BARK_KEY=你的BarkKey \
  -e LYNKCO_TOKEN_CACHE_PATH=/data/token_cache.json \
  -v lynkco-data:/data \
  mrlj147/lynkco-auto-checkin:latest
```

### 方式二：Docker Compose 部署

```bash
git clone https://github.com/mrlj147/lynkco-auto-checkin.git
cd lynkco-auto-checkin
cp .env.example .env
vi .env
docker compose up -d
```

## 抓取 Token（iPhone）

1. 下载 [Stream](https://apps.apple.com/app/stream-network-debug-tool/id1312141691)
2. 开始抓包 → 打开领克 App 退出重新登录 → 停止抓包
3. 筛选域名 `app-services.lynkco.com.cn`，寻找 `login` → 从响应里复制 `token` 和 `refreshToken`

登录成功后返回类似结构：

```json
{
  "code": "success",
  "data": {
    "centerTokenDto": {
      "token": "bearer****-****-****-****-************",
      "refreshToken": "bearer****-****-****-****-************",
      "expireAt": 1783000000000,
      "refreshExpireAt": 1786000000000
    }
  }
}
```

填写 `.env` 时：
- `centerTokenDto.token` → `LYNKCO_CENTER_TOKEN`
- `centerTokenDto.refreshToken` → `LYNKCO_REFRESH_TOKEN`

## Token 自动续期

每次签到时会自动刷新 Token，服务端返回新的 `centerToken` 和 `refreshToken`，保存到 `token_cache.json`。

只要容器每天运行一次签到，Token 就永远不会过期，无需手动更新。

**验证 Token 续期：**

```bash
docker exec lynkco-checkin cat /data/token_cache.json
```

如果 `center_token` 和 `.env` 里的不同，说明续期正常工作。

## 自动分享文章

每次签到后会自动执行分享任务。

## 环境变量

| 变量 | 必填 | 说明 |
|---|---|---|
| `LYNKCO_CENTER_TOKEN` | 是 | 登录态 Token（30 分钟过期，自动刷新） |
| `LYNKCO_REFRESH_TOKEN` | 是 | 刷新 Token（每次签到自动续期） |
| `LYNKCO_BARK_KEY` | 是 | Bark 推送 Key |
| `LYNKCO_TOKEN_CACHE_PATH` | 是 | Token 缓存路径，必须设置为 `/data/token_cache.json` |
| `LYNKCO_CHECKIN_START` | 否 | 签到开始小时（默认 1，即凌晨 1 点） |
| `LYNKCO_CHECKIN_END` | 否 | 签到结束小时（默认 6，即凌晨 6 点） |

签到时间示例：
- `1` / `6` = 凌晨 1:00-6:00 随机签到
- `13` / `18` = 下午 13:00-18:00 随机签到

## 注意事项

- `LYNKCO_TOKEN_CACHE_PATH` 必须设置为 `/data/token_cache.json`，否则容器重建后 Token 丢失
- volume 必须挂载到 `/data`，保证 Token 缓存和分享历史持久化
- 不要在手机上退出领克 App 登录，否则 refreshToken 会立即失效
- `restart: unless-stopped` 保证容器自动重启，每天自动签到续期

## Bark 通知示例

```
领克App 自动签到
签到成功
Co积分: +1
能量体: +5
连续签到: 2 天
分享文章: 1881101031748870144
补签卡: 0 张
签到任务:
  连续签到7天: 2/7 (1能量体)
  本月度签到25天: 3/25 (1补签卡)
  本季度签到85天: 3/85 (2补签卡)
  连续签到365天: 2/365 (20能量体, 365Co积分)
积分余额: ****
累计积分: ****
能量体: ***
token 有效期截至到:2026-**-** **:**:**
```

## 常见问题

**Q: 会不会挤掉手机上的账号？**
A: 不会。Token 和手机登录会话是独立的。

**Q: refreshToken 会过期吗？**
A: 不会。每次签到都会自动刷新，新 Token 会保存到 `token_cache.json`。只要每天运行一次，就永远不会过期。

**Q: 怎么查看日志？**
```bash
docker logs -f lynkco-checkin
```

**Q: 怎么更新 Token？**
A: 正常情况下不需要手动更新。如果长期停止运行超过 30 天，才需要重新抓包获取新的 refreshToken。

## 致谢

感谢 [四十六](https://github.com/suyunkai) 大佬提供的 token 续期思路。

## 免责声明

仅供学习交流，请勿滥用。