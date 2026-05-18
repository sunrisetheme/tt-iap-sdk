HS TikTok MiniGame SDK — Integration Guide

Requirements
Unity 2020+
TikTok MiniGame Unity Plugin (with TTSDK_MIX_ENGINE scripting define)
Installation
Package Manager → + → Add package from tarball... Select the .tgz file provided by the SDK distributor.

1. Setup the SDK in your Scene
Menu: HS TikTok SDK > Setup SDK in Scene

This creates a persistent HS_TikTokSDK GameObject with all required components:

TikTokSDK — Core config & session
TikTokAuth — Login
TikTokIAP — In-App Purchases
TikTokSync — Cloud Save (optional)
2. Configure in Inspector
Select the HS_TikTokSDK GameObject and set:

Field	Description
Game ID	Your TikTok App ID (numeric, from TikTok Developer Portal)
SDK Key	Your SDK key (starts with sk_live_ or sk_test_, from HS Developer Portal)
Enable Debug Logs	Toggle SDK console logging (disable for production)
Note: Test mode is auto-detected at runtime. Unity Editor and non-WebGL platforms run in test mode. WebGL TikTok builds run in production mode automatically.
3. Login
1using HS.TikTokSDK.Auth;
2
3// Subscribe to auth events
4TikTokAuth.Instance.OnLoginSuccess += (string profileJson, long serverTimestamp) =>
5{
6    Debug.Log("Login success!");
7    // profileJson contains the player's cloud save data (if any)
8    // serverTimestamp is the server-side last update time
9};
10
11TikTokAuth.Instance.OnLoginFailed += (string error) =>
12{
13    Debug.LogError($"Login failed: {error}");
14};
15
16// Get auth code from TikTok, then exchange via SDK
17TT.Login(
18    successCallback: (code) => TikTokAuth.Instance.LoginWithCode(code),
19    failedCallback: (err) => Debug.LogError($"TT.Login failed: {err}")
20);
After a successful login, TikTokSDK.Instance.IsAuthenticated returns true and you can access:

TikTokSDK.Instance.SessionToken
TikTokSDK.Instance.OpenId
4. In-App Purchases
Buy a Product
1using HS.TikTokSDK.IAP;
2
3TikTokIAP.Instance.BuyProduct("1"); // product ID from your backend
Product Config
Products are configured on the backend. The SDK only needs the product ID to initiate a purchase. The backend handles pricing and reward delivery.

Each product on the backend requires:

Field	Type	Description
id	string	Unique product ID (passed to BuyProduct())
beans_cost	int	Price in TikTok Beans
rewards	array	List of rewards returned after payment confirmation
Example backend product config:

1[
2  { "id": "1", "beans_cost": 10, "rewards": [{ "type": "hints", "amount": 3 }] },
3  { "id": "2", "beans_cost": 50, "rewards": [{ "type": "no_ads", "amount": 1 }] }
4]
TikTokReward Model
The SDK returns TikTokReward objects after a successful purchase:

1public class TikTokReward
2{
3    public string type;   // "hints", "no_ads", "gold", etc.
4    public int amount;     // For countable rewards (3 hints, 100 gold)
5    public bool value;     // For boolean flags (ads_removed = true)
6}
Reward Type	amount	value	Use Case
"hints"	Count to add	—	player.hints += reward.amount
"no_ads"	—	true	player.adsRemoved = true
"gold"	Count to add	—	player.gold += reward.amount
Handle Purchase Events
1using HS.TikTokSDK;
2using HS.TikTokSDK.IAP;
3
4// Purchase confirmed
5TikTokIAP.Instance.OnPurchaseSuccess += (string productId, List<TikTokReward> rewards) =>
6{
7    foreach (var reward in rewards)
8    {
9        Debug.Log($"Reward: {reward.type} x{reward.amount}");
10        // Grant rewards to player
11    }
12};
13
14// Purchase failed or canceled
15TikTokIAP.Instance.OnPurchaseFailed += (string error) =>
16{
17    Debug.LogWarning($"Purchase failed: {error}");
18};
19
20// Crash recovery — undelivered rewards from previous sessions
21TikTokIAP.Instance.OnPendingRewardRecovered += (string productId, List<TikTokReward> rewards) =>
22{
23    // Handle the same as OnPurchaseSuccess
24};
Using ITikTokRewardHandler (Recommended)
Instead of subscribing to events, implement the interface for cleaner code:

1using HS.TikTokSDK;
2using System.Collections.Generic;
3
4public class MyShopController : MonoBehaviour, ITikTokRewardHandler
5{
6    void Start()
7    {
8        TikTokSDK.Instance.RegisterRewardHandler(this);
9    }
10
11    public void OnPurchaseRewardsGranted(string productId, List<TikTokReward> rewards)
12    {
13        foreach (var r in rewards)
14        {
15            if (r.type == "hints") AddHints(r.amount);
16            if (r.type == "ads_removed") RemoveAds();
17        }
18    }
19
20    public void OnPendingRewardsRecovered(string productId, List<TikTokReward> rewards)
21    {
22        // Same handling as a normal purchase
23        OnPurchaseRewardsGranted(productId, rewards);
24    }
25}
5. Cloud Sync (Optional)
Implement ITikTokSyncProvider to enable automatic cloud saves:

1using HS.TikTokSDK;
2
3public class MySaveManager : MonoBehaviour, ITikTokSyncProvider
4{
5    private MyPlayerData _data;
6
7    public bool IsDirtyForCloud { get; set; }
8
9    public string SerializeForCloud()
10    {
11        // Return your save data as JSON
12        return JsonUtility.ToJson(_data);
13    }
14
15    public void OnCloudSyncSuccess(long serverTimestamp)
16    {
17        // Save the server timestamp locally (do NOT set IsDirtyForCloud here)
18        _data.lastUpdated = serverTimestamp;
19        SaveLocally();
20    }
21
22    public void OnCloudDataReceived(string profileJson, long serverTimestamp)
23    {
24        // Server has newer data — overwrite local save
25        _data = JsonUtility.FromJson<MyPlayerData>(profileJson);
26        _data.lastUpdated = serverTimestamp;
27        SaveLocally();
28    }
29
30    void Start()
31    {
32        TikTokSDK.Instance.RegisterSyncProvider(this);
33    }
34
35    // Call this whenever player data changes
36    public void OnDataChanged()
37    {
38        IsDirtyForCloud = true;
39        SaveLocally();
40    }
41}
How Sync Works
The SDK auto-syncs every 10 seconds when IsDirtyForCloud is true
You can trigger a manual sync anytime:
1using HS.TikTokSDK.Sync;
2
3TikTokSync.Instance.SyncNow(success =>
4{
5    Debug.Log(success ? "Synced!" : "Sync skipped or failed");
6});
Architecture
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
Troubleshooting
Issue	Solution
SDK Key and Game ID appear to be SWAPPED	Check Inspector — SDK Key starts with sk_, Game ID is numeric
Cannot buy — player is not logged in	Call TikTokAuth.Instance.LoginWithCode() before purchasing
Already processing a purchase	Wait for current purchase to complete (double-tap guard)
Purchases not working in Editor	Editor uses mock payments — test real payments on TikTok device
Sync not running	Ensure RegisterSyncProvider() is called and IsDirtyForCloud = true
