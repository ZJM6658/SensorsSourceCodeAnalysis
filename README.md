### 神策数据（SensorsAnalyticsSDK）iOS SDK源码粗略分析

[TOC]

#### 一、注册SDK
1. serverURL为数据上报地址，当debugMode不为DebugOff时，会自动在url后面添加"/debug";
2. launchOptions为app启动参数，用于记录app是否为被动启动（远程通知启动，位置变动启动），当是被动启动时，本次启动后的b上报事件$app_state参数为background;
3. debugMode为调试模式,三种模式含义可见枚举值说明;
4. 设置一些公共属性（如应用名、渠道等）；
5. enableAutoTrack函数供使用者选择是否自动收集一些app生命周期事件；
6. addWebViewUserAgentSensorsDataFlags用于打通app与H5，在userAgent后加上 /sa-sdk-ios/sensors-verify/${host}?${project}供H5端判断在App内，host和projectc都从serverURL从获得;

#### 二、从打一个点开始
> 核心函数
```

 /**
* @param event             event的名称
* @param propertyDict  event的属性
* @param type               event的类型（此参数并未暴露给开发者，开发者打点事件类型均为“track”）
*/
 - (void)track:(NSString *)event withProperties:(NSDictionary *)propertieDict withType:(NSString *)type;

```

此方法做了哪些事情：
* 数据收集准备
1. 构建libProperties，此对象包含

```
{
"$lib":"iOS"; // 平台
"$lib_version":"1.10.17", //SDK版本
"$app_version":"xxx", //app版本
"$lib_method":"code",
"$lib_detail":lib_detail  // 事件触发的类和函数（$AppClick和$AppViewScreen事件的”$screen_name"）
}

```
2. 构建行为数据properties，此对象包含

> 下面这些Map中的值

- automaticProperties：App版本、网络、设备厂商、系统、系统版本、deviceModel、SDK信息、device_id、屏幕参数等，具体见函数`- (NSDictionary *)collectAutomaticProperties`;
- superProperties：通过registerSuperProperties设置的公共属性（比如注册SDK时候设置的appName）；
- dynamicSuperPropertiesDict：通过registerDynamicSuperProperties设置的动态公共属性(表示用户在使用过程中会动态变化的属性)(详见https://www.sensorsdata.cn/manual/super_properties.html);
- 传入参数propertieDict中除$project和$token外的其他所有参数；

> 其他添加属性


```

$network_type：网络类型；
$wifi：是否wifi；
event_duration：事件时长（可选，开发者进行统计时长的事件才有，默认AppEnd事件会统计使用时长）；
$is_first_day：是否首日访问
$app_state：被动启动时值为background
$screen_orientation：设备方向
$latitude，$longitude：经纬度

````

备注：
- 如果是打点事件`(type:track)`或注册事件`(type:track_signup)`，需要将各种公共属性按照优先级从低到高，依次是`automaticProperties, superProperties,dynamicSuperPropertiesDict,propertieDict`组装;
- 数据交付前为了避免传入参数`propertieDict`中有`$device_id`字段，防止用户故意修改修正，进行了纠正;

3.最终提交数据Map格式：

```

{
"event":"event",  // 事件名
"properties":p, //事件核心参数都在这里
"distinct_id":"xxx",  // 唯一id，未设置前按照优先级IDFA>IDFV>UUID进行获取，当通过$sign_up事件设置之后发生改变
"original_id":"xxx", //$sign_up事件才会有（当调用$sign_up事件后更新了distinct_id，会将改变前的original_id带上）
"time":timeStamp, //事件时间
"type":"type",       //事件类型
"lib":libProperties, // lib规定的一些参数
"_track_id":arc4random(), //事件id，随机值
"project":"xxx",  //从传入参数propertieDict中取得（不设置则没有）
"token":"token", //从传入参数propertieDict中取得（不设置则没有）
}

```

* 数据库存储
1. 检查存储限制，默认本地最多存10000条，超过则删除前100条；
2. 通过JSONUtil将本条数据转换为Data后插入sqlite数据库;

* 上传服务器
1. 当`DebugMode != SensorsAnalyticsDebugOff`（非生产环境）或者事件类型为$sign_up事件时，马上上传服务器，其他情况则>=100条数据未上传时上传服务器；
2. 当App进入后台时也会将开启上传队列，同时通过`SQLite Vacuum`缩减表格文件空洞数据的空间
3. 上传数据处理：将数组元素转换为字符串以逗号分割，使用gzip进行压缩，base64Encode，上传格式`@"crc=%d&gzip=1&data_list=%@", hashCode, b64String`，并加上request需要补充的User-Agent、Cookie，Header字段，队列中发起上传，使用信号量同步等待上传结果；
4. 如果当前网络符合上传条件，`DebugMode != SensorsAnalyticsDebugOff`（非生产环境）是实时上传（1条为单位），所以是，生产环境50条一组上传，上传完为止，上传成功即删除已上传的统计数据；

#### 三、用户对象：SensorsAnalyticsPeople
用户属性的增删改也是通过打点事件来向服务端传递，没有event字段，只有属性参数和事件类型，且事件类型穷举如下：

- "profile_set"   //设置用户的属性的内容
- "profile_set_once"  //首次设置用户的属性的内容
- "profile_unset"     //删除某个属性的全部内容
- "profile_increment" //给一个数值类型的属性增加一个数值（如付费次数）
- "profile_append"    //向一个 NSSet 或者 NSArray 类型的 value 添加一些值
- "profile_delete" //删除当前这个用户的所有记录

```

#### 四、远程控制：SASDKRemoteConfig
```
 - (void)requestFunctionalManagermentConfigWithCompletion:(void(^)(BOOL success, NSDictionary*configDict )) completion;
```
当App进入后台时会请求`${serverURL}/config/iOS.conf?v=${version}` 版本参数`v`可选配置，更新远程配置后存储在本地，用户没有配置远程控制选项，服务端默认返回{"disableSDK":false,"disableDebugMode":false}；
> 可以远程开关SDK统计、关闭调试模式、修改收集自动统计App生命周期事件的类型；

#### 五、SDK文件一览
command: "tree -I "*.h""

```
.
├── AutoTrackUtils              `自动统计辅助工具，可以给页面加属性备注，默认拦截TableView、CollectionView的itemSelect事件`
├── JSONUtil                    `将对象转换为Data`
├── MessageQueueBySqlite        `sqlite操作消息队列管理`
├── NSInvocation+SAHelpers      `对象消息操作扩展`
├── NSString+HashCode           `hash值计算`
├── NSThread+SAHelpers          `线程扩展`
├── SAAbstractDesignerMessage
├── SAAbstractHeatMapMessage
├── SAAppExtensionDataManager
├── SAApplicationStateSerializer
├── SAClassDescription
├── SACommonUtility
├── SADeviceOrientationManager
├── SAEnumDescription
├── SAGzipUtility
├── SAHeatMapConnection
├── SAHeatMapSnapshotMessage
├── SAKeyChainItemWrapper
├── SALocationManager
├── SALogger
├── SAObjectIdentityProvider
├── SAObjectSelector
├── SAObjectSerializer
├── SAObjectSerializerConfig
├── SAObjectSerializerContext
├── SAPropertyDescription
├── SAReachability
├── SASDKRemoteConfig   `远程控制SDK、DebugMode、autoTrackMode`
├── SAServerUrl         `将传入的severUrl进行分解获取想要的参数等`
├── SASwizzle
├── SASwizzler
├── SATypeDescription
├── SAValueTransformers
├── SensorsAnalyticsExceptionHandler
├── SensorsAnalyticsSDK.bundle
│   ├── sa_autotrack_viewcontroller_blacklist.json
│   ├── sa_headmap_path.json
│   └── sa_mcc_mnc_mini.json
├── SensorsAnalyticsSDK
├── UIApplication+AutoTrack
├── UIView+AutoTrack
├── UIView+SAHelpers
└── UIViewController+AutoTrack
```

iOS开发者先看看官方使用说明：https://www.sensorsdata.cn/manual/ios_sdk.html

