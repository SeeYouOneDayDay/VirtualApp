# VirtualApp技术黑产利用研究报告
## **一、 前言**

**VirtualApp（以下称 VA）是一个 App 虚拟化引擎（简称 VA）。VirtualApp 创建了一个虚拟空间，你可以在虚拟空间内任意的安装、启动和卸载 APK，这一切都与外部隔离，如同一个沙盒。运行在 VA 中的 APK 无需在 Android 系统中安装即可运行，也就是我们熟知的多开应用。**

VA 免安装运行 APK 的特性使得 VA 内应用与 VA 相比具有不同的应用特征，这使得 VA 可用于免杀。此外，VA 对被多开应用有较大权限，可能构成安全风险。

本报告首先简要介绍 VA 的多开实现原理，之后分析目前在灰色产业的应用，针对在免杀的应用，安全云对此的应对，并给出色情应用作为例子。另一方面，通过对样本分析，展示了 VA 对于安装在其内应用的高度控制能力，及其带来的安全风险。最后对本报告进行总结。

## **二、 VirtualApp 原理**

Android 应用启动 Activity 时，无论通过何种 API 调用，最终会调用到 ActivityManager.startActivity() 方法。该调用为远程 Binder 服务 (加速该调用，Android 应用会先在本地进程查找 Binder 服务缓存，如果找到，则直接调用。VA 介入了该调用过程，通过以下方式：

1. 替换本地的 ActivityManagerServise Binder 服务为 VA 构造的代理对象，以接管该调用。这一步通过反射实现。 2. 接管后，当调用 startActivity 启动多开应用时，VA 修改 Intent 中的 Activity 为 VA 中己声明的占位 Activity。这一步的目的是绕过 Android 无法启动未在 AndroidManifest.xml 中声明 Activity 的限制。 3. 在被多开应用进程启动后，增加 ActivityThread.mH.mCallback 的消息处理回调。这一步接管了多开应用主线程的消息回调。

在以上修改的基础上，多开应用的 Activity 启动过程可分为以下两步骤：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018618.jpeg)



**步骤一 修改 Activity 为己声明的 StubActivity**



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018627.jpeg)



**步骤二 mCallback 从 Intent 中恢复 Acitivty 信息**

AMS：Android 系统的 ActivityManagerService，是管理 Activity 的系统服务

VAMS：VA 用于管理多开应用 Activity 的服务，大量 API 名称与 AMS 雷同。

VApp：被多开应用所在的进程，该进程实际为 VA 派生的进程。

由图可知，VA 在 AMS 和 VApp 中通过增加 VAMS 对启动 Intent 进行了修改，实现了对 Android 系统的欺骗，而当应用进程启动后，还原 Activity 信息。通过自定义 Classloader 令使得 Android 加载并构造了未在 VA 的 AndroidManifest.xml 中声明的 Activity。

以上是启动过程的简化描述，实际上，VA 对大量 Android 系统 API 进行了 Hook，这使得运行在其中的应用在 VA 的控制下，为 VA 的应用带来可能性。

## **三、 在灰色产业的应用**

### **3.1 免杀**

VA 等多开工具将 Android 系统与 VA 内的应用隔离，使得应用的静态特征被掩盖，目前己有恶意应用使用 VA 对自身重打包，重打包后的应用包名、软件名与原应用不同，从而实现免杀。安全云使用动态检测关联恶意应用和 VA 的方式应对该免杀技术。

免杀的常见做法是：恶意应用加密后打包在 VA 内，由 VA 在运行时解密 APK，将恶意应用的 APK 安装到 VA 内并运行。

经过打包后，VA 用的包名、证书可以与恶意应用不同，资源文件、二进制库文件与恶意应用相互独立。基于包名、证书等特征维度的静态检测方式的准确性受到影响。

如图，当静态引擎对 VA 应用检测时，获得的应用信息（包名、证书、代码等）是 VA 的信息，没有恶意特征。而当 VA 运行时，可以解密恶意应用 APK，通过反射等技术欺骗 Android 系统运行未安装在系统中的 APK，实现了免杀。



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018631.jpeg)



传统静态检测方式

针对该免杀方式，安全云的 APK 动态检测实现了 VA 内应用 APK 的自动化提取，可将 VA 母包与恶意应用 APK 子包进行关联查杀。

如图，动态引擎安装并启动 APK，当识别出是 VA 应用时，提取出 VA 内的己解密的子包，对提取的子包进行检测。根据子包检测结果综合判定母包安全性，并对母包的安全风险进行标记查杀。



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018638.jpeg)



动态检测查杀示意图



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018645.jpeg)



**免杀案例 色情应用**

2017 年 8 月以来，安全云监测到部分色情应用使用 VA 的对自身打包，以达到绕过安全检测的目的。这些应用使用了随机的包名和软件名，并且均对恶意应用子包进行了加密。部分包名如表所示：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018670.jpeg)



从动态检测引擎提取的子包看，一个色情应用子包对应的带 VA 壳的母包（SHA1 维度）数量从 1 到 529 不等。27 个色情 APK 共对应 1464 个 VA 母包。



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018964.jpeg)



该类应用会以各种理由诱导用户升到更高等级的 VIP 不断支付：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018280.jpeg)



读取用户[短信](https://cloud.tencent.com/product/sms?from=10680)收件箱：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018690.jpeg)



并且可以通过远程[服务器](https://cloud.tencent.com/product/cvm?from=10680)控制应用是否运行，控制支付宝和微信支付的开启以逃避支付平台打击：



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/0xie1xu3rg.jpeg)



目前该类色情应用的 VA 母包和子包均己被标记为灰色。

### **3.2 重打包**

相较于以往反编译后插入代码进行打包编译的方式，使用 VA 进行重打包具有以下优点：

**1. 不需要对原应用进行反编译修改。**

由于 VA 是多开工具，这一优点是显然的。传统的重打包方式是对应用进行反编译成 Smali 代码，对 Application 类或 Activity 进行修改，插入广告展示等代码，再重编译打包回去。而 VA 重打包的应用只要让应用运行在 VA 内即可。

**2. 有效规避重打包检测**

应用可能通过检测签名、包名等方式检查是否被修改。而 VA 对 Android 系统的 API 进行了 Hook，其中包括 PackageManager 的 API，这些 API 用于获得包括签名在内的软件包信息。

**3. 通用性强**

同样的 VA 代码未经修改就可打包众多应用，可批量制造多开应用。

**4. 功能众多**

由于应用运行在 VA 进程内，VA 代码具有与应用等同的权限，从下面的例子可知 VA 能做到包括但不限于：模拟点击、截图、在 Activity 创建时插入广告。以

以某一类重打包样本为例，应用被重打包后启动界面增加一个插屏广告，如图：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018176.jpeg)



点击插屏广告后，将下载对应的应用（图中为 “斗地主” 及”炸金花”）并安装到 VA 中，并在桌面添加图标。区别于其他应用推广方式，此种方式安装的应用不必通过 Android 的包管理器进行安装，必要时也可静默安装。

除了增加启动时的插屏广告，该应用还有以下行为

**1. 检查反病毒软件**

检查手机上是否安装了反病毒软件，如果存在，则不连接服务器获取命令。

反病毒软件列表由服务器下发：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018528.jpeg)



内容如下：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018836.jpeg)



主流的手机安全应用如 30、QQ 手机管家、LE 安全大师等均在该列表中。

如果存在，则不连接服务器读取命令脚本：



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/6ptt8wmixi.jpeg)



**2. 启动应用**

可由服务器下发指令控制运行 VA 内的指定应用：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018056.jpeg)



**3. 模拟点击**

可对运行在 VA 内的应用进行点击。

1) 当 VA 内应用启动时注册 Broadcast Receiver：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018102.jpeg)



2) 接收服务器脚本，发送广播



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018946.jpeg)



3) 执行点击脚本

（1） 获得 DecorViews，该 View 为 Android 应用的底层 View。因为被多开的应用跑在 VA 内，因此 VA 有权限对应用类进行操作。



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018596.jpeg)



（2） 对 (1) 获得的 View，调用 View.dispatchTouchEvent()模拟触摸操作，支持的操作有，ACTION_DOWN（按下）、ACTION_MOVE（按下和抬起之间的操作）、ACTION_UP（抬起）。



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/mzeu415m9f.jpeg)



（3） 值得注意的是，只有当用户不存在（未点亮屏幕，未锁屏）时，服务器的任务才会执行：



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/ib4cgm1ase.jpeg)



**4. 部分版本可对应用界面进行截图**

实现方式与模拟触摸操作类似，先获得 DecorView，之后调用 Android 系统提供的方法进行截屏：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018514.jpeg)



对广播进行响应，并保存截图：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018142.jpeg)



相应的上传截图功能：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018213.jpeg)



**5. 在 Activity 创建时显示广告**

VA 对 Activity 的生命周期函数进行了 Hook，因此可以方便地在 Activity 调用 onCreate 函数时显示广告：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018472.jpeg)





![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018340.jpeg)



**6. 上传设备信息**



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018382.jpeg)



包括设备的型号、Android Id、分辨率等信息。

**7. 上传己安装应用列表**



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018396.jpeg)



### **3.3 免 Root Hook**

VA 可在应用 Application 类创建时执行代码，这些代码先于应用执行。通过结合 Hook 框架（如 YAHFA、AndFix）、VA 可以方便对应用进行 Hook，其 Hook 能力与 Xposed 框架等同。与 Xposed 框架比较如表所示：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018952.jpeg)



相较于 Xposed 框架，通过此方式 Hook 具有如下优点：

\1. 不需要 Root 权限 2. 不需要重启系统就可以重新加载 Hook 代码，重启应用即可 3. 可与 Native Hook 框架结合，Hook 二进制库。实际上 VA 本身己使用 Native Hook 框架对应用的 IO 操作进行了重定向

VA 的免 Root Hook 能力对于被多开应用是一种安全威胁。VA 可做到的包括但不限于：

\1. Hook 密码相关函数，截取用户输入的密码 2. Hook 网络通信函数，监听网络通信 3. Hook Android API。伪造 Android 设备信息、GPS 定位记录等。

下面分析某微信抢红包应用，以展示 VA 免 Root Hook 的能力。

该样本是一个微信抢红包应用。目前流行的抢红包功能实现上有两种方案，一种是通过 Android AccessiblityServices 监测用户窗口，当红包关键字出现时，点击对应的 View 对象；一种是使用 Xposed 框架对红包相关的函数进行 Hook，这种方案需要 Root 权限，但是不必打开微信界面即可抢红包。此应用抢红包也使用 Hook 红包相关函数的方式，但是不需要 Root。

**1. 注入代码**

VA 实现了插件化的注入模块，其中一个注入模块为 FixBug_AppInstrumentation，该模块替换了 ActivityThread 的 mInstrumentation 对象：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018826.jpeg)



mInstrumentation 对象会在应用 Application 类及 Activity 类创建时被执行相应的回调，该应用了修改了其中一个回调 callApplicationOnCreate，在 Application 执行了红包代码：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018093.jpeg)



其中 LuckyMoneyDispatcher 为红包功能模块。

函数 LuckyMoneyDispatcher.andFixForLuckMoney() 实现了方法替换：



![img](https://raw.githubusercontent.com/hhhaiai/Picture/main/img/202210271018374.jpeg)



使用开源热修补框架 AndFix 替换 com.tencent.mm.booter.notification.b.a() 为 LuckMoneyMethProxy.a()，并将被替换函数保存为 LuckMoneyMethProxy.aOriginal()。

**2. 模拟点击红包消息**

LuckMoneyMethProxy.a() 为替换后的函数，当微信接收到消息时被调用。



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/tlva9lt3lw.jpeg)



函数先判断消息类型，当确定是红包（436207665）后，解析消息，构造 Intent 并发送。这一步模拟了点击红包消息时的弹窗。

**3. 模拟拆开红包**

上一步的弹窗是一个 Activity，当弹出时（对应 Activity 的 onResume），mInstrumentation 将被回调：



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/nejiqkbapo.jpeg)



onLuckyMoneyResume 根据版本号确定要反射调用的 “拆开红包按钮”（包括 BUTTON_OPEN、OBJECT_OPEN、METHOD_OPEN）



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/t0lanq47rm.jpeg)



最终由 MonitorHandler 反射调用拆开红包函数：



![img](https://ask.qcloudimg.com/http-save/yehe-1268449/oen3hv3y4b.jpeg)



## **四、 总结**

VirtualApp 作为开源的多开应用框架，可以被任何人使用。它在 Android 系统和被多开应用间增加了中间层。这带来了两方面问题，一方面，VA 可掩盖应用的静态特征（包名、证书、资源文件、代码等），使得单纯的静态检测方法失效，应用具有了一定免杀的能力。同一个恶意应用可以有众多 VA 母包，且母包不包含恶意特征，这给检测引擎识别恶意应用带来了难度。安全云通过动态检测在 VA 母包运行时动态提取 VA 应用中的子包，并结合子包的恶意情况对母包的恶意情况进行综合判定，可有效对恶意应用的 VA 母包进行标记查杀。

另一方面，由于多开应用运行在 VA 中，VA 对被多开应用具有不弱于 Root 的权限，可方便有效介入应用运行流程。例如：当应用运行时展示广告，对多开应用进行截屏、模拟点击。更进一步的，VA 可通过 Hook 修改应用的执行流程，获得应用的隐私数据，包括但不限于密码、与服务器的数据通信、照片等。应用应当对运行在 VA 或其他多开应用内的带来的安全风险有所了解并加以防范，特别是金融、通讯类应用。

安全云己对相关 VA 应用进行监测，并及时对新型安全威胁作出响应。
