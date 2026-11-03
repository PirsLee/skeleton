# main

1. 初始化默认标志位

```
常规日志：                  loggingEnabled
调优模式：                  tuningEnabled
调试模式：                  debugEnabled
核心PID计算结结果：         coreLoggingEnabled
```

2. 解析命令行参数

使用CLI：App解析用户传入的命令行选项

```
指定JSON配置文件的路径（必须是已经存在的路径）          --conf
指定日志输出目录（必须是已经存在的目录）                --log
启用调优模式                                        --tuning
启用调试模式                                        --debug
启用核心PID计算日志                                 --core
```

3. 检查运行时标记文件

除了命令行参数，程序还会检查/etc/thermal.d/下的特殊文件

```
如果对应的文件存在那么就开启相应的模式
```

4. 初始化D-Bus总线并注册服务

程序创建了四个独立的connection对象，对应不同的总线角色

```
modeControlBus          连接到系统总线，申请服务名：xyz.openbmc_project.State.FanCtrl，在该总线/xyz/openbmc_project/settings/fanctrl路径下注册一个object manager对象管理器
hostBus                 连接到系统总线，申请服务名：xyz.openbmc_project.Hwmon.external，用于与主机侧的硬件监控hwmon进行通信
passiveBus              连接到系统总线，用于被动传感器数据订阅
hostMatchBus            连接到系统总线，用于监听主机状态变化
```

5. 注册信号处理器

通过`boost::asio::signal_set`注册SIGHUP和SIGTERM的异步监听，并关联共享回调函数`signalHandler`

```
SIGHUP：触发配置重加载流程tryRestartControlLoops，但是会先等待1秒以防止抖动，然后重新读取配置并重建控制循环
SIGTERM：触发优雅关闭流程tryTerminateControlLoops，先停止所有PID控制循环，然后退出事件循环
```

6. 启动PID控制循环

调用`pid_control::tryRestartControlLoops`，这是整个程序的核心启动步骤

```
如果是第一次调用，会立即通过boost::asio::post异步执行restartControlLoops
restartControlLoops：
    1.先调用stopControlLoops停止所有正在运行的定时器和控制区
    2.查找配置文件：优先使用命令行指定的-c路径，如果没有指定，则依次搜索当前目录/var/lib/swampd、/usr/share/swampd/下的config.json
    3.如果找到配置文件，则解析JSON并构建传感器配置sensorConfig、PID配置zoneConfig和区域配置zoneDetailsCOnfig
    4.如果找不到文件，则厂是通过DBus动态配置dbus_configuration::init，如果尚未就绪则直接返回
    5.配置就绪后，调用buildSensors创建传感器管理器state::mgmr
    6.调用buildZones为每个区域创建PID控制器（ZoneInterface对象）
    7.调用buildFailsafeLoggers为每个区域添加限速日志，每秒最多20条
    8.为每个区域创建一个boost::asio::steady_timer，并启动pidControlLoop回调：这是一个周期性执行的函数，负责读取传感器、计算PID输出、调节风扇转速
如果restartControlLoops抛出异常，则tryRestartControlLoops会捕获异常并安排10秒后重试，直到成功为止
```

7. 启动主机状态监控

通过单例HostStateMonitor::getInstance(hostMatchBus)获取主机状态监测器实例，并调用startMonitoring开始监听主机电源状态变化。当主机进入休眠或关机时，风扇控制逻辑可能会相应调整。

8. 进入事件循环

最后调用io.run进入Boost.Asio事件循环。此后程序的所有的活动都由异步事件驱动

- 定时器到期触发PID计算

- DBus消息到达触发传感器更新或配置变更

- 信号到来触发重载或退出

io.run()之前将PID循环注册进入事件循环队列，io.run之后才开始周期性运行。

# buildXYZ

## buildSensorsFromJson

输出的结果是

```
config = {
  "fan1":      SensorConfig{ type="fan",  readPath=".../fan1", writePath=".../pwm1", min=0, max=255 },
  "fan2":      SensorConfig{ type="fan",  readPath=".../fan2", writePath=".../pwm2", min=0, max=255 },
  "fan3":      SensorConfig{ type="fan",  readPath=".../fan3", writePath=".../pwm3", min=0, max=255 },
  "fan4":      SensorConfig{ type="fan",  readPath=".../fan4", writePath=".../pwm4", min=0, max=255 },
  "fan5":      SensorConfig{ type="fan",  readPath=".../fan5" },   // 无 writePath / min / max
  "fan6":      SensorConfig{ type="fan",  readPath=".../fan6" },
  "fan7":      SensorConfig{ type="fan",  readPath=".../fan7" },
  "fan8":      SensorConfig{ type="fan",  readPath=".../fan8" },
  "fleeting0": SensorConfig{ type="margin", readPath=".../margin/fleeting0", timeout=4 }
}

conf中的SensorConfig的样子
conf::SensorConfig {
    .type = "fan",
    .readPath = "/xyz/openbmc_project/sensors/fan_tach/fan1",
    .writePath = "/sys/devices/platform/ahb/ahb:apb/1e786000.pwm-tacho-controller/hwmon/**/pwm1",
    .min = 0,
    .max = 255,
    .timeout = 0,
    .ignoreDbusMinMax = false,
    .unavailableAsFailed = true,
    .ignoreFailIfHostOff = false
}
```

注意这里没有显式调用conf::from_json函数，是因为nlohmann使用ADL规则使编译器自动在conf命名空间找到候选函数，其实是使用了的。

## buildPIDsFromJson

执行的字段为JSON中的`zones`，zones字段中的pidConfig和zonesConfig共享一个id，结果如下：

**pidConfig = std::map\<int64_t, conf::PIDConf\>**

```
"fan1-5"
ControllerInfo {
    .type = "fan",
    .inputs = {
        SensorInput{.name = "fan1", .convertMarginZero = NaN, .convertTempToMargin = false, .missingIsAcceptable = false},
        SensorInput{.name = "fan5", .convertMarginZero = NaN, .convertTempToMargin = false, .missingIsAcceptable = false}
    },
    .setpoint = 90.0,
    .pidInfo = {
        .samplePeriod = 0.1,
        .proportionalCoeff = 0.0,
        .integralCoeff = 0.0,
        .feedFwdOffsetCoeff = 0.0,
        .feedFwdGainCoeff = 0.01,
        .integralLimit_min = 0.0,
        .integralLimit_max = 0.0,
        .outLim_min = 30.0,
        .outLim_max = 100.0,
        .slewNeg = 0.0,
        .slewPos = 0.0
    },
    .stepwiseInfo = {},          // 未提供，默认空
    .failSafePercent = 0.0       // 未提供，默认 0
}

"fan2-6"
ControllerInfo{
    .type = "fan",
    .inputs = {
        SensorInput{.name = "fan2", ...},
        SensorInput{.name = "fan6", ...}
    },
    .setpoint = 90.0,
    .pidInfo = { /* 同 fan1-5 的 pidInfo */ },
    .stepwiseInfo = {},
    .failSafePercent = 0.0
}

"fan3-7"
ControllerInfo{
    .type = "fan",
    .inputs = {
        SensorInput{.name = "fan3", ...},
        SensorInput{.name = "fan7", ...}
    },
    .setpoint = 90.0,
    .pidInfo = { /* 同上 */ },
    .stepwiseInfo = {},
    .failSafePercent = 0.0
}

"fan4-8"
ControllerInfo{
    .type = "fan",
    .inputs = {
        SensorInput{.name = "fan4", ...},
        SensorInput{.name = "fan8", ...}
    },
    .setpoint = 90.0,
    .pidInfo = { /* 同上 */ },
    .stepwiseInfo = {},
    .failSafePercent = 0.0
}

"fleetingpid0"
ControllerInfo{
    .type = "margin",
    .inputs = {
        SensorInput{.name = "fleeting0", .convertMarginZero = NaN, .convertTempToMargin = false, .missingIsAcceptable = false}
    },
    .setpoint = 10.0,
    .pidInfo = {
        .samplePeriod = 1.0,
        .proportionalCoeff = 750.0,
        .integralCoeff = 16.0,
        .feedFwdOffsetCoeff = 0.0,
        .feedFwdGainCoeff = 0.0,
        .integralLimit_min = 3000.0,
        .integralLimit_max = 10000.0,
        .outLim_min = 3000.0,
        .outLim_max = 10000.0,
        .slewNeg = -100.0,
        .slewPos = 0.0
    },
    .stepwiseInfo = {},
    .failSafePercent = 0.0
}
```

**zoneConfig=std::map<int64_t, conf::ZoneConfig>**

```
ZoneConfig{
    .minThermalOutput = 3000.0,
    .failsafePercent = 75.0,
    .cycleTime = {
        .cycleIntervalTimeMS = 100,   // 默认值（JSON 未指定）
        .updateThermalsTimeMS = 1000  // 默认值（JSON 未指定）
    },
    .accumulateSetPoint = false       // 默认值（JSON 未指定）
}
```

## buildSensors

返回结果：返回的SensorManager mgmr内部维护了两个重要成员：

- passiveListenBus 传入的passive总线

- hostSensorBus    传入的host总线

- 一个按照`type`分类的传感器容器

各个传感器具体创建的对象

**fan1~fan4**

```
配置特征：
    type = fan
    readPath = xxx
    writePath = yyy //可读可写
    min=0,max=255
创建过程：
    rtype = IOInterfaceType::DBUSPASSIVE
    wtype = IOInterfaceType::SYSFS
创建的接口：
    ri读接口：通过DbusPassive::createDbusPassive创建，类型为DbusPassive
    wi写接口：因为max>0所以创建SysFsWritePercent（将百分比映射到[min,max]范围后写入sysfs文件
最终传感器：
    auto sensor = std::make_unique<PluggableSensor>(
        name, 
        info->timeout,
        std::move(ri),
        std::move(wi),
        info->ignoreFailIfHostOff
    )
添加到mgmr：
    mgmr.addSensor("fan", name, std::move(sensor))
```

**fan5-fan8**

```
配置特征：
    type = fan
    readPath = xxx
    writePath = 无
    min=0,max=0
创建过程：
    rtype=IOInterfaceType::DBUSPASSIVE
    wtype：getWriteInterfaceType返回IOInterfaceType::UNKNOWN，创建为只读
创建的接口：
    ri读接口：同样是DbusPassive
    wi写接口：std::make_unique<ReadOnlyNoExcept>一个空的写接口，任何写操作都会被忽略
最终传感器：
    auto sensor = std::make_unique<PluggableSensor>(
        name, 
        info->timeout,
        std::move(ri),
        std::move(wi),
        info->ignoreFailIfHostOff
    )
添加到mgmr：
    mgmr.addSensor("fan", name, std::move(sensor))
```

**fleeting0**

与之同理

**最终SensorManager**

```
SensorManager mgmr(passiveBus, hostBus);
// 内部数据结构大致如下：
// type -> map<name, unique_ptr<Sensor>>
// 其中 Sensor 实际上是 PluggableSensor

mgmr.sensorsByType["fan"] = {
    {"fan1", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=SysFsWritePercent)},
    {"fan2", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=SysFsWritePercent)},
    {"fan3", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=SysFsWritePercent)},
    {"fan4", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=SysFsWritePercent)},
    {"fan5", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=ReadOnlyNoExcept)},
    {"fan6", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=ReadOnlyNoExcept)},
    {"fan7", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=ReadOnlyNoExcept)},
    {"fan8", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=ReadOnlyNoExcept)}
};

mgmr.sensorsByType["margin"] = {
    {"fleeting0", unique_ptr<PluggableSensor>(ri=DbusPassive, wi=ReadOnlyNoExcept)}
};
```

## buildZones

返回对象：

    std::unordered_map<int64_t, std::shared_ptr<ZoneInterface>> zones = {
        { 1, shared_ptr<DbusPidZone>{} }
    };

