# Android 名词
## AIDL
AIDL是Android Interface Definition Language，即Android接口定义语言。首先它是一种语言，它被设计出来的目的就是为了实现进程间的通信。 通过使用AIDL，可以帮我们生成进程间通信时需要用到的类和接口方法。 当然，我们也可以不借助AIDL，而是自己实现这些类和方法，但是借助AIDL会让这个过程变得简单方便。

# Android WiFi 框架
WifiService，WifiManager的关系如下图所示：



## WifiService 

## WifiManager
WifiManager管理所有Wifi连接的基本API，WIFI的多数功能都以该类的方法的形式提供。WifiManager主要是由IWifiManager和IWifi组成，
如果说Binder是连接 App 和 system_server 的桥梁，那么WifiManager和WiFiService就是桥梁的两头。App 与 system_server 通过Binder通信，但Binder本身只实现了IPC，即类似socket通信的能力。而App端的WifiManager和system_server端的WifiService及Binder等类共同实现了RPC(remote procedure call)。
WifiManager只是系统为App提供的接口，返回的实际对象类型是IWifiManager.Stub.Proxy，将WifiManager方法的参数序列化到Parcel，再经Binder发送给system_server进程。system_server内的WifiService收App传来的WifiManager调用，完成实际工作。这样，实际和下层通信的工作就由App转移到了system_server进程（WifiService对象）。

## IWifiManager
IWifiManager是一个接口，IWifiManager, IWifiManager.Stub, IWifiManager.Stub.Proxy都由IWifiManger.aidl生成；aidl自动生成相关的java代码，简化了用Binder实现RPC的过程。
IWifiManager.Stub, IWifiManager.Stub.Proxy用于实现IWifiManager接口，
IWifiManager.Stub.Proxy，WifiManager，BinberProxy用于客户端（App进程）；而IWifiManager.Stub，WifiService，Binder用于服务端（SystemServer进程）。
