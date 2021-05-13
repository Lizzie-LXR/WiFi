# Android 名词
## AIDL
AIDL是Android Interface Definition Language，即Android接口定义语言。首先它是一种语言，它被设计出来的目的就是为了实现进程间的通信。 通过使用AIDL，可以帮我们生成进程间通信时需要用到的类和接口方法。 当然，我们也可以不借助AIDL，而是自己实现这些类和方法，但是借助AIDL会让这个过程变得简单方便。

# Android WiFi 
Android WiFi系统引入了wpa_supplicant，它的整个WiFi系统以wpa_supplicant为核心来定义上层用户接口和下层驱动接口。WiFi框架如下图所示：

![image](https://github.com/Lizzie-LXR/WiFi/blob/main/IMG/android_o_wifi%E6%A1%86%E6%9E%B6.png)

WiFi各层关系如下所示：
![image](https://github.com/Lizzie-LXR/WiFi/blob/main/IMG/wifi层.png)

下面将一层一层进行介绍。

## WifiSettings
WiFi App层

### WifiSettings
操作WiFi的App。

### WifiEnabler
负责WiFi的开关逻辑。

## WiFi Framework
Android系统用来维护WiFi整个逻辑的框架代码。

### WifiManager
WifiManager管理所有Wifi连接的基本API，WIFI的多数功能都以该类的方法的形式提供。其它所有应用都可以通过WifiManager来操作Wifi的各项功能，但是WifiManager本身不具备处理请求的能力，而是把所有的请求转发给WifServiceImpl来处理。WifiManager主要是由IWifiManager和IWifi组成。

IWifiManager是一个接口，IWifiManager, IWifiManager.Stub, IWifiManager.Stub.Proxy都由IWifiManger.aidl生成；aidl自动生成相关的java代码，简化了用Binder实现RPC的过程。IWifiManager.Stub, IWifiManager.Stub.Proxy用于实现IWifiManager接口。

IWifi

### WifiController
高级别的WiFi状态机，它管理的状态是WiFi开关，或者说是作为STA/SoftAp/P2P角色

### WifiService 
Java Framework中Wifi功能的总入口，负责Wifi功能的核心业务，由SystemServer启动的时候生成的ConnecttivityService创建。它是服务器端的实现，作为Wifi部分的核心，处理实际的驱动加载、扫描、链接、断开等命令，以及底层上报的事件。对于主动的命令控制，WiFi是一个简单的封装，针对来自客户端的控制命令，处理其它模块通过IWifiManager接口发送过来的远端WiFi操作，调用相应的WifiNative底层实现，负责启动关闭wpa_supplicant,启动和关闭WifiMonitor线程，把命令下发给wpa_supplicant以及更新WIFI的状态。

WifiService的功能在WifiServiceImpl中实现。

### WifiMonitor
专门负责接收来自Wpa_supplicant的事件，将这些信息进行分类，通过广播等方式分发给关注事件的Listener，交予StateMachine处理。

### WifiStateTracker:
除了负责WiFi的电源管理模式等功能外，其核心是WifiMonitor所实现的事件轮询机制，以及消息处理函数handleMessage()。

### WifiNative:
一个接口类，主要是提供一些native方法用于wifi framework层和WPAS通信。WifiNative的主要实现都在wifi.c函数里,WifiNative不过是将其封装,供framework层调用。

## JNI
android_net_wifi_Wifi.cpp就是典型jni接口，通过它可以直接调用WiFi的硬件抽象层。在Android O之后已禁用，改为HIDL方法。

## HAL/HIDL
HAL也叫wpa_supplicant适配层，是通用wpa_supplicant的封装，并向上层提供操作wpa_supplicant的接口，是wpa_supplicant的上层调用者，以供给WiFi框架层使用。
HIDL的功能就是为了java可以直接调用到c/c++，wifi的framework层动用的Iwifi的类，Iwifi.hal文件定义了未实现的接口。 HIDL起着承上启下的作用，主要用于Framework与wpa_supplicant守护进程的通信。

## wpa_supplicant
该层是Wifi FrameWork层的基石，也叫Wifi服务层。包含两个互相相关的开源项目wpa_supplicant和hostapd，它们将会生成两个可执行文件：wpa_supplicant和hostapd，分别为STA模式和AP模式时的守护进程。主要用来支持WEP，WPA/WPA2/WPA3和WAPI无线协议和加密认证的，而实际上的工作内容是通过socket与驱动交互，并上报数据给用户，而用户可以通过socket发送命令给wpa_supplicant调动驱动来对WiFi芯片进行操作。简单来说，wpa_supplicant是WiFi驱动和用户的中转站，并负责对协议和加密认证的支持。supplicant也是一个通道，理论上它自己可以管理STA工作了，现在Android也在基于它搞，
最主要核心是为了用它的wifi加密认证相关的功能，其他功能其实没它也能弄。扫描，连接，密钥认证。P2P，SoftAP对比STA来说，有更多的逻辑是在supplicant或对等的hostapd中。
以driver+supplicant来说：分析STA问题，绝大部分需要看driver，分析P2P，看supplicant比例多很多。

## kernel/Driver
厂商提供的source。

## WiFi IPC
WifiService，WifiManager的关系如下图所示：

![image](https://github.com/Lizzie-LXR/WiFi/blob/main/IMG/wifiservice%26wifimanager%E7%B1%BB%E5%9B%BE.jpg)

由上图可知，
WifiService继承自IWifiManager.Stub；IWifiManager.Stub又继承自Binder，同时实现了IWifiManager接口;WifiManager.Stu.proxy也实现了IWifiManager接口。

IWifiManager.Stub.Proxy，WifiManager，BinberProxy用于客户端（App进程）；而IWifiManager.Stub，WifiService，Binder用于服务端（SystemServer进程）。

WiFi是通过Binder进行通信的，Binder是连接App和SystemServer的桥梁，WifiManager和WiFiService是桥梁的两头。App与SystemServer通过Binder通信，但Binder本身只实现了IPC，即类似socket通信的能力。而App端的WifiManager和SystemServer端的WifiService及Binder等类共同实现了RPC(remote procedure call)。

WifiManager只是系统为App提供的接口，返回的实际对象类型是IWifiManager.Stub.Proxy，将WifiManager方法的参数序列化到Parcel，再经Binder发送给SystemServer进程。SystemServer内的WifiService收App传来的WifiManager调用，完成实际工作。这样，实际和下层通信的工作就由App转移到了SystemServer进程（WifiService对象）。

## Android HSM
### state模式
state的创建是为了一个对象在其内部状态改变时改变它的行为，对象看起来修改了它的类。Command模式是将命令请求封装成一个为对象，将不同的请求对象参数化以达到同样的调用执行不同的命令；同样State模式是将对象的状态封装成一个对象，是在不同的状态下同样的调用执行不同的操作。用于 1.一个对象的行为取决于它的状态，并且它必须在运行时刻根据状态来改变它的行为；2.一个操作中含有庞大的分支条件语句，且这些分支依赖于该对象的状态。
state结构图如下所示：

![image](https://github.com/Lizzie-LXR/WiFi/blob/main/IMG/state%E7%8A%B6%E6%80%81.png)

### HSM
从宏观上来讲，StateMachine机制是State模式在Android中的应用。HSM在传统的State模式基础上做了一些改变，使状态之间有了层次关系。这种层次关系和Java中父子类的派生和继承关系类似，都是由父类实现Generic的功能，而在子类实现一些特定的处理；但是，HSM中的父子类对应的是毫无派生关系的两个类，所以需要创建两个对象，而在Java中，创建一个子类对象即可完成父类的工作。hsm的机构图如下所示：

![image](https://github.com/Lizzie-LXR/WiFi/blob/main/IMG/hsm%E7%BB%93%E6%9E%84.png)

ClientModeImpl是WifiService的核心。








