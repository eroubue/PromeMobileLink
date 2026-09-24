# 开发接入文档（0.2.2）

适用于 PR 插件和 ACR 作者。推荐通过 Dalamud IPC 注册设置、读取云端授权，无需引用 `PromeMobileLink.dll`，无需在消费方保存账号密码或自行请求云端。此仓库分发二进制和文档，不提供生产源码、构建凭据或部署配置。

## 1. 当前能力与边界

| 能力 | 0.2.1 状态 |
| --- | --- |
| 当前角色云端授权查询／变化通知 | 已有 IPC |
| QT 快捷开关手机控制 | 已有内置适配 |
| 第三方设置来源与通用表单 | 已有 IPC，提供方需主动接入 |
| toggle / slider / number / select / text / button | 手机已有渲染支持 |
| 插件安装、卸载、任意方法调用 | 未提供 |
| 统一插件控制中心、ACR 选择／生命周期控制 | 本版未提供 |
| 执行回执、模块独立授权、请求幂等 | 当前设置 IPC 不提供 |

```mermaid
sequenceDiagram
    participant App as Android
    participant Cloud as 云端
    participant Link as MobileLink
    participant Module as 插件／ACR
    Module->>Link: RegisterSettings
    App->>Cloud: 登录、PR 绑定、扫码确认
    Cloud-->>App: 短期 LAN ticket
    Link->>Cloud: 同步当前角色签名授权
    App->>Link: mDNS 发现后 POST /api/link
    Link-->>App: 短期 session
    App->>Link: POST /api/settings
    Link->>Module: SettingChanged.sourceId
    Module->>Link: 执行前查询 IsVerified
    Module->>Module: 框架线程应用设置
    Module->>Link: UpdateSetting 回传真实值
    Link-->>App: SSE settings
```

手机与电脑须在支持 mDNS 的同一局域网。`IsVerified == true` 不代表手机在线；手机云端验证成功也不代表 LAN 已连接。

## 2. 仅接入云端验证

完整可复制封装、状态 JSON、线程和到期语义见 [VERIFICATION_IPC.md](VERIFICATION_IPC.md)。最小判断：

```csharp
bool IsMobileLinkVerified(Dalamud.Plugin.IDalamudPluginInterface pi)
{
    try
    {
        var gate = pi.GetIpcSubscriber<bool>("PromeRotation.MobileLink.IsVerified");
        return gate.HasFunction && gate.InvokeFunc();
    }
    catch (System.Exception) { return false; }
}
```

生产代码缓存订阅对象，每次实际受保护操作重新读取结果。提供方未加载、卸载或异常均按未验证处理；持续运行的 ACR 在每次决策入口检查，角色登出或切换时停止旧角色逻辑。不要永久缓存一次成功，也不要用 `GetStatus.authorization.authorized` 原始摘要代替实时结果。

授权最多有效 120 秒；云端不可达不延长原证明。收到明确拒绝会失效，但尚未收到的撤销存在同步延迟，不能宣称所有离线客户端即时撤销。`VerificationChanged` 是无参数重读通知，可能不在框架线程；自然到期和卸载不能只依赖事件判断。

## 3. 设置 IPC 契约

以下名称统一加前缀 `PromeRotation.MobileLink.`，大小写敏感。`pi` 是宿主提供的 `IDalamudPluginInterface`。

| 名称 | 谁注册 | Dalamud 类型／调用 |
| --- | --- | --- |
| `IsVerified` | MobileLink | `GetIpcSubscriber<bool>` → `InvokeFunc()` |
| `GetStatus` / `GetSettings` | MobileLink | `GetIpcSubscriber<string>` → `InvokeFunc()` |
| `GetSetting` | MobileLink | `GetIpcSubscriber<string, string>` → `InvokeFunc(id)`，缺失返回 JSON `null` |
| `RegisterSettings` | MobileLink | `GetIpcSubscriber<string, object>` → `InvokeAction(json)` |
| `UnregisterSource` | MobileLink | 同上，参数为 `sourceId` |
| `UpdateSetting` | MobileLink | 同上，参数为 `{sourceId,id,value}` JSON |
| `ProvideSettings.<sourceId>` | 接入插件 | `GetIpcProvider<string>` → `RegisterFunc(Func<string>)` |
| `SettingChanged.<sourceId>` | 接入插件 | `GetIpcProvider<string, object>` → `RegisterAction(Action<string>)` |
| `VerificationChanged` | MobileLink | `GetIpcSubscriber<object>` → `Subscribe(Action)`／`Unsubscribe(Action)` |

另有 `GetChallengeInfo` / `GetPairingInfo`（无参 string 函数）、`RequestChallenge` / `RequestPairing`、`RevokeVerification`（无参 action）。它们操作共享授权状态，不应成为保存普通设置的隐式副作用。

注册和拉取均推荐完整对象：

```json
{
  "sourceId": "plugin.example",
  "sourceName": "示例插件",
  "settings": [{
    "id": "plugin.example.enabled", "group": "常规", "label": "启用功能",
    "description": "控制本插件公开的开关", "kind": "toggle", "value": false,
    "order": 10, "readonly": false, "confirm": true
  }]
}
```

| 字段 | 要求 |
| --- | --- |
| `sourceId` | 稳定命名空间，如 `plugin.<作者>.<名称>` / `acr.<作者>.<名称>`，勿占用内置 `qt` |
| `id` | **全局唯一**，使用完整来源前缀；手机按 id 保存、发送变更，不能只在来源内唯一 |
| `sourceName` / `group` / `label` / `description` | 展示名称、分组、标题、说明 |
| `kind` / `value` | 按下表匹配；类型变化需迁移或新 id |
| `min` / `max` / `step` | 数值范围与步长；提供方执行前仍检查业务约束 |
| `options` | select 的 `{value,label}` 字符串数组 |
| `order` | 同组排序，越小越靠前 |
| `readonly` | MobileLink 拒绝手机写入 |
| `confirm` | 手机操作前确认提示，不是权限或签名证明 |

| kind | 推荐 JSON value | 行为 |
| --- | --- | --- |
| `toggle` | `true` / `false` | 布尔开关 |
| `slider` / `number` | 有限数字 | 中间层做范围／步长处理，提供方复核业务范围 |
| `select` | `options[].value` 字符串 | 选择项 |
| `text` | 字符串 | 最长 512 字符，不用于密码或 PR 码 |
| `button` | `null` | 一次性动作，当前无可靠执行回执，不用于危险或不可重复操作 |

`SettingChanged.<sourceId>` 收到 `{"id":"plugin.example.enabled","value":true,"origin":"mobile"}`，**不含 sourceId、账号、CID、session 或 requestId**。来源由 IPC 名称确定；`origin` 不能用来认证。

## 4. 可复制的设置接入示例

示例公开一个布尔开关。调用方注入自己的配置读写、线程安全的角色快照和框架线程调度函数。角色快照是 `(Cid, Generation)`：CID 使用原始十进制字符串（不含前导零），不限制位数或数值上限，未登录返回 `null`；每次登出／登录／切角递增 Generation，即使回到同一 CID 也不能复用旧代次。`applyEnabled` 必须同步完成变更和必要保存，不能再排队而跳过第二次校验。

```csharp
using System;
using System.Text.Json;
using System.Threading;
using Dalamud.Plugin;
using Dalamud.Plugin.Ipc;

public sealed class ExampleMobileSettings : IDisposable
{
    private const string Prefix = "PromeRotation.MobileLink.";
    private const string Source = "plugin.example";
    private const string SettingId = Source + ".enabled";
    private readonly ICallGateSubscriber<bool> verified;
    private readonly ICallGateSubscriber<string> status;
    private readonly ICallGateSubscriber<string, object> register;
    private readonly ICallGateSubscriber<string, object> unregister;
    private readonly ICallGateSubscriber<string, object> update;
    private readonly ICallGateProvider<string> provide;
    private readonly ICallGateProvider<string, object> changed;
    private readonly Func<(string? Cid, long Generation)> readCharacter;
    private readonly Func<bool> readEnabled;
    private readonly Action<bool> applyEnabled;
    private readonly Action<Action> enqueueOnFramework;
    private string snapshot = "{}";
    private volatile bool disposed;

    public ExampleMobileSettings(IDalamudPluginInterface pi,
        Func<(string? Cid, long Generation)> readCharacter, Func<bool> readEnabled,
        Action<bool> applyEnabled, Action<Action> enqueueOnFramework)
    {
        this.readCharacter = readCharacter;
        this.readEnabled = readEnabled;
        this.applyEnabled = applyEnabled;
        this.enqueueOnFramework = enqueueOnFramework;
        verified = pi.GetIpcSubscriber<bool>(Prefix + "IsVerified");
        status = pi.GetIpcSubscriber<string>(Prefix + "GetStatus");
        register = pi.GetIpcSubscriber<string, object>(Prefix + "RegisterSettings");
        unregister = pi.GetIpcSubscriber<string, object>(Prefix + "UnregisterSource");
        update = pi.GetIpcSubscriber<string, object>(Prefix + "UpdateSetting");
        provide = pi.GetIpcProvider<string>(Prefix + "ProvideSettings." + Source);
        changed = pi.GetIpcProvider<string, object>(Prefix + "SettingChanged." + Source);
        RebuildSnapshot(); // 在消费方框架线程初始化。
        provide.RegisterFunc(() => Volatile.Read(ref snapshot));
        changed.RegisterAction(OnChanged);
    }

    public bool TryRegister() // 初始化及 MobileLink 重载后调用。
    {
        if (disposed) return false;
        try
        {
            if (!register.HasAction) return false;
            register.InvokeAction(Volatile.Read(ref snapshot));
            return true; // 仅代表调用完成；用 GetSettings 核实来源。
        }
        catch (Exception) { return false; }
    }

    private bool Allowed(string? expectedCid)
    {
        if (disposed || string.IsNullOrEmpty(expectedCid)) return false;
        try
        {
            if (!verified.HasFunction || !verified.InvokeFunc() || !status.HasFunction) return false;
            using var doc = JsonDocument.Parse(status.InvokeFunc());
            var root = doc.RootElement;
            return root.ValueKind == JsonValueKind.Object &&
                root.TryGetProperty("authorized", out var authorized) &&
                authorized.ValueKind == JsonValueKind.True &&
                root.TryGetProperty("authorization", out var authorization) &&
                authorization.ValueKind == JsonValueKind.Object &&
                authorization.TryGetProperty("cid", out var cid) &&
                cid.ValueKind == JsonValueKind.String && cid.GetString() == expectedCid;
        }
        catch (Exception) { return false; }
    }

    private void OnChanged(string json)
    {
        // 读取消费方维护的原子角色快照，不从 IPC 回调读取游戏指针。
        var identity = readCharacter();
        if (!Allowed(identity.Cid) || json.Length > 4096) return;
        try
        {
            using var doc = JsonDocument.Parse(json);
            var root = doc.RootElement;
            if (root.ValueKind != JsonValueKind.Object ||
                !root.TryGetProperty("id", out var id) ||
                id.ValueKind != JsonValueKind.String || id.GetString() != SettingId ||
                !root.TryGetProperty("value", out var value) ||
                value.ValueKind is not (JsonValueKind.True or JsonValueKind.False)) return;
            var enabled = value.GetBoolean(); // 不让 JsonElement 超出 Document 生命周期。
            enqueueOnFramework(() =>
            {
                if (disposed) return;
                try
                {
                    if (readCharacter() == identity && Allowed(identity.Cid)) applyEnabled(enabled);
                }
                finally { PublishCurrentValue(); }
            });
        }
        catch (JsonException) { /* 畸形输入不执行。 */ }
    }

    private void RebuildSnapshot()
    {
        Volatile.Write(ref snapshot, JsonSerializer.Serialize(new
        {
            sourceId = Source, sourceName = "示例插件",
            settings = new[] { new {
                id = SettingId, group = "常规", label = "启用功能",
                kind = "toggle", value = readEnabled(),
                order = 10, @readonly = false, confirm = true
            } }
        }));
    }

    public void PublishCurrentValue() // 框架线程；PC 本地改配置后也调用。
    {
        if (disposed) return;
        RebuildSnapshot();
        try
        {
            if (update.HasAction) update.InvokeAction(JsonSerializer.Serialize(new
            { sourceId = Source, id = SettingId, value = readEnabled() }));
        }
        catch (Exception) { /* MobileLink 重载后重新注册并刷新。 */ }
    }

    public void Dispose()
    {
        if (disposed) return;
        disposed = true;
        try { if (unregister.HasAction) unregister.InvokeAction(Source); }
        catch (Exception) { /* 提供方可能已卸载。 */ }
        finally
        {
            changed.UnregisterAction();
            provide.UnregisterFunc();
        }
    }
}
```

初始化后调用 `TryRegister()`；MobileLink 未加载时延迟重试，重载后再次注册。可以低频用 `GetSettings` 检查来源，只在缺失时重新注册，不要每帧全量注册。PC 修改配置后调用 `PublishCurrentValue()`；卸载时取消维护任务并 `Dispose()`。消费方记录自己的执行异常，不记录票据或凭据。

该示例入队和执行时检查顶层授权结果、授权 CID 与消费方实时 CID，并通过登录代次拒绝旧队列。MobileLink 的角色快照可能比消费方滞后，只有 `IsVerified=true` 不足以证明是当前 CID。当前回调不提供原 LAN session／请求上下文，不适用于要求原请求身份校验和可靠执行回执的操作。

## 5. 手机与 LAN 接口

App 经云端确认后，通过 `_prome-mobilelink._tcp.` 精确匹配实例，再用 ticket 建立 session。mDNS 名称和 info 展示信息不构成加密身份认证。

| HTTP 接口 | 作用 |
| --- | --- |
| `GET /api/info` | 无 session 的发现元信息 |
| `POST /api/link` | ES256 ticket 换 session，jti 单次消费 |
| `GET /api/status` | 当前连接状态 |
| `GET /api/settings` | 所有来源快照 |
| `POST /api/settings` | `{"updates":[{"id":"plugin.example.enabled","value":true}]}` |
| `POST /api/settings/refresh` | 拉取提供方真实设置 |
| `GET /api/events` | SSE：`hello` / `settings` / `verification` / `cloudAuthorization` |
| `POST /api/unlink` | 吊销当前连接；`{"all":true}` 吊销全部设备连接 |

除 info 和 link 外均需 `X-Prome-Session` 请求头。成功包络 `{"ok":true,"data":...}`，失败包络 `{"ok":false,"error":{"code":"...","message":"..."}}`。设置 POST 部分失败仍可返回 200，必须读取 `data.accepted`／`data.rejected`。每批最多 200 项，请求体不超过 64 KiB。

外部来源回调无返回值：**accepted 只表示 MobileLink 接受请求，不证明目标配置已保存**。回调缺失或内部失败也可能 accepted；刷新真实值用于校正界面，不是幂等机制。不可重复的 button 不自动重试。

session 只存内存，不跨插件重启恢复、不滑动延期；期限取 ticket 与授权的较早值，最多 120 秒，App 自动换票重连。LAN 使用 HTTP/SSE，仅用于可信局域网。

## 6. 云端版本与授权

| 请求头 | App | 插件 |
| --- | --- | --- |
| `X-MobileLink-Client` | `android` | `plugin` |
| `X-MobileLink-Version` | `0.2.1` | `0.2.1` |

公开 `GET /v1/version` 返回 `requiredVersion`、`updateMethod`、`updateUrl`。业务请求精确比较版本，缺失、重复、错类型、旧版或未发布新版被拒绝。HTTP 426 `client_update_required` 指明需更新客户端：App 为 `url`，插件为 `plugin_manager`。扫码确认也可能因电脑插件版本被拒绝，不能让用户反复更新手机。

签名为 ES256，插件仅内置公钥；校验用途、账号、实例、CID、授权 ID、撤销版本、nonce 和时间。CID 是原始十进制字符串，不含前导零，例如 `"18014449510679448"`；不得转成十六进制、JSON 数值或浮点数。仅校验格式，不限制位数或数值上限。

## 7. 接入验收

1. 未安装 MobileLink、调用异常、授权到期均不执行。
2. 切换角色、账号、实例或提供方重载，不复用旧授权与设置。
3. 手机修改→框架线程执行→PC 保存→刷新／SSE 显示真实值；PC 修改也同步。
4. readonly、非法值、未知 id 在提供方再次拒绝。
5. 排队期间授权失效／卸载／切角，旧请求不执行。
6. 提供方重载能重新注册；注销后来源消失，无事件泄漏。
7. 未接入具体 ACR 适配前，不宣称已支持该 ACR。

授权查询的完整封装见 [VERIFICATION_IPC.md](VERIFICATION_IPC.md)，版本变化见 [CHANGELOG.md](CHANGELOG.md)。
