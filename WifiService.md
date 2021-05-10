# 一、WIFI服务的初始化

###### WIFI服务的初始化分为两个部分，WifiService的初始化和WifiManager的初始化，下面分别介绍。

## 1.1、WifiService的初始化流程

##### WifiService的初始化流程是在SystemService中被启动的：
```
        @SystemServer.java
        private static final String WIFI_SERVICE_CLASS = "com.android.server.wifi.WifiService";
        private void startOtherServices() {
            mSystemServiceManager.startService(WIFI_SERVICE_CLASS);
        }
```
##
