# HS TikTok MiniGame SDK — Integration Guide

## Requirements

- Unity 2020+
- TikTok MiniGame Unity Plugin (with `TTSDK_MIX_ENGINE` scripting define)

## Installation

**Package Manager → + → Add package from tarball...**
Select the `.tgz` file provided by the SDK distributor.

---

## 1. Setup the SDK in your Scene

**Menu:** `HS TikTok SDK > Setup SDK in Scene`

This creates a persistent `HS_TikTokSDK` GameObject with all required components:
- `TikTokSDK` — Core config & session
- `TikTokAuth` — Login
- `TikTokIAP` — In-App Purchases
- `TikTokSync` — Cloud Save (optional)

## 2. Configure in Inspector

Select the `HS_TikTokSDK` GameObject and set:

| Field | Description |
|-------|-------------|
| **Game ID** | Your TikTok App ID (numeric, from TikTok Developer Portal) |
| **SDK Key** | Your SDK key (starts with `sk_live_` or `sk_test_`, from HS Developer Portal) |
| **Enable Debug Logs** | Toggle SDK console logging (disable for production) |

> **Note:** Test mode is auto-detected at runtime. Unity Editor and non-WebGL platforms run in test mode. WebGL TikTok builds run in production mode automatically.

---

## 3. Login

```csharp
using HS.TikTokSDK.Auth;

// Subscribe to auth events
TikTokAuth.Instance.OnLoginSuccess += (string profileJson, long serverTimestamp) =>
{
    Debug.Log("Login success!");
    // profileJson contains the player's cloud save data (if any)
    // serverTimestamp is the server-side last update time
};

TikTokAuth.Instance.OnLoginFailed += (string error) =>
{
    Debug.LogError($"Login failed: {error}");
};

// Get auth code from TikTok, then exchange via SDK
TT.Login(
    successCallback: (code) => TikTokAuth.Instance.LoginWithCode(code),
    failedCallback: (err) => Debug.LogError($"TT.Login failed: {err}")
);
```

After a successful login, `TikTokSDK.Instance.IsAuthenticated` returns `true` and you can access:
- `TikTokSDK.Instance.SessionToken`
- `TikTokSDK.Instance.OpenId`

---

## 4. In-App Purchases

### Buy a Product

```csharp
using HS.TikTokSDK.IAP;

TikTokIAP.Instance.BuyProduct("1"); // product ID from your backend
```

### Product Config

Products are configured on the backend. The SDK only needs the **product ID** to initiate a purchase. The backend handles pricing and reward delivery.

Each product on the backend requires:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique product ID (passed to `BuyProduct()`) |
| `beans_cost` | int | Price in TikTok Beans |
| `rewards` | array | List of rewards returned after payment confirmation |

Example backend product config:

```json
[
  { "id": "1", "beans_cost": 10, "rewards": [{ "type": "hints", "amount": 3 }] },
  { "id": "2", "beans_cost": 50, "rewards": [{ "type": "no_ads", "amount": 1 }] }
]
```

### TikTokReward Model

The SDK returns `TikTokReward` objects after a successful purchase:

```csharp
public class TikTokReward
{
    public string type;   // "hints", "no_ads", "gold", etc.
    public int amount;     // For countable rewards (3 hints, 100 gold)
    public bool value;     // For boolean flags (ads_removed = true)
}
```

| Reward Type | `amount` | `value` | Use Case |
|-------------|----------|---------|----------|
| `"hints"` | Count to add | — | `player.hints += reward.amount` |
| `"no_ads"` | — | `true` | `player.adsRemoved = true` |
| `"gold"` | Count to add | — | `player.gold += reward.amount` |

### Handle Purchase Events

```csharp
using HS.TikTokSDK;
using HS.TikTokSDK.IAP;

// Purchase confirmed
TikTokIAP.Instance.OnPurchaseSuccess += (string productId, List<TikTokReward> rewards) =>
{
    foreach (var reward in rewards)
    {
        Debug.Log($"Reward: {reward.type} x{reward.amount}");
        // Grant rewards to player
    }
};

// Purchase failed or canceled
TikTokIAP.Instance.OnPurchaseFailed += (string error) =>
{
    Debug.LogWarning($"Purchase failed: {error}");
};

// Crash recovery — undelivered rewards from previous sessions
TikTokIAP.Instance.OnPendingRewardRecovered += (string productId, List<TikTokReward> rewards) =>
{
    // Handle the same as OnPurchaseSuccess
};
```

### Using ITikTokRewardHandler (Recommended)

Instead of subscribing to events, implement the interface for cleaner code:

```csharp
using HS.TikTokSDK;
using System.Collections.Generic;

public class MyShopController : MonoBehaviour, ITikTokRewardHandler
{
    void Start()
    {
        TikTokSDK.Instance.RegisterRewardHandler(this);
    }

    public void OnPurchaseRewardsGranted(string productId, List<TikTokReward> rewards)
    {
        foreach (var r in rewards)
        {
            if (r.type == "hints") AddHints(r.amount);
            if (r.type == "ads_removed") RemoveAds();
        }
    }

    public void OnPendingRewardsRecovered(string productId, List<TikTokReward> rewards)
    {
        // Same handling as a normal purchase
        OnPurchaseRewardsGranted(productId, rewards);
    }
}
```

---

## 5. Cloud Sync (Optional)

Implement `ITikTokSyncProvider` to enable automatic cloud saves:

```csharp
using HS.TikTokSDK;

public class MySaveManager : MonoBehaviour, ITikTokSyncProvider
{
    private MyPlayerData _data;

    public bool IsDirtyForCloud { get; set; }

    public string SerializeForCloud()
    {
        // Return your save data as JSON
        return JsonUtility.ToJson(_data);
    }

    public void OnCloudSyncSuccess(long serverTimestamp)
    {
        // Save the server timestamp locally (do NOT set IsDirtyForCloud here)
        _data.lastUpdated = serverTimestamp;
        SaveLocally();
    }

    public void OnCloudDataReceived(string profileJson, long serverTimestamp)
    {
        // Server has newer data — overwrite local save
        _data = JsonUtility.FromJson<MyPlayerData>(profileJson);
        _data.lastUpdated = serverTimestamp;
        SaveLocally();
    }

    void Start()
    {
        TikTokSDK.Instance.RegisterSyncProvider(this);
    }

    // Call this whenever player data changes
    public void OnDataChanged()
    {
        IsDirtyForCloud = true;
        SaveLocally();
    }
}
```

### How Sync Works

- The SDK **auto-syncs** every 10 seconds when `IsDirtyForCloud` is `true`
- You can trigger a **manual sync** anytime:

```csharp
using HS.TikTokSDK.Sync;

TikTokSync.Instance.SyncNow(success =>
{
    Debug.Log(success ? "Synced!" : "Sync skipped or failed");
});
```

---

## Architecture

```
Your Game Code                     HS TikTok SDK
┌──────────────────┐              ┌──────────────────────────┐
│ MySaveManager    │──implements──▶ ITikTokSyncProvider      │
│ MyShopController │──implements──▶ ITikTokRewardHandler     │
└──────────────────┘              │                          │
                                  │ TikTokSDK    (config)    │
         registers ──────────────▶│ TikTokAuth   (login)     │
                                  │ TikTokIAP    (purchases) │
                                  │ TikTokSync   (cloud)     │
                                  └──────────────────────────┘
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `SDK Key and Game ID appear to be SWAPPED` | Check Inspector — SDK Key starts with `sk_`, Game ID is numeric |
| `Cannot buy — player is not logged in` | Call `TikTokAuth.Instance.LoginWithCode()` before purchasing |
| `Already processing a purchase` | Wait for current purchase to complete (double-tap guard) |
| Purchases not working in Editor | Editor uses mock payments — test real payments on TikTok device |
| Sync not running | Ensure `RegisterSyncProvider()` is called and `IsDirtyForCloud = true` |
