# 一、WifiService的初始化

WifiService的初始化分为两个部分，WifiService的初始化和WifiManager的初始化，下面分别介绍。

## 1.1、WifiService的初始化流程

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

WifiService启动过程：
```
@WifiService.java
public final class WifiService extends SystemService {
    private static final String TAG = "WifiService";
    private final WifiServiceImpl mImpl;
    private final WifiContext mWifiContext;
    public WifiService(Context contextBase) {
        super(contextBase);
        mWifiContext = new WifiContext(contextBase);
        WifiInjector injector = new WifiInjector(mWifiContext);
        WifiAsyncChannel channel =  new WifiAsyncChannel(TAG);
        mImpl = new WifiServiceImpl(mWifiContext, injector, channel);
    }
    @Override
    public void onStart() {
        Log.i(TAG, "Registering " + Context.WIFI_SERVICE);
        publishBinderService(Context.WIFI_SERVICE, mImpl);
    }
        @Override
    public void onBootPhase(int phase) {
        if (phase == SystemService.PHASE_SYSTEM_SERVICES_READY) {
            createNotificationChannels(mWifiContext);
            mImpl.checkAndStartWifi();
        } else if (phase == SystemService.PHASE_BOOT_COMPLETED) {
            mImpl.handleBootCompleted();
        }
    }
```
##
