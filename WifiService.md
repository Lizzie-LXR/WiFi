# 一、WifiService的初始化

WifiService的初始化分为两个部分，WifiService的初始化和WifiManager的初始化，下面分别介绍。

## 1.1、WifiService的初始化流程

WifiService的初始化流程是在SystemServer进程中：

```
    private static final String WIFI_SERVICE_CLASS =
            "com.android.server.wifi.WifiService";
```
```
        if (context.getPackageManager().hasSystemFeature(
                    PackageManager.FEATURE_WIFI)) {
                // Wifi Service must be started first for wifi-related services.
                t.traceBegin("StartWifi");
                mSystemServiceManager.startServiceFromJar(
                        WIFI_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
                t.traceEnd();
                t.traceBegin("StartWifiScanning");
                mSystemServiceManager.startServiceFromJar(
                        WIFI_SCANNING_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
                t.traceEnd();
            }
```
##
