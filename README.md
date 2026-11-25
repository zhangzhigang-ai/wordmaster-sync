# WordMaster 云同步仓库

本仓库仅用于存储「单词小达人 / WordMaster」用户的学习进度同步文件。

## 一句话说明

APP 启动后按用户名算出自己的文件路径，只上传、只下载自己那一个文件。
仓库里没有账号密码表，也没有任何服务端代码——GitHub 本身就是存储。

## 目录结构

```
u/
  <hash16>.json        # 每个用户一个文件，hash16 = sha256(小写用户名)[:16]
```

- 用哈希做目录名，是为了让公开仓库里扫一眼看不出「谁是谁」。
- 同一个用户名在任何设备上算出的 hash 完全一致，所以换手机也能找回。

## 文件格式

```json
{
  "s": 1,
  "salt": "<16 字节随机 hex>",
  "hmac": "<对压缩载荷的 HMAC-SHA256>",
  "gz": "<base64(gzip(payload))>"
}
```

`payload` 解压后：

```json
{
  "user": "张三",
  "ageGroup": "junior",
  "rev": 12,
  "updatedAt": "2026-11-26T10:22:00Z",
  "deviceId": "<首次安装时生成的 UUID>",
  "deviceName": "Xiaomi 15",
  "coins": 24278,
  "achievements": ["streak7", "words500"],
  "mastered": ["apple", "banana"],
  "unknown": ["retroactive"],
  "stats": { "studiedDays": 42, "totalAnswers": 9860, "correctAnswers": 9102 }
}
```

## 身份校验

- 密码**不上传、不落地**，也不存哈希到仓库。
- 用 `PBKDF2(密码, salt, 100000)` 派生密钥，对 gz 载荷做 HMAC-SHA256，写进 `hmac`。
- 新设备输错密码时，本地算出的 HMAC 与文件里的不一致 → 直接提示「密码错误」。
- 因此即使有人拿到仓库也改不出一个「密码正确」的假文件。

## 写入规则

- 读取：走 `raw.githubusercontent.com`，无需凭证，天然只能读。
- 写入：走 GitHub Contents API，PUT 时带上文件的 `sha` 做乐观锁。
  - 409 冲突 → 重新拉取，按合并规则合并后再提交，最多重试 3 次。
- 每次提交 message 形如 `sync: <hash16> rev <n>`。
- Git 历史本身就是版本回滚的依据：误覆盖也能从历史里捞回来。

## 多设备合并规则

| 字段 | 规则 |
| --- | --- |
| coins | 取较大值（并保留双方原值到 ledgerHint） |
| mastered / unknown | 取并集 |
| achievements | 取并集 |
| rev | 取较大值 + 1 |
| stats | 各字段取较大值 |
| deviceId | 保留本次写入的设备 |
| ageGroup | 以本机注册时填写的为准 |

## 隐私边界

- 本仓库是公开的，任何人可以读到这些文件，但内容是压缩+哈希目录名，不含手机号、邮箱、设备序列号。
- 不收集精确年龄，只存学段（primary / junior / senior / adult）。
- 用户可在 APP 内「注销云端数据」，流程为把本文件覆写为 `{"s":1,"deleted":true}`；历史仍留在 Git 提交里，需要彻底清除请联系仓库所有者。

## 维护

- 不建议在此仓库放任何密钥、CI 配置或个人资料。
- 单文件建议控制在 1MB 以内（GitHub Contents API 上限），已背单词量极大时改分词条分片。
