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
        WifiInjector injector = new WifiInjector(mWifiContext);
        WifiAsyncChannel channel =  new WifiAsyncChannel(TAG);
        mImpl = new WifiServiceImpl(mWifiContext, injector, channel);
    }
    @Override
    public void onStart() {
        Log.i(TAG, "Registering " + Context.WIFI_SERVICE);
        //将WifiServiceImpl注册到ServiceManager
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
    //ClientModeImpl 作为客户端的模式实现，作为客户端事件处理在这里完成，并所有的连接状态变化在这里初始化。
    private final ClientModeImpl mClientModeImpl;
    //ActiveModeWarden 提供WiFi不同操作模式的配置实现。
    private final ActiveModeWarden mActiveModeWarden;
    private final ScanRequestProxy mScanRequestProxy;
    private final Context mContext;
    ...
}
    
```

### 1.1.2  ClientModeImpl代码分析
WifiServiceImpl的初始化，其中包含最重要的一个就是WIFI的状态机ClientModeImpl，他是整个WiFi机制的核心。状态机的初始化流程如下：
```
@ClientModeImpl.java
public class ClientModeImpl extends StateMachine {
    ...
    
    
    
    private final ExtendedWifiInfo mWifiInfo;
    
    
    
```

到此，系统启动完成，WifiService服务已正常运行。










