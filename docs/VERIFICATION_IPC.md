# PR 插件 / ACR 云端验证 IPC 接入

其他 PR 插件或 ACR 可以调用 `PromeRotation.MobileLink.IsVerified`，判断当前游戏角色是否持有有效的 MobileLink 云端授权。接口由 PromeMobileLink 注册，通过 Dalamud IPC 调用；消费方只需使用宿主提供的 Dalamud 接口，无需引用 `PromeMobileLink.dll` 或自行请求云端 API。

## 1. 接口契约

以下名称均为完整 IPC 名称，区分大小写。

| IPC 名称 | Dalamud 订阅类型 | 调用方式 | 用途 |
| --- | --- | --- | --- |
| `PromeRotation.MobileLink.IsVerified` | `GetIpcSubscriber<bool>` | `InvokeFunc()` | 返回当前角色的有效云端授权结果，作为功能放行依据。 |
| `PromeRotation.MobileLink.GetStatus` | `GetIpcSubscriber<string>` | `InvokeFunc()` | 返回状态 JSON，用于界面提示和诊断。 |
| `PromeRotation.MobileLink.VerificationChanged` | `GetIpcSubscriber<object>` | `Subscribe(Action)` / `Unsubscribe(Action)` | 无参数状态变化通知；收到后重新查询授权结果。 |

`IsVerified` 和 `GetStatus` 都是同步读取插件内存中的状态，不会在调用时发起网络请求。调用函数前检查 `HasFunction`，并捕获调用异常，以覆盖 MobileLink 未加载、初始化未完成或正在卸载的情况。

## 2. `IsVerified` 的准确含义

返回 `true` 时，插件当前保存的云端授权同时满足：

1. `authorizationProof` 的 ES256 签名由内置 P-256 公钥验证，kid 在允许列表内。
2. 固定 issuer/audience/version/purpose 正确，随机 nonce 与本次请求匹配。
3. `0 < exp - iat <= 120`，UTC 和接收时固定的单调截止点都未到期；重复证明不延期。
4. 当前角色 CID 为非零 16 位大写 hex，签名账号、实例、CID、授权 ID 和撤销版本全部匹配。
5. 当前进程已取得该证明；未签名摘要或磁盘缓存不产生权限。

这是当前仍有效的授权结果，不是“历史上曾经扫过码”的永久标记。每次执行受保护操作、启动 ACR，或进入持续运行逻辑时，都应重新调用 `IsVerified()`；不要在首次成功后一直缓存 `true`。

| 场景 | 判断方式与当前行为 |
| --- | --- |
| 手机已完成云端确认，插件已同步有效授权 | `true`，不要求手机已建立 LAN 连接。 |
| 手机断开 Wi-Fi、mDNS 发现失败或 LAN session 失效 | 这些变化本身不改变云端授权结果；仍由授权、角色及有效期决定。 |
| 没有授权、授权到期或缺少有效的到期时间 | `false`。 |
| 退出角色或切换到其他 CID | 当前角色快照更新后，原角色授权不能用于新角色。 |
| 插件已收到云端吊销、解绑或其他明确的拒绝结果 | 原授权失效，返回 `false`。 |
| 云端暂时不可达，但存在当前角色尚未到期的有效签名证明 | 仍可返回 `true`，但不延长原证明期限，到期后返回 `false`；它不代表“此刻云端在线”。 |
| MobileLink 未加载、IPC 不可用或调用抛出异常 | 消费方按未验证处理；下方封装统一返回 `false`。 |

插件约每秒刷新一次当前角色快照，约每 2 秒调度一次云端同步；实例授权通常按 60 秒间隔刷新，进行中的 challenge 按约 2 秒间隔查询。网络请求耗时也会影响同步，因此 App 显示验证成功、云端撤销授权、角色切换与 IPC 结果更新之间可能存在延迟。对角色切换敏感的 ACR 还应使用宿主自身的登录/角色生命周期停止旧角色的运行逻辑。

重启插件后必须取得新的签名授权，旧磁盘缓存始终不能放行。`IsVerified` 不提供“每次调用都向服务器重新确认”的保证。

## 3. 可直接复制的 C# 封装

此封装缓存 IPC 订阅对象，不缓存授权结果。`TakeRefreshRequest()` 仅用于通知界面重新读取状态，回调中不直接操作游戏对象或 UI。

```csharp
using System;
using System.Threading;
using Dalamud.Plugin;
using Dalamud.Plugin.Ipc;

public sealed class MobileLinkCloudVerification : IDisposable
{
    private const string Prefix = "PromeRotation.MobileLink.";
    private readonly ICallGateSubscriber<bool> _isVerified;
    private readonly ICallGateSubscriber<string> _getStatus;
    private readonly ICallGateSubscriber<object> _changed;
    private int _refreshRequested = 1;
    private bool _disposed;

    public MobileLinkCloudVerification(IDalamudPluginInterface pluginInterface)
    {
        _isVerified = pluginInterface.GetIpcSubscriber<bool>(Prefix + "IsVerified");
        _getStatus = pluginInterface.GetIpcSubscriber<string>(Prefix + "GetStatus");
        _changed = pluginInterface.GetIpcSubscriber<object>(Prefix + "VerificationChanged");
        _changed.Subscribe(OnVerificationChanged);
    }

    public bool IsVerified()
    {
        if (_disposed) return false;

        try
        {
            return _isVerified.HasFunction && _isVerified.InvokeFunc();
        }
        catch (Exception)
        {
            // HasFunction 检查之后，提供方仍可能卸载或调用失败。
            return false;
        }
    }

    public string? GetStatusJson()
    {
        if (_disposed) return null;

        try
        {
            return _getStatus.HasFunction ? _getStatus.InvokeFunc() : null;
        }
        catch (Exception)
        {
            return null;
        }
    }

    public bool TakeRefreshRequest() =>
        Interlocked.Exchange(ref _refreshRequested, 0) != 0;

    private void OnVerificationChanged() =>
        Interlocked.Exchange(ref _refreshRequested, 1);

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _changed.Unsubscribe(OnVerificationChanged);
    }
}
```

在 PR 插件 / ACR 的初始化阶段创建一次，并保存为实例字段。使用 ECommons 的宿主可以传入 `Svc.PluginInterface`；独立 Dalamud 插件可以传入自己注入的 `IDalamudPluginInterface`。

```csharp
// 初始化时执行一次；verification 保存为消费方的实例字段。
verification = new MobileLinkCloudVerification(
    ECommons.DalamudServices.Svc.PluginInterface);
```

在需要云端验证的实际执行入口检查：

```csharp
if (verification is null || !verification.IsVerified())
{
    // 返回你自己的未验证结果，或跳过当前受保护的执行逻辑。
    return;
}

// 继续执行已经满足云端验证条件的逻辑。
```

以上是接入位置示意，不假定某个 PR / ACR SDK 的回调名称或返回类型；请按消费方实际入口调整 `return`。持续运行的 ACR 应在其后续执行入口继续检查，不能仅在启动时检查一次。

消费方卸载时调用 `verification?.Dispose()`，取消事件订阅。不要在每帧或每次判断时重复创建封装实例。

## 4. 状态变化通知

`VerificationChanged` 没有参数，也没有携带新的 `bool`。订阅使用 `Subscribe`，不要用 `InvokeAction()` 读取结果。调用 `Subscribe` 时无需等待提供方的 `HasAction` 变为 `true`；函数查询本身仍需检查 `HasFunction`。

当前实现会在云端状态更新、授权清理、角色变化处理等路径发出通知；即使 `IsVerified` 的值没有改变，也可能收到通知。通知可能来自异步云端任务，不保证运行在游戏框架线程。

上方封装只在回调中设置线程安全的刷新标记。消费方可以在自己的框架更新或 UI 流程中调用 `TakeRefreshRequest()`，为 `true` 时重新调用 `IsVerified()` 和 `GetStatusJson()`。首次创建封装时也会请求一次刷新。

授权随时间自然到期、提供方卸载时，不保证额外发出通知。界面可每秒刷新一次，并利用通知提前刷新；功能放行始终在执行入口重新查询，不能仅依赖事件维护的布尔缓存。

## 5. `GetStatus` JSON

返回值是直接的 JSON 对象字符串，没有云端 HTTP API 的 `{ "ok": ..., "data": ... }` 包络。字段采用 camelCase；值为 `null` 的可选字段会被省略。解析时允许字段缺失和新增字段。

以下是结构示例，时间和标识符均为示例值：

```json
{
  "state": "authorized",
  "authorized": true,
  "enabled": true,
  "cloudBaseUrl": "https://47.108.24.74",
  "pluginInstanceId": "pi_example",
  "challengeId": "qrc_example",
  "challengeStatus": "bound",
  "challengeExpiresAt": 1800000300,
  "lastCloudSyncAt": 1800000000,
  "authorization": {
    "state": "authorized",
    "authorized": true,
    "accountId": "acc_example",
    "prCodeId": "pr_example",
    "characterId": "char_example",
    "cid": "0000000000000001",
    "characterName": "角色名",
    "server": "区服名",
    "issuedAt": 1800000000,
    "expiresAt": 1800000120,
    "lastCloudSyncAt": 1800000000
  }
}
```

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `authorized` | boolean | 与 `IsVerified` 使用相同的计算逻辑，已经检查角色 CID 和授权有效期；两次调用之间状态仍可能变化。 |
| `state` | string | 状态展示文本，例如 `authorized`、`unknown`、`degraded`，也可能透传云端状态。不要把它当作固定枚举或独立的放行条件。 |
| `enabled` | boolean | 当前实现固定为 `true`，表示启用云端模式；不代表已经验证。 |
| `cloudBaseUrl` | string | 固定云端地址。 |
| `pluginInstanceId` | string，可省略 | 当前插件实例 ID。 |
| `challengeId` / `challengeStatus` | string，可省略 | 最近的扫码 challenge 及状态；`confirmed` / `bound` 本身不足以证明当前授权仍有效。 |
| `challengeExpiresAt` | integer，可省略 | 二维码 challenge 的到期时间，Unix 秒，与授权到期时间不同。 |
| `lastCloudSyncAt` | integer | 本次服务运行最近成功云端交互的 Unix 秒；当前实现重启后尚未同步时为 `0`，消费方仍应兼容字段缺失，不能证明云端此刻在线。 |
| `lastError` | string，可省略 | 最近错误信息，用于提示；没有错误时省略。存在网络错误也可能仍持有未到期授权。 |
| `authorization` | object，可省略 | 最近保存的云端授权摘要。 |
| `authorization.authorized` / `authorization.state` | boolean / string | 云端摘要的原始授权字段，没有重新计算当前角色及时间。使用顶层 `authorized` 或 `IsVerified` 作判断。 |
| `authorization.cid` | string，可省略 | 被授权角色的 CID，按字符串处理；本插件使用 16 位大写十六进制表示角色 CID。 |
| `authorization.accountId` / `prCodeId` / `characterId` | string，可省略 | 云端摘要中的账号、PR 码记录和角色记录 ID。 |
| `authorization.characterName` / `server` | string，可省略 | 云端摘要中的角色和区服名称。 |
| `authorization.issuedAt` / `expiresAt` / `lastCloudSyncAt` | integer，可省略 | 授权摘要的签发、到期及同步时间，均为 Unix 秒；C# 用 `long` / `long?` 接收。 |

当前实现查询状态时会检查有效期，发现授权到期后清理证明和摘要，顶层 `authorized` 返回 `false`。消费方先前读到或缓存的摘要不会自动更新，因此不能直接读取原始摘要字段绕过有效期检查；功能放行始终重新查询 `IsVerified`。

`GetStatusJson()` 返回 `null` 或 JSON 解析失败时，界面可显示“MobileLink 状态不可用”；功能是否放行仍使用前面的 `IsVerified()` 封装。

## 6. 更多接入资料

设置注册、手机控制与接入验收见 [DEVELOPMENT.md](DEVELOPMENT.md)，版本变化见 [CHANGELOG.md](CHANGELOG.md)。
