# 1  WifiService的初始化

WifiService的初始化分为两个部分，WifiService的初始化和WifiManager的初始化，下面分别介绍。

## 1.1  WifiService的初始化流程

### 1.1.1  Wifiservice代码分析
WifiService的初始化流程是在SystemServer进程中，通过SystemServiceManager将WiFi的主服务WifiService启动。

```
@SystemServer.java
private static final String WIFI_SERVICE_CLASS =
        "com.android.server.wifi.WifiService";
private void startOtherServices() {
    if (context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_WIFI)) {
            // Wifi Service must be started first for wifi-related services.
            t.traceBegin("StartWifi");
            mSystemServiceManager.startServiceFromJar(
                WIFI_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
            t.traceEnd();
    }
}
```

WifiService启动过程中创建了WifiServiceImpl实例对象，并将该对象注册到ServiceManager中。WifiServiceImpl实例对象是整个系统中WiFi服务的管理者，接受来自WifiManager的请求，所有的用户端操作请求都将由它进行初步的处理，然后再分发给不同的服务执行相应的任务。
```
@WifiService.java
public final class WifiService extends SystemService {
    private static final String TAG = "WifiService";
    //创建WifiServiceImpl实例对象
    private final WifiServiceImpl mImpl;
    private final WifiContext mWifiContext;
    public WifiService(Context contextBase) {
        super(contextBase);
        mWifiContext = new WifiContext(contextBase);
        // 工具类，里面初始化了Wi-Fi相关的很多类实例，包含了WifiStateMachine，WifiController，WifiNative等重要属性的初始化工作
        WifiInjector injector = new WifiInjector(mWifiContext);
        // 对AsyncChannel简单的封装，在消息处理过程中增加了一些log，方便调试
        WifiAsyncChannel channel =  new WifiAsyncChannel(TAG);
        // 初始化Wi-Fi服务
        mImpl = new WifiServiceImpl(mWifiContext, injector, channel);
    }
    @Override
    public void onStart() {
        Log.i(TAG, "Registering " + Context.WIFI_SERVICE);
        //将WifiServiceImpl注册到ServiceManager，发布Wi-Fi服务
        publishBinderService(Context.WIFI_SERVICE, mImpl);
    }
        @Override
    public void onBootPhase(int phase) {
        if (phase == SystemService.PHASE_SYSTEM_SERVICES_READY) {
            createNotificationChannels(mWifiContext);
            //检查是否需要启动WiFi。如果关机前WiFi是打开的，则重启后WiFi功能将在此函数中打开。如果想让系统开机默认打开或关闭Wi-Fi，可以在这个地方修改。
            mImpl.checkAndStartWifi();
        } else if (phase == SystemService.PHASE_BOOT_COMPLETED) {
            // 等SystemServer执行到PHASE_BOOT_COMPLETED时，由SystemServer统一调用
            mImpl.handleBootCompleted();
        }
    }
```

WifiInjector的初始化如下所示
```
@WifiInjector.java
public class WifiInjector {
    ...
    // 从这个属性读取默认WiFi国家码
    private static final String BOOT_DEFAULT_WIFI_COUNTRY_CODE = "ro.boot.wificountrycode"; 
    static WifiInjector sWifiInjector = null;
    ...
    public WifiInjector(WifiContext context) {
        if (context == null) {
            throw new IllegalStateException(
                    "WifiInjector should not be initialized with a null Context.");
        }

        if (sWifiInjector != null) {
            throw new IllegalStateException(
                    "WifiInjector was already created, use getInstance instead.");
        }
        //单例模式时返回这个对象
        sWifiInjector = this;
        
        // Now create and start handler threads
        mAsyncChannelHandlerThread = new HandlerThread("AsyncChannelHandlerThread");
        mAsyncChannelHandlerThread.start();
        mWifiHandlerThread = new HandlerThread("WifiHandlerThread");
        mWifiHandlerThread.start();
        Looper wifiLooper = mWifiHandlerThread.getLooper();
        Handler wifiHandler = new Handler(wifiLooper);
        ...
        mNetworkScoreManager = mContext.getSystemService(NetworkScoreManager.class);
        mWifiNetworkScoreCache = new WifiNetworkScoreCache(mContext);
        mNetworkScoreManager.registerNetworkScoreCallback(NetworkKey.TYPE_WIFI,
                NetworkScoreManager.SCORE_FILTER_NONE,
                new HandlerExecutor(wifiHandler), mWifiNetworkScoreCache);
        ...
        mWifiBackupRestore = new WifiBackupRestore(mWifiPermissionsUtil);
        mSoftApBackupRestore = new SoftApBackupRestore(mContext, mSettingsMigrationDataHolder);
        mWifiStateTracker = new WifiStateTracker(mBatteryStats);
        mWifiThreadRunner = new WifiThreadRunner(wifiHandler);
        mWifiP2pServiceHandlerThread = new HandlerThread("WifiP2pService");
        mWifiP2pServiceHandlerThread.start();
        mPasspointProvisionerHandlerThread =
                new HandlerThread("PasspointProvisionerHandlerThread");
        mPasspointProvisionerHandlerThread.start();
        ...
        // Modules interacting with Native.
        mWifiMonitor = new WifiMonitor(this);
        mHalDeviceManager = new HalDeviceManager(mClock, wifiHandler);
        mWifiVendorHal = new WifiVendorHal(mHalDeviceManager, wifiHandler);
        mSupplicantStaIfaceHal = new SupplicantStaIfaceHal(
                mContext, mWifiMonitor, mFrameworkFacade, wifiHandler, mClock, mWifiMetrics);
        mHostapdHal = new HostapdHal(mContext, wifiHandler);
        mWifiCondManager = (WifiNl80211Manager) mContext.getSystemService(
                Context.WIFI_NL80211_SERVICE);
        mWifiNative = new WifiNative(
                mWifiVendorHal, mSupplicantStaIfaceHal, mHostapdHal, mWifiCondManager,
                mWifiMonitor, mPropertyService, mWifiMetrics,
                wifiHandler, new Random(), this);
        mWifiP2pMonitor = new WifiP2pMonitor(this);
        mSupplicantP2pIfaceHal = new SupplicantP2pIfaceHal(mWifiP2pMonitor);
        mWifiP2pNative = new WifiP2pNative(this,
                mWifiVendorHal, mSupplicantP2pIfaceHal, mHalDeviceManager,
                mPropertyService);

        // Now get instances of all the objects that depend on the HandlerThreads
        mWifiTrafficPoller = new WifiTrafficPoller(wifiHandler);
        mCountryCode = new WifiCountryCode(mContext, wifiHandler, mWifiNative,
                SystemProperties.get(BOOT_DEFAULT_WIFI_COUNTRY_CODE));
        ...
        mWifiScoreCard = new WifiScoreCard(mClock, l2KeySeed, mDeviceConfigFacade);
        ...
        // Config Manager
        mWifiConfigManager = new WifiConfigManager(mContext, mClock,
                mUserManager, mWifiCarrierInfoManager,
                mWifiKeyStore, mWifiConfigStore, mWifiPermissionsUtil,
                mWifiPermissionsWrapper, this,
                new NetworkListSharedStoreData(mContext),
                new NetworkListUserStoreData(mContext),
                new RandomizedMacStoreData(), mFrameworkFacade, wifiHandler, mDeviceConfigFacade,
                mWifiScoreCard, mLruConnectionTracker);
        mSettingsConfigStore = new WifiSettingsConfigStore(context, wifiHandler,
                mSettingsMigrationDataHolder, mWifiConfigManager, mWifiConfigStore);
        mSettingsStore = new WifiSettingsStore(mContext, mSettingsConfigStore);
        ...
        mWifiNetworkSelector = new WifiNetworkSelector(mContext, mWifiScoreCard, mScoringParams,
                mWifiConfigManager, mClock, mConnectivityLocalLog, mWifiMetrics, mWifiNative,
                mThroughputPredictor);
        CompatibilityScorer compatibilityScorer = new CompatibilityScorer(mScoringParams);
        mWifiNetworkSelector.registerCandidateScorer(compatibilityScorer);
        ScoreCardBasedScorer scoreCardBasedScorer = new ScoreCardBasedScorer(mScoringParams);
        mWifiNetworkSelector.registerCandidateScorer(scoreCardBasedScorer);
        BubbleFunScorer bubbleFunScorer = new BubbleFunScorer(mScoringParams);
        mWifiNetworkSelector.registerCandidateScorer(bubbleFunScorer);
        ThroughputScorer throughputScorer = new ThroughputScorer(mScoringParams);
        mWifiNetworkSelector.registerCandidateScorer(throughputScorer);
        mWifiMetrics.setWifiNetworkSelector(mWifiNetworkSelector);
        mWifiNetworkSuggestionsManager = new WifiNetworkSuggestionsManager(mContext, wifiHandler,
                this, mWifiPermissionsUtil, mWifiConfigManager, mWifiConfigStore, mWifiMetrics,
                mWifiCarrierInfoManager, mWifiKeyStore, mLruConnectionTracker);
        mPasspointManager = new PasspointManager(mContext, this,
                wifiHandler, mWifiNative, mWifiKeyStore, mClock, new PasspointObjectFactory(),
                mWifiConfigManager, mWifiConfigStore, mWifiMetrics, mWifiCarrierInfoManager,
                mMacAddressUtil, mWifiPermissionsUtil);
        PasspointNetworkNominateHelper nominateHelper =
                new PasspointNetworkNominateHelper(mPasspointManager, mWifiConfigManager,
                        mConnectivityLocalLog);
        mSavedNetworkNominator = new SavedNetworkNominator(
                mWifiConfigManager, nominateHelper, mConnectivityLocalLog, mWifiCarrierInfoManager,
                mWifiPermissionsUtil, mWifiNetworkSuggestionsManager);
        mNetworkSuggestionNominator = new NetworkSuggestionNominator(mWifiNetworkSuggestionsManager,
                mWifiConfigManager, nominateHelper, mConnectivityLocalLog, mWifiCarrierInfoManager);
        mScoredNetworkNominator = new ScoredNetworkNominator(mContext, wifiHandler,
                mFrameworkFacade, mNetworkScoreManager, mContext.getPackageManager(),
                mWifiConfigManager, mConnectivityLocalLog,
                mWifiNetworkScoreCache, mWifiPermissionsUtil);
        ...
        mScanRequestProxy = new ScanRequestProxy(mContext,
                (AppOpsManager) mContext.getSystemService(Context.APP_OPS_SERVICE),
                (ActivityManager) mContext.getSystemService(Context.ACTIVITY_SERVICE),
                this, mWifiConfigManager,
                mWifiPermissionsUtil, mWifiMetrics, mClock, wifiHandler, mSettingsConfigStore);
        mSarManager = new SarManager(mContext, makeTelephonyManager(), wifiLooper,
                mWifiNative);
        ...
        mWifiDataStall = new WifiDataStall(mFrameworkFacade, mWifiMetrics, mContext,
                mDeviceConfigFacade, mWifiChannelUtilizationConnected, mClock, wifiHandler,
                mThroughputPredictor);
        ...
        mLinkProbeManager = new LinkProbeManager(mClock, mWifiNative, mWifiMetrics,
                mFrameworkFacade, wifiHandler, mContext);
        SupplicantStateTracker supplicantStateTracker = new SupplicantStateTracker(
                mContext, mWifiConfigManager, mBatteryStats, wifiHandler);
        mMboOceController = new MboOceController(makeTelephonyManager(), mWifiNative);
        mWifiHealthMonitor = new WifiHealthMonitor(mContext, this, mClock, mWifiConfigManager,
                mWifiScoreCard, wifiHandler, mWifiNative, l2KeySeed, mDeviceConfigFacade);
        ...
        
        //ClientModeImpl即旧版本的WifiStateMachine，这个里面管理Client(STA)模式时的状态
        mClientModeImpl = new ClientModeImpl(mContext, mFrameworkFacade,
                wifiLooper, mUserManager,
                this, mBackupManagerProxy, mCountryCode, mWifiNative,
                new WrongPasswordNotifier(mContext, mFrameworkFacade),
                mSarManager, mWifiTrafficPoller, mLinkProbeManager, mBatteryStats,
                supplicantStateTracker, mMboOceController, mWifiCarrierInfoManager,
                new EapFailureNotifier(mContext, mFrameworkFacade, mWifiCarrierInfoManager),
                new SimRequiredNotifier(mContext, mFrameworkFacade));
        mActiveModeWarden = new ActiveModeWarden(this, wifiLooper,
                mWifiNative, new DefaultModeManager(mContext), mBatteryStats, mWifiDiagnostics,
                mContext, mClientModeImpl, mSettingsStore, mFrameworkFacade, mWifiPermissionsUtil);
        mWifiScanAlwaysAvailableSettingsCompatibility =
                new WifiScanAlwaysAvailableSettingsCompatibility(mContext, wifiHandler,
                        mSettingsStore, mActiveModeWarden, mFrameworkFacade);
        mWifiApConfigStore = new WifiApConfigStore(
                mContext, this, wifiHandler, mBackupManagerProxy,
                mWifiConfigStore, mWifiConfigManager, mActiveModeWarden, mWifiMetrics);
       ...
       mSelfRecovery = new SelfRecovery(mContext, mActiveModeWarden, mClock);
       ...
       // Register the various network Nominators with the network selector.
        mWifiNetworkSelector.registerNetworkNominator(mSavedNetworkNominator);
        mWifiNetworkSelector.registerNetworkNominator(mNetworkSuggestionNominator);
        mWifiNetworkSelector.registerNetworkNominator(mScoredNetworkNominator);
        
        //启动Client模式状态机
        mClientModeImpl.start();
    }
```


WifiServiceImpl的属性可以看出是一个服务的实现类，继承了IWifiManager.Stub，实现了IWifiManager定义的所有接口功能。
```
public class WifiServiceImpl extends BaseWifiService {}
public class BaseWifiService extends IWifiManager.Stub {}
```
WifiServiceImpl的初始化流程如下所示，在这里初始化各种与WIFI管理有关的辅助类：
```
@WifiServiceImpl.java
public class WifiServiceImpl extends BaseWifiService {
    ...
    // 主要就是从WifiInjector将一些对象赋值给自己的成员变量，获取了包括WifiStateMachine和WifiController在内的很多wifi变量，注册一些本类实现的回调函数
    public WifiServiceImpl(Context context, WifiInjector wifiInjector, AsyncChannel asyncChannel) {
        mContext = context;
        mWifiInjector = wifiInjector;
        mClock = wifiInjector.getClock();

        mFacade = mWifiInjector.getFrameworkFacade();
        mWifiMetrics = mWifiInjector.getWifiMetrics();
        mWifiTrafficPoller = mWifiInjector.getWifiTrafficPoller();
        mUserManager = mWifiInjector.getUserManager();
        mCountryCode = mWifiInjector.getWifiCountryCode();
        //ClientModeImpl 作为客户端的模式实现，作为客户端事件处理在这里完成，并所有的连接状态变化在这里初始化。
        mClientModeImpl = mWifiInjector.getClientModeImpl();
        //ActiveModeWarden 提供WiFi不同操作模式的配置实现。
        mActiveModeWarden = mWifiInjector.getActiveModeWarden();
        mScanRequestProxy = mWifiInjector.getScanRequestProxy();
        mSettingsStore = mWifiInjector.getWifiSettingsStore();
        mPowerManager = mContext.getSystemService(PowerManager.class);
        mAppOps = (AppOpsManager) mContext.getSystemService(Context.APP_OPS_SERVICE);
        mWifiLockManager = mWifiInjector.getWifiLockManager();
        mWifiMulticastLockManager = mWifiInjector.getWifiMulticastLockManager();
        /*
          这个handler类存在的意义，是作为mClientModeImplChannel的两个handler中的一个，
          另一个handler是ClientModeImpl所在的Handler，在mClientModeImplChannel连上这两个handler后就可以通过AsyncChannel机制在不同线程甚至不同进程间进行消息传递，
          这里主要是用来在WifiServiceImpl所在的线程与ClientModeImpl所在的线程WifiHandlerThread之间进行消息传递，
          后面可以看到WifiServiceImpl里面很多函数都是类似mClientModeImpl.syncGetCurrentNetwork(mClientModeImplChannel)这样的，要明白这个调用的意义。
        */
        mClientModeImplHandler = new ClientModeImplHandler(TAG,
                mWifiInjector.getAsyncChannelHandlerThread().getLooper(), asyncChannel);
        mWifiBackupRestore = mWifiInjector.getWifiBackupRestore();
        mSoftApBackupRestore = mWifiInjector.getSoftApBackupRestore();
        mWifiApConfigStore = mWifiInjector.getWifiApConfigStore();
        mWifiPermissionsUtil = mWifiInjector.getWifiPermissionsUtil();
        mLog = mWifiInjector.makeLog(TAG);
        mFrameworkFacade = wifiInjector.getFrameworkFacade();
        mTetheredSoftApTracker = new TetheredSoftApTracker();
        mActiveModeWarden.registerSoftApCallback(mTetheredSoftApTracker);
        mLohsSoftApTracker = new LohsSoftApTracker();
        mActiveModeWarden.registerLohsCallback(mLohsSoftApTracker);
        mWifiNetworkSuggestionsManager = mWifiInjector.getWifiNetworkSuggestionsManager();
        mDppManager = mWifiInjector.getDppManager();
        mWifiThreadRunner = mWifiInjector.getWifiThreadRunner();
        mWifiConfigManager = mWifiInjector.getWifiConfigManager();
        mPasspointManager = mWifiInjector.getPasspointManager();
        mWifiScoreCard = mWifiInjector.getWifiScoreCard();
        mMemoryStoreImpl = new MemoryStoreImpl(mContext, mWifiInjector,
                mWifiScoreCard,  mWifiInjector.getWifiHealthMonitor());
    }
}
    
```

### 1.1.2  ClientModeImpl代码分析
WifiServiceImpl的初始化，其中包含最重要的一个就是WIFI的状态机ClientModeImpl，他是整个WiFi机制的核心。状态机的初始化流程如下：
```
@ClientModeImpl.java
public class ClientModeImpl extends StateMachine {
    ...
    private boolean mVerboseLoggingEnabled = false;
    private final WifiPermissionsWrapper mWifiPermissionsWrapper;
    
    
    private final ExtendedWifiInfo mWifiInfo;
    ...
    public ClientModeImpl(Context context, FrameworkFacade facade, Looper looper,
                            UserManager userManager, WifiInjector wifiInjector,
                            BackupManagerProxy backupManagerProxy, WifiCountryCode countryCode,
                            WifiNative wifiNative, WrongPasswordNotifier wrongPasswordNotifier,
                            SarManager sarManager, WifiTrafficPoller wifiTrafficPoller,
                            LinkProbeManager linkProbeManager,
                            BatteryStatsManager batteryStatsManager,
                            SupplicantStateTracker supplicantStateTracker,
                            MboOceController mboOceController,
                            WifiCarrierInfoManager wifiCarrierInfoManager,
                            EapFailureNotifier eapFailureNotifier,
                            SimRequiredNotifier simRequiredNotifier) {
        super(TAG, looper);
        mWifiInjector = wifiInjector;
        //记录WiFi的连接失败次数，时间，路由器的信息等
        mWifiMetrics = mWifiInjector.getWifiMetrics();
        mClock = wifiInjector.getClock();
        //key
        mPropertyService = wifiInjector.getPropertyService();
        //user/userdebug build
        mBuildProperties = wifiInjector.getBuildProperties();
        //为网络选择和切换提供信息，帮助健康监视器检测网络问题
        mWifiScoreCard = wifiInjector.getWifiScoreCard();
        mContext = context;
        mFacade = facade;
        mWifiNative = wifiNative;
        mBackupManagerProxy = backupManagerProxy;
        mWrongPasswordNotifier = wrongPasswordNotifier;
        mEapFailureNotifier = eapFailureNotifier;
        mSimRequiredNotifier = simRequiredNotifier;
        //提供SAR控制WiFi TX功率的限制
        mSarManager = sarManager;
        //轮询流量统计并通知客户端
        mWifiTrafficPoller = wifiTrafficPoller;
        //一个状态，决定是否应该执行链路探测
        mLinkProbeManager = linkProbeManager;
        //负责控制多频段操作（MBO）和优化的连接体验（OCE）的操作。
        mMboOceController = mboOceController;
        //提供API从电话服务获得运营商信息
        mWifiCarrierInfoManager = wifiCarrierInfoManager;
        //NetworkInfo目前已禁用
        mNetworkAgentState = DetailedState.DISCONNECTED;
        //提供API，报告WiFi的耗电情况
        mBatteryStatsManager = batteryStatsManager;
        //用于跟踪WifiState以更新BatteryStats
        mWifiStateTracker = wifiInjector.getWifiStateTracker();
        //判断是否支持P2P
        mP2pSupported = mContext.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_WIFI_DIRECT);
        //判断能不能调WiFi的扫描结果
        mWifiPermissionsUtil = mWifiInjector.getWifiPermissionsUtil();
        //提供管理已配置WiFi网络的API
        mWifiConfigManager = mWifiInjector.getWifiConfigManager();

        mPasspointManager = mWifiInjector.getPasspointManager();

        mWifiMonitor = mWifiInjector.getWifiMonitor();
        //跟踪framework的日志
        mWifiDiagnostics = mWifiInjector.getWifiDiagnostics();
        //一个wifi权限依赖类，用于封装对静态方法的外部调用，以支持测试。
        mWifiPermissionsWrapper = mWifiInjector.getWifiPermissionsWrapper();
        //搜WiFi
        mWifiDataStall = mWifiInjector.getWifiDataStall();
        //使用计算平均数据包速率的方法扩展WifiInfo
        mWifiInfo = new ExtendedWifiInfo(context);
        //跟踪wpa_supplicant的状态切换
        mSupplicantStateTracker = supplicantStateTracker;
        //管理所有与连接相关的扫描活动。
        mWifiConnectivityManager = mWifiInjector.makeWifiConnectivityManager(this);
        //管理添加和删除BSSID到BSSID黑名单，用于固件漫游和网络选择。
        mBssidBlocklistMonitor = mWifiInjector.getBssidBlocklistMonitor();
        //WiFi连接失败提醒
        mConnectionFailureNotifier = mWifiInjector.makeConnectionFailureNotifier(
                mWifiConnectivityManager);
        //描述网络链路的属性，不能用于改变网络
        mLinkProperties = new LinkProperties();
        //
        mMcastLockManagerFilterController = new McastLockManagerFilterController();
        mActivityManager = context.getSystemService(ActivityManager.class);

        mLastBssid = null;
        mLastNetworkId = WifiConfiguration.INVALID_NETWORK_ID;
        mLastSubId = SubscriptionManager.INVALID_SUBSCRIPTION_ID;
        mLastSimBasedConnectionCarrierName = null;
        mLastSignalLevel = -1;

        mCountryCode = countryCode;

        mWifiScoreReport = new WifiScoreReport(mWifiInjector.getScoringParams(), mClock,
                mWifiMetrics, mWifiInfo, mWifiNative, mBssidBlocklistMonitor,
                mWifiInjector.getWifiThreadRunner(), mWifiInjector.getDeviceConfigFacade(),
                mContext, looper, mFacade);

        NetworkCapabilities.Builder builder = new NetworkCapabilities.Builder()
                .addTransportType(NetworkCapabilities.TRANSPORT_WIFI)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_METERED)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_ROAMING)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_CONGESTED)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_RESTRICTED)
                .addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_SUSPENDED)
                // TODO - needs to be a bit more dynamic
                .setLinkUpstreamBandwidthKbps(1024 * 1024)
                .setLinkDownstreamBandwidthKbps(1024 * 1024)
                .setNetworkSpecifier(new MatchAllNetworkSpecifier());
        if (SdkLevel.isAtLeastS()) {
            builder.addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_VCN_MANAGED);
        }
        mNetworkCapabilitiesFilter = builder.build();

        // Make the network factories.
        mNetworkFactory = mWifiInjector.makeWifiNetworkFactory(
                mNetworkCapabilitiesFilter, mWifiConnectivityManager);
        // We can't filter untrusted network in the capabilities filter because a trusted
        // network would still satisfy a request that accepts untrusted ones.
        // We need a second network factory for untrusted network requests because we need a
        // different score filter for these requests.
        mUntrustedNetworkFactory = mWifiInjector.makeUntrustedWifiNetworkFactory(
                mNetworkCapabilitiesFilter, mWifiConnectivityManager);

        mWifiNetworkSuggestionsManager = mWifiInjector.getWifiNetworkSuggestionsManager();
        mProcessingActionListeners = new ExternalCallbackTracker<>(getHandler());
        mWifiHealthMonitor = mWifiInjector.getWifiHealthMonitor();

        IntentFilter filter = new IntentFilter();
        filter.addAction(Intent.ACTION_SCREEN_ON);
        filter.addAction(Intent.ACTION_SCREEN_OFF);
        mContext.registerReceiver(
                new BroadcastReceiver() {
                    @Override
                    public void onReceive(Context context, Intent intent) {
                        String action = intent.getAction();

                        if (action.equals(Intent.ACTION_SCREEN_ON)) {
                            sendMessage(CMD_SCREEN_STATE_CHANGED, 1);
                        } else if (action.equals(Intent.ACTION_SCREEN_OFF)) {
                            sendMessage(CMD_SCREEN_STATE_CHANGED, 0);
                        }
                    }
                }, filter);

        PowerManager powerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);

        mSuspendWakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "WifiSuspend");
        mSuspendWakeLock.setReferenceCounted(false);

        mWifiConfigManager.addOnNetworkUpdateListener(new OnNetworkUpdateListener());

        // CHECKSTYLE:OFF IndentationCheck
        addState(mDefaultState);
            addState(mConnectModeState, mDefaultState);
                addState(mL2ConnectedState, mConnectModeState);
                    addState(mObtainingIpState, mL2ConnectedState);
                    addState(mConnectedState, mL2ConnectedState);
                    addState(mRoamingState, mL2ConnectedState);
                addState(mDisconnectingState, mConnectModeState);
                addState(mDisconnectedState, mConnectModeState);
        // CHECKSTYLE:ON IndentationCheck

        setInitialState(mDefaultState);

        setLogRecSize(NUM_LOG_RECS_NORMAL);
        setLogOnlyTransitions(false);
    }
    
    
    
} 
```




下面看一下checkAndStartWifi()的实现：
```
    /**
     * Check if we are ready to start wifi.
     *
     * First check if we will be restarting system services to decrypt the device. If the device is
     * not encrypted, check if Wi-Fi needs to be enabled and start if needed
     *
     * This function is used only at boot time.
     */
    public void checkAndStartWifi() {
        mWifiThreadRunner.post(() -> {
        	// 首先，从/data/misc/wifi/WifiConfigStore.xml加载配置
            if (!mWifiConfigManager.loadFromStore()) {
                Log.e(TAG, "Failed to load from config store");
            }
            // config store is read, check if verbose logging is enabled.
            enableVerboseLoggingInternal(getVerboseLoggingLevel());
            // Check if wi-fi needs to be enabled
            // 从SettingsProvider检查Wi-Fi开关状态
            boolean wifiEnabled = mSettingsStore.isWifiToggleEnabled();
            Log.i(TAG,
                    "WifiService starting up with Wi-Fi " + (wifiEnabled ? "enabled" : "disabled"));
			
			// 开始监听SettingsProvider中ScanAlwaysAvailable字段
            mWifiInjector.getWifiScanAlwaysAvailableSettingsCompatibility().initialize();
            // 下面几个监听SIM卡状态的都是为了更新Passpoint网络永久/临时鉴权信息的
            mContext.registerReceiver(
                    new BroadcastReceiver() {
                        @Override
                        public void onReceive(Context context, Intent intent) {
                            if (mSettingsStore.handleAirplaneModeToggled()) {
                                mActiveModeWarden.airplaneModeToggled();
                            }
                        }
                    },
                    new IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED));

            mContext.registerReceiver(
                    new BroadcastReceiver() {
                        @Override
                        public void onReceive(Context context, Intent intent) {
                            int state = intent.getIntExtra(TelephonyManager.EXTRA_SIM_STATE,
                                    TelephonyManager.SIM_STATE_UNKNOWN);
                            if (TelephonyManager.SIM_STATE_ABSENT == state) {
                                Log.d(TAG, "resetting networks because SIM was removed");
                                mClientModeImpl.resetSimAuthNetworks(
                                        ClientModeImpl.RESET_SIM_REASON_SIM_REMOVED);
                            }
                        }
                    },
                    new IntentFilter(TelephonyManager.ACTION_SIM_CARD_STATE_CHANGED));

            mContext.registerReceiver(
                    new BroadcastReceiver() {
                        @Override
                        public void onReceive(Context context, Intent intent) {
                            int state = intent.getIntExtra(TelephonyManager.EXTRA_SIM_STATE,
                                    TelephonyManager.SIM_STATE_UNKNOWN);
                            if (TelephonyManager.SIM_STATE_LOADED == state) {
                                Log.d(TAG, "resetting networks because SIM was loaded");
                                mClientModeImpl.resetSimAuthNetworks(
                                        ClientModeImpl.RESET_SIM_REASON_SIM_INSERTED);
                            }
                        }
                    },
                    new IntentFilter(TelephonyManager.ACTION_SIM_APPLICATION_STATE_CHANGED));

            mContext.registerReceiver(
                    new BroadcastReceiver() {
                        private int mLastSubId = SubscriptionManager.INVALID_SUBSCRIPTION_ID;
                        @Override
                        public void onReceive(Context context, Intent intent) {
                            final int subId = intent.getIntExtra("subscription",
                                    SubscriptionManager.INVALID_SUBSCRIPTION_ID);
                            if (subId != mLastSubId) {
                                Log.d(TAG, "resetting networks as default data SIM is changed");
                                mClientModeImpl.resetSimAuthNetworks(
                                        ClientModeImpl.RESET_SIM_REASON_DEFAULT_DATA_SIM_CHANGED);
                                mLastSubId = subId;
                            }
                        }
                    },
                    new IntentFilter(TelephonyManager.ACTION_DEFAULT_DATA_SUBSCRIPTION_CHANGED));

            // Adding optimizations of only receiving broadcasts when wifi is enabled
            // can result in race conditions when apps toggle wifi in the background
            // without active user involvement. Always receive broadcasts.
            registerForBroadcasts();
            mInIdleMode = mPowerManager.isDeviceIdleMode();
			
			// 初始化WifiNative, HAL Service等
            mClientModeImpl.initialize();
            // 启动ClientModeManager和WifiController，注意在R上面WifiController变成了ActiveModeWarden的内部类了
            mActiveModeWarden.start();
            registerForCarrierConfigChange();
        });
    }
 ```   
    
    

到此，系统启动完成，WifiService服务已正常运行。

到这里，Wi-Fi服务，WifiNative, Wifi HAL Service, wpa_supplicant, wlan driver，wlan firmware, ConnectivityServcie, NetworkStack, DNSResolver都初始化好了，剩下的就是等待客户端调用。










