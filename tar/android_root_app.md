# android root下管理app报告  
## autoedge的安卓端client存在形式
在探究解决方案的时候主要思路是**对比Nomad的client端功能，结合开源软件AnExplore源码**来进行。

需要满足的功能如下:
- 应用部署和卸载
- 应用监控
- 宿主机的状态监控
- 无交互
- 后台常驻
- 与云端交互
- 内容分发

首先需要明确client的存在形式，是以app，又或是其他形式。可以预见:
- 部署和监控的应用是以app形式运行在`ART(Android Run Time=Dalvik虚拟机+一些库)`上的沙箱环境中。
- 现有Android API或者root权限下可用的`shell命令`可以与这个沙箱环境通过`Activity Service Manager`等提供的接口与app交互
- 如果要以其他形式存在势必要去耦合`ART`提供给外界与app通信的接口，且例如**和Manager同级别的程序**需要修改android源码并编译刷入，难度较大

综上client的形式是以app的service形式后台常驻，使用`java`语言，需要的话辅以`JNI(Java Native Interface)`调用其他语言如`c++`(官方支持，其他语言不成熟)编写成的lib，结合root权限下`/system/bin`下的`shell命令`来完成上述功能

##  应用部署和卸载
此功能的实现采用**Android API**，安卓框架中负责包管理的部分为一个常驻后台的线程`PackageMangerService`,编程提供的接口主要在`android.content.pm`中，如:
```java
//查看已安装app的info
List<PackageInfo> allAppList = packageManager.getInstalledPackages(0)
//查看具体某个app的info
PackageInfo packageInfo = packageManager.getPackageInfo(packageName,0)
//通过intent调用系统内置安装软件安装app
Intent intent = new Intent();
intent.setAction(Intent.ACTION_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.setDataAndType(Uri.parse("file://"+ fileName)，"application/vnd.android.package-archive");
context.startActivity(intent);
//通过intent调用系统内置安装软件卸载app
Uri packageUri = Uri.fromParts("package", packageName, null);
Intent intentUninstall = new Intent(Intent.ACTION_DELETE, packageUri);
intentUninstall.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
context.startActivity(intentUninstall);
```
上述方法有一个缺点即**需要点击确认这个交互动作**，为满足无交互即静默安装与卸载，可以使用`shell命令pm`:
```java
//应用获取root权限
Process process = Runtime.getRuntime().exec("su");
//调用pm命令安装
DataOutputStream os = new DataOutputStream(process.getOutputStream());
os.writeBytes("pm install -r /sdcard/autoedge.apk \n");
os.writeBytes("exit\n");
//调用pm命令卸载
DataOutputStream os = new DataOutputStream(process.getOutputStream());
os.writeBytes("pm uninstall -r com.HarmonyCloud.autoedge \n");
os.writeBytes("exit\n");
```
## 应用监控
此功能的实现采用**Android API**，安卓中负责应用管理的部分为一个常驻后台的线程`ActivityMangerService`，编程提供的接口主要在`android.app.ActivityManager`
```java
//通过intent唤醒应用，被唤醒的应用需要在manifest.xml中注册
Intent intent = new Intent("包名");
startActivity(intent);

//通过设置intent的componentName方式唤醒应用
Intent intent = new Intent();
ComponentName comp = new ComponentName("包名(应用id)"，"activity");
intent.setComponent(comp);
startActivity(intent);

//查看正在运行的app进程
ActivityManager am = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
List<ActivityManager.RunningAppProcessInfo> infoList = am.getRunningAppProcesses();

//杀掉后台进程，但api无法彻底杀死，可以被唤醒
am.killBackgroundProcess("packageName");
```
调用api的方法有诸多局限但规范，root权限下同样也有`shell命令am`和`shell命令ps`来管理应用
```java
//启动应用
am start -n com.HarmonyCloud.autoedge/com.HarmonyCloud.autoedge.MainActivity
//启动某个服务
am startservice -n com.HarmonyCloud.autoedge/com.HarmonyCloud.autoedge.OneService 
//查看正在运行的进程，和进程所消耗的资源，cpu、内存等
ps
//杀死和某个应用关联的所有可以杀死的进程
am kill 包名
//强制杀死某个应用
am force-stop 包名
```
## 宿主机状态监控
此功能的实现采用**Android API**，安卓中同样可以利用`android.app.ActivityManager`中的api查看内存占用，或者`android.os`中的api查看cpu信息
```java
//查看内存信息
ActivityManager am = (ActivityManager)this.getSystemService(Context.ACTIVITY_SERVICE);
ActivityManager.MemoryInfo memoryInfo = new ActivityManager.MemoryInfo();
am.getMemoryInfo(memoryInfo);
Log.v(TAG,"我的机器一共有:" + memoryInfo.totalMem + "内存");
Log.v(TAG,"其中可用的有:" + memoryInfo.availMem + "内存");
Log.v(TAG,"其中达到:" + memoryInfo.threshold + "就会有可能触发LMK(LowMemoryKiller)，系统开始杀进程了");
Log.v(TAG,"所以现在的状态是:" + memoryInfo.lowMemory );

//查看cpu信息，以变量形式储存
Build.BOARD // 主板   
Build.BRAND // android系统定制商   
Build.CPU_ABI // cpu指令集   
Build.DEVICE // 设备参数   
Build.DISPLAY // 显示屏参数   
Build.FINGERPRINT // 硬件名称   
Build.HOST  
Build.ID // 修订版本列表   
Build.MANUFACTURER // 硬件制造商   
Build.MODEL // 版本   
Build.PRODUCT // 手机制造商   
Build.TAGS // 描述build的标签   
```
同样root下有命令dumpsys查看各类信息
```
dumpsys activity //查询AMS服务相关信息
dumpsys window //查询WMS服务相关信息
dumpsys cpuinfo //查询CPU情况
dumpsys meminfo //查询内存情况
dumpsys diskstats //磁盘相关信息
```
## 无交互和后台常驻
- shell命令的背后实现
深入那些`shell命令`背后的源码，发现其是`shell脚本`再调用`java程序`，真正的源码是`java`语言，调用的库是安卓系统源码`ActivityManagerService`中的方法，且摒弃了其中的权限认证相关，又由于调用这个命令时切换到root用户，所以命令的运行不需要交互。这种方法可以作为后续开发中解决交互问题的备选。

- 伪装成系统app
通过root权限伪装成系统，系统应用的权限较大，且应用`oom_adj`(out of memory adjustment)优先级较高，不容易被杀掉   
## 内容分发
安卓中数据的存储方式主要为sqlite内置数据库和文件，若分发的内容为文件则直接`shell命令cp`拷贝到具体存储目录即可。

由于安卓中的数据库一般为应用私有，查看可以通过`shell cp`命令先将其他应用的数据库其复制到autoedge，再读取

若分发的内容为表单，可以将其他应用的数据库复制到autoedge,再通过`android.database`下的api操作数据库，增删改查，再将修改后的数据库文件放回。这种方法可能会遇到读写锁的问题，但是还未找到其他方法，安卓设计思想数据库就是私有的，
```java
SQLiteDatabase database = userDB.getWritableDatabase();
//使用sql语句插入数据
database.execSQL("insert into user (name, salary, phone)values(?, ?, ?)", new Object[]{"mary", 13000, "13838"});
database.execSQL("insert into user (name, salary, phone)values(?, ?, ?)", new Object[]{"jack", 14000, "13888"});
database.execSQL("insert into user (name, salary, phone)values(?, ?, ?)", new Object[]{"xiaoming", 14000, "13888"});
//使用api插入数据
ContentValues values = new ContentValues();
//往表中的列中添加数据
values.put("name", "xiaoming");//values.put(键，值);
values.put("salary", 16000);
values.put("phone", "15999");//这里的值可以不是字符串也可以，sqlite能把数据以字符串的形式存储进去
//修改
int i = database.update("user", values, "name = ?", new String[]{"小明"});
//删除
int i = database.delete("user", "name = ? and _id = ?", new String[]{"小明", "3"});
//查询
Cursor cursor = database.query("user", null, null, null, null, null, null, null);//返回指向满足条件的一条数据
```
## 与云端交互
上节中内容分发，最主要的一部分是如何从云端拉取需要分发的内容。以及client端与端间的交互，考虑到不仅仅是安卓的问题，该方面还未探究认证，待续。

## 总结和PS
总的来说，安卓官方库提供了丰富的api，但是具体的使用需要对这些api非常的熟悉,对一个需求有许多不同的解决方式。时间有限，就对一些常见问题的实现进行了探究。

另外在调研中尝试寻找框架以便于编程，但都和ui有关。

另外安卓的版本不同，高版本兼容低版本，但编程上也有些许不同，有些api在高本版中被弃用，在`AnExplore`中的解决方式是对于有出入的地方，先查询版本，再根据具体版本调用不同版本的api。

