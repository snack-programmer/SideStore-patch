# SideSign 认证与 anisette 身份说明

本文合并并取代 `WINDOWS_AUTH_FIX.md` 与旧的 `SIDEINSTALLER_AUTH.md`，描述当前代码
（SideSign `cc14e41`、SideStore「adi.pb 只存 Keychain」之后）的登录、2FA 与机器身份模型。

## 1. 身份模型

一次登录只使用一套 `client-info` / `User-Agent`：

| 层 | 责任 |
|---|---|
| `SideStore/AppConstants.swift` | 唯一定义 `macClientInfo` / `macUserAgent` |
| `AnisetteProvider` | 根据服务器类型选择身份并写入 `AnisetteData` |
| `SideSign` | 只消费调用方传入的 `AnisetteData`，不再内置设备字面量 |

机器身份（`adi.pb`）只存 Keychain：

- `AnisetteConfigManager.anisetteAdiBlob` 是唯一持久化入口；
- `SideSign` 通过注入的 `AnisetteIdentityStorage` 读写，不碰宿主 UserDefaults；
- 旧版本写入的 `UserDefaults["SideSign.anisetteAdiBlob"]` 会在启动时删除。

> 结论：不要再引入第二处 `MacBookPro…` 字面量；如需换机型，只改 `AppConstants`。

## 2. 登录流程

1. `Authentication: init` → `complete`（SRP）。
2. 若 Apple 要求 2FA：按 `au` 字段进入设备码或手机码流程（见 §3）。
3. 2FA 成功后，用**同一身份**重新登录：
   - 先刷新一次 OTP（同一 `adi.pb`，只换 `X-Apple-I-MD`）；
   - 最多 3 次尝试，退避 1s / 3s / 5s；
   - 整体 60s deadline，超时返回可读错误。

最多允许两轮 2FA（首轮 + 重登后 Apple 再要求一次），第三轮报
`tooManyAttempts`，避免无限循环。

## 3. 2FA 渠道

| 状态 | 行为 |
|---|---|
| `needsDevice2FA` | GET 设备码；Apple 不确认也可继续输入 |
| `needsDevice2FAVerification` | 提示设备码并校验 |
| `needsPhone2FA(id, mode)` | 按 `sms` / `voice` 发码 |
| `needsPhone2FAVerification(id, mode)` | **用发码时的同一 `mode`** 提示并校验 |

约束：

- 发码与验证各最多 5 次，超限抛 `tooManyAttempts`；
- 电话被拒（401/403/423、XMLUI 告警、iCloud 文案）自动回退设备码；
- 5xx / 网关 / 限流保留真实错误，不伪装成“电话被拒”；
- 手机号 id 必须为数字；非数字时按号码列表索引映射，失败即报错。

## 4. 服务器选择

| 设置 | 行为 |
|---|---|
| Auto 开 + 允许轮换 | 使用候选列表（官方优先），按最近失败记录跳过 |
| Auto 开 + 关闭轮换 | 只用当前选中的服务器 |
| Auto 关 | 只用当前选中的服务器 |

默认服务器为官方 `https://new.ani.sidestore.io`。第三方 ADI 服务器不再自动加入
候选列表，需要用户在“Anisette Servers”里显式添加；添加页会提示第三方可以看到
设备身份。

## 5. 身份恢复

只有在明确的身份类错误上才清空 `adi.pb` 并重新 provision：

- `-45061`、`-80045`、`-80046`
- `invalid anisette`、`not provisioned`

限流（429）、服务端 5xx、账号问题（密码错误等）**不会**清空身份，避免反复换
身份加重 Apple 风控。

启动流程不再清空 `adi.pb`；若旧身份损坏，首次登录失败后由上述错误触发重建。

## 6. 验证

```sh
# SideSign
cd SideSign-patch
swift build
swift test          # 5 tests

# SideStore
plutil -lint AltStore/Info.plist
swiftc -frontend -parse SideStore/Core/Anisette/AnisetteProvider.swift
```

排障日志关键字（`debugLog`）：

- `Post-2FA re-login attempt`
- `phone 2FA (sms|voice) refused`
- `Identity error — purging adi.pb`
- `Auto=… rotation=…`

## 7. 已知限制

- On-Device Anisette 在新流程下不可用，`AnisetteProvider.fetch` 会保持关闭；
  设置界面的开关目前仍可见，后续应隐藏或标注为不可用。
- 第三方 anisette 服务器需要手动添加，Auto 不再自动覆盖。
- 2FA 重登总时长受 60s deadline 约束；单次请求超时可能使实际耗时略高。
