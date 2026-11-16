# EntityManager业务详解：一块网卡插进去，BMC 怎么知道它上面有温度传感器

> 本文讲的不是"entity-manager 是什么"，而是"一块板子插到机器上之后，BMC 里到底发生了什么，最后谁把温度读数显示出来"。
>
> 仓库路径：`E:/openbmc/entity-manager`（下文所有 `路径:行号` 都相对这个根目录，行号来自当前检出的源码）。
>
> 读者背景：写过 Java/Spring、多年没碰 C++。文中会用 Spring 的东西做类比，但**结论全部回到代码本身**——每个技术论断后面都跟着 `相对路径/文件:行号`，你可以逐个点开核对。

---

## 0. 一句话说清业务

如果你用过 openUBMC，脑子里一定有"主板配置 / 硬件描述"这个概念：一份描述文件告诉 BMC 这块板上有哪些器件、它们在哪个总线地址、怎么读。entity-manager 干的就是这件事在 OpenBMC 里的落地版本，而且把它做成了一个**通用的翻译器**：

> **entity-manager 把 `configurations/` 下的 JSON 配置文件，翻译成 D-Bus 上的一棵对象树，告诉其他服务"这块板上有什么器件、怎么读"。它自己不读传感器、不挂驱动、不显示网页。**

仓库自己给的定义是："管理物理系统组件，并把它们映射到 BMC 内的软件资源"（`README.md:3-6`）。它明确把"不直接参与设备探测、不直接参与设备管理"列为范围外（`README.md:118-126`）——探测交给别的守护进程，管理交给别的守护进程，它只负责翻译和发布。

用 Java 的话说：**entity-manager 就是一个按配置实例化 Bean 的 IoC 容器**，只不过它产出的"Bean"不是 Java 对象，而是 D-Bus 对象。

- `configurations/*.json` ≈ Spring 的 `@Configuration` + `@Bean` 声明：写清楚有什么、在哪、怎么连；
- `Probe` 字段 ≈ `@ConditionalOnProperty` / `@ConditionalOnBean`：条件成立才把这个 Bean 注册进容器；
- D-Bus 对象路径 ≈ 一个按名字索引的注册表（可以类比 JNDI 目录）：`/xyz/openbmc_project/inventory/system/board/AMD_AMC`；
- 消费方（dbus-sensors、bmcweb）≈ 按类型查 Bean 的调用方：`getBeansOfType(Configuration.TMP441.class)`。

这套设计要解决的问题在 README 里写得很直白：让移植到新系统更快、减少平台间不同的代码、做到跨平台可维护（`README.md:33-41`）。换句话说，**同一份传感器代码，靠换 JSON 就能适配不同硬件**。

---

## 1. 真实场景一：新主板插上电，BMC 怎么知道有哪些传感器

我们用一个仓库里真实存在的配置来讲，不用虚构例子。`configurations/amd/mi455x_amc.json:1-26` 描述一块 AMD Instinct MI455X 的 AMC 卡：

```json
{
    "Exposes": [
        {
            "Address": "$address",
            "Bus": "$bus",
            "Name": "AMD AMC FRU",
            "Type": "EEPROM"
        }
    ],
    "Name": "AMD AMC",
    "Probe": "xyz.openbmc_project.FruDevice({'BOARD_PRODUCT_NAME': 'AMD Instinct MI455X AMC', 'BOARD_MANUFACTURER': 'AMD'})",
    "Type": "Board",
    "xyz.openbmc_project.Inventory.Decorator.Asset": {
        "BuildDate": "$BOARD_MANUFACTURE_DATE",
        "Manufacturer": "$BOARD_MANUFACTURER",
        "Model": "$BOARD_PRODUCT_NAME",
        "PartNumber": "$BOARD_PART_NUMBER",
        "SerialNumber": "$BOARD_SERIAL_NUMBER"
    }
}
```

（`configurations/amd/mi455x_amc.json:1-26`）

这段配置用 Java 的话翻译一下：

- `Name`/`Type`：这个 Bean 叫什么、属于哪一类（Board）；
- `Probe`：**创建条件**——"当 D-Bus 上出现 `xyz.openbmc_project.FruDevice` 接口，且它的 `BOARD_PRODUCT_NAME` 匹配 `AMD Instinct MI455X AMC`、`BOARD_MANUFACTURER` 匹配 `AMD` 时，条件成立"；
- `Exposes`：条件成立后要注册的东西——一个叫 `AMD AMC FRU` 的 EEPROM；
- 最后那个 `xyz.openbmc_project.Inventory.Decorator.Asset`：额外挂上去的资产信息接口，值里的 `$BOARD_MANUFACTURER` 是占位符，条件匹配时会被真实值替换。

`$bus`、`$address`、`$BOARD_MANUFACTURER` 这套占位符，就是 Spring 里的 `${...}`；替换动作在 `src/entity_manager/perform_scan.cpp:516-537`（`replaceTemplateFields` 处理 Name/Type/Probe/Exposes）和 `src/entity_manager/perform_scan.cpp:465-478`（处理每个 Exposes 里的属性）完成。

现在看整条链路。

### 1.1 第 1 步：探测进程先把"硬件事实"摆到 D-Bus 上

entity-manager 自己不做探测（`README.md:120-122`）。它依赖的探测进程有几种：

- **fru-device**：扫所有 I2C 总线，找 FRU EEPROM，解析后发到 D-Bus（`README.md:47-50`）；
- **devicetree-vpd-parser**：解析设备树里的 VPD，进程启动后申请 `xyz.openbmc_project.MachineContext` 这个名字（`src/devicetree_vpd_parser/devicetree_vpd_parser.cpp:5-8,25-38`）；
- **gpio-presence**：检测风扇板、挡风板、线缆这类没有标准探测方式的器件，`src/gpio-presence/README.md:84-91` 写了它的用途；它靠 `xyz.openbmc_project.Inventory.Source.DevicePresence` 接口上报（`src/gpio-presence/README.md:79-82`）。

以 fru-device 为例。进程启动时先跑一次全总线扫描（`src/fru_device/fru_device.cpp:1677-1678`），找到 FRU 后调用 `addFruObjectToDbus`：

```cpp
std::string productName =
    "/xyz/openbmc_project/FruDevice/" + optionalProductName.value();
...
std::shared_ptr<sdbusplus::asio::dbus_interface> iface =
    objServer.add_interface(productName, "xyz.openbmc_project.FruDevice");
```

（`src/fru_device/fru_device.cpp:898-910`）

于是 D-Bus 上出现一个 `/xyz/openbmc_project/FruDevice/Super_Great` 这样的对象，带 `xyz.openbmc_project.FruDevice` 接口，属性有 `BUS`、`ADDRESS`、`PRODUCT_PRODUCT_NAME`、`PRODUCT_MANUFACTURER` 等（真实 `busctl introspect` 输出见 `docs/my_first_sensors.md:86-113`）。

**注意这里的分工**：fru-device 只回答"我读到了一块 FRU，产品名是什么、在哪条总线、什么地址"，它完全不知道"AMD AMC"这个概念，也不知道上面有没有温度传感器。

### 1.2 第 2 步：entity-manager 启动时把配置读进内存

entity-manager 的 `main()` 很短，核心就三件事（`src/entity_manager/main.cpp:12-30`）：

```cpp
const std::vector<std::filesystem::path> configurationDirectories = {
    PACKAGE_DIR "configurations", SYSCONF_DIR "configurations"};
const std::filesystem::path schemaDirectory(PACKAGE_DIR "schemas");
...
systemBus->request_name(emDbusName);
EntityManager em(systemBus, io, configurationDirectories, schemaDirectory);
...
em.handleCurrentConfigurationJson();
io.run();
```

（`src/entity_manager/main.cpp:14-28`）

- 配置来自两个目录：打包进镜像的 `configurations/`，以及可覆盖它的系统配置目录（`src/entity_manager/main.cpp:14-15`）；
- schema 目录来自 `schemas/`（`src/entity_manager/main.cpp:17`）；
- 申请的总线名是 `xyz.openbmc_project.EntityManager`（常量定义在 `src/entity_manager/utils.hpp:13-14`）。

`EntityManager` 构造函数里创建 `Configuration` 对象（`src/entity_manager/entity_manager.cpp:52-65`），而 `Configuration` 的构造函数立刻干两件事：加载所有配置文件、抽取 Probe 里要用到的 D-Bus 接口名（`src/entity_manager/configuration.cpp:20-28`）。

加载过程：

1. `findFiles(configurationDirectories, R"(.*\.json)", jsonPaths)` 递归扫出所有 `.json`（`src/entity_manager/configuration.cpp:79`，`findFiles` 的实现在 `src/utils.cpp:44-70`）；
2. 对每个文件调 `loadSingleConfigFile`（`src/entity_manager/configuration.cpp:111-114`）；
3. 先解析 JSON，语法错就记日志跳过（`src/entity_manager/configuration.cpp:33-44`）；
4. 打开运行时校验时才用 valijson 校验 schema（`src/entity_manager/configuration.cpp:46-50`，校验函数本体在 `src/entity_manager/configuration.cpp:146-156`）；
5. 一个文件可以是单个对象，也可以是数组；每一项交给 `EMConfig::fromJson`，成功就塞进 `configurations` 这个 vector（`src/entity_manager/configuration.cpp:52-70`）。

`EMConfig` 就是把一份 JSON 拆成强类型字段的载体：`name`、`type`、`probeStmt`、`exposesRecords`、`extraInterfaces`（`src/entity_manager/em_config.hpp:20-29`）。它的构造函数强制要求有 `Name`、`Type`、`Probe`、`Exposes`，少一个就抛异常（`src/entity_manager/em_config.cpp:190-219`）；以 `xyz.openbmc_project.` 开头的顶层键会被收进 `extraInterfaces`（`src/entity_manager/em_config.cpp:167-188`）——这就是上面那份 AMD 配置里 `Asset` 接口的去处。

> 到这里你可以把它想成：Spring 容器启动时扫描配置文件，把每一份 JSON 变成一个 `EMConfig` BeanDefinition。

### 1.3 第 3 步：D-Bus 信号把"硬件事实"通知给 entity-manager

entity-manager 怎么知道 fru-device 发布了新对象？靠 D-Bus 的 match 规则。构造函数末尾调用 `initFilters`（`src/entity_manager/entity_manager.cpp:79`），它注册了三类关注点（`src/entity_manager/entity_manager.cpp:730-775`）：

- `NameOwnerChanged`：某个 well-known name 的属主变了（有进程起来或退出）。注释解释了原因：有些静态 provider 先创建对象再申请名字，不会发 PropertiesChanged，所以得靠这个补一刀（`src/entity_manager/entity_manager.cpp:725-747`）；
- `InterfacesAdded`：D-Bus 上新增了接口，只要新接口在 Probe 关心的高频接口集合里，就触发（`src/entity_manager/entity_manager.cpp:751-759`）；
- `InterfacesRemoved`：接口消失，同样触发，并清掉该路径上之前注册的 match 防止泄漏（`src/entity_manager/entity_manager.cpp:761-774`）。

另外，扫描过程中发现设备时，还会对那个路径单独注册 `PropertiesChanged` 监听（`src/entity_manager/perform_scan.cpp:102-105`，`src/entity_manager/entity_manager.cpp:706-723`）。

这三类信号最后都汇到同一个入口 `propertiesChangedCallback()`（`src/entity_manager/entity_manager.cpp:746,757,772`）。这个入口自己不干活，只做两件事：计数 + 起一个 500ms 的定时器（`src/entity_manager/entity_manager.cpp:619-631`）。注释说得很清楚：因为信号会"flooded"（蜂拥而至），所以要防抖（`src/entity_manager/entity_manager.cpp:628`）。

> Java 类比：这就是典型的事件驱动 + 防抖。别把它想成 entity-manager 在轮询硬件——它是被动等通知的。

### 1.4 第 4 步：PerformScan 拿"硬件事实"和"配置清单"做匹配

定时器到期后进 `propertiesChangedCallbackDebounced`（`src/entity_manager/entity_manager.cpp:559-617`）。这一段是整个业务的中枢，我们拆开看：

```cpp
nlohmann::json oldConfiguration = systemConfiguration;
auto missingConfigurations = std::make_shared<nlohmann::json>();
*missingConfigurations = systemConfiguration;

auto perfScan = std::make_shared<scan::PerformScan>(
    *this, *missingConfigurations, configuration.configurations, io, ...);
perfScan->run();
```

（`src/entity_manager/entity_manager.cpp:583-588,616`）

- `oldConfiguration`：改动前的整机配置快照；
- `missingConfigurations`：先复制一份全量配置，谁被匹配到了就从这份里删掉，剩下的就是"上次有、这次没了"的（`src/entity_manager/entity_manager.cpp:584-585`）；
- `configuration.configurations`：第 2 步加载出来的所有 `EMConfig`。

`PerformScan::run()` 分两段（`src/entity_manager/perform_scan.cpp:752-763`）：

1. `processConfigurations`：遍历每份配置，解析它的 Probe 语句，把里面提到的 D-Bus 接口收集起来（`src/entity_manager/perform_scan.cpp:714-750`）。Probe 语句先用空格拼成一行再交给词法分析器（`src/entity_manager/perform_scan.cpp:667-693`），解析出的 Token 类型定义在 `src/entity_manager/probe_lexer.hpp:16-25`：`TRUE`、`FALSE`、`AND`、`OR`、`MATCH_ONE`、`FOUND('name')`、`iface({...})`。这套语法的完整说明在 `docs/probe_grammar.md:20-44`；
2. `findDbusObjects`：拿收集到的接口名，通过 object mapper 的 `GetSubTree` 到 "/" 下查所有提供这些接口的对象（`src/entity_manager/perform_scan.cpp:198-200`），再对每个对象调 `org.freedesktop.DBus.Properties.GetAll` 把属性全拉回来（`src/entity_manager/perform_scan.cpp:90-94`），结果放进 `dbusProbeObjects`。

这里有两个细节值得记住：

- 扫描会跳过 entity-manager 自己（`busname != emDbusName`），避免自己的接口被自己探测导致循环（`src/entity_manager/perform_scan.cpp:109-113`）；
- 和 mapper 通信失败会重试 5 次（`src/entity_manager/perform_scan.cpp:46,48-54`），连 mapper 都联系不上则直接退出进程（`src/entity_manager/perform_scan.cpp:144-151`）。

### 1.5 第 5 步：匹配成功后，生成"运行时配置"

每个配置对应一个 `PerformProbe` 对象。它的析构函数里做真正的判断（`src/entity_manager/perform_probe.cpp:174-182`）：

```cpp
PerformProbe::~PerformProbe()
{
    scan::FoundDevices foundDevs;
    if (detail::doProbe(_probeCommand, scan->dbusProbeObjects,
                        scan->passedProbes, foundDevs))
    {
        scan->updateSystemConfiguration(recordRef, probeName, foundDevs);
    }
}
```

`doProbe` 就是 Probe 表达式的求值器（`src/entity_manager/perform_probe.cpp:69-161`）：

- `FOUND('x')`：查 `passedProbes`，看之前有没有配置已经匹配成功（`src/entity_manager/perform_probe.cpp:107-111`）；
- `iface({...})`：调 `probeDbus` 逐个 D-Bus 路径检查接口和属性（`src/entity_manager/perform_probe.cpp:19-65`）。字符串值按 `std::regex` 匹配，其他值按相等比较（`docs/probe_grammar.md:57-59`）；
- `AND`/`OR` **没有优先级，严格从左到右**（`docs/probe_grammar.md:63-65`），这点很重要，文档专门解释了为什么不能引入常规优先级（`docs/probe_grammar.md:70-88`）；
- 表达式为真但一个设备都没匹配到（比如 `Probe: "TRUE"`），会补一条空设备记录（`src/entity_manager/perform_probe.cpp:145-151`）；
- `MATCH_ONE` 只保留最后一条匹配设备（`src/entity_manager/perform_probe.cpp:152-159`）。

回到 AMD AMC 那份配置：它的 Probe 会在 fru-device 发布的对象的 `BOARD_PRODUCT_NAME`、`BOARD_MANUFACTURER` 属性上做正则匹配（`configurations/amd/mi455x_amc.json:11`），匹配成功后，`$bus`、`$address` 被换成探测到的真实总线的和地址，`$BOARD_MANUFACTURER` 被换成 FRU 里的厂商名。

匹配成功调 `updateSystemConfiguration`（`src/entity_manager/perform_scan.cpp:641-665`）。这里做几件事：

- 生成配置记录的 key。它不是直接用名字，而是 `probeName + 属性值` 的哈希（`src/entity_manager/perform_scan.cpp:203-225`），这样同名设备可以有多份实例；
- 处理 `$index` / 重名：重名时把模板里的某段换成设备序号并记日志（`src/entity_manager/perform_scan.cpp:438-464`）；
- 恢复上次已持久化的配置（"我已经报告过这个设备，别再当成新设备"），见 `restorePersistedConfigurations`（`src/entity_manager/perform_scan.cpp:480-514`）；
- 应用模板替换和 `Bind*`、`DisableNode` 这类 Expose 动作（`src/entity_manager/perform_scan.cpp:465-478,540-572`）；
- 最后把成型的记录写进 `systemConfiguration` 这份内存 JSON（`src/entity_manager/perform_scan.cpp:636-638`）。

> 这一步就是"运行时配置"的诞生：`EMConfig`（静态模板） + D-Bus 属性（硬件事实） → `systemConfiguration` 里的一条具体记录。Spring 里相当于 BeanDefinition + 环境变量 → 真正注册进容器里的 Bean。

### 1.6 第 6 步：翻译成 D-Bus 对象树

内存里的 JSON 还不是别的进程能读的东西，得发布到 D-Bus。扫描结束后回调进 `publishNewConfiguration`（`src/entity_manager/entity_manager.cpp:530-557`），它依次做三件事：

1. `loadOverlays(newConfiguration, io)`：按 `Exposes` 里的 `Type` 找 export 模板，把设备真正实例化到内核——也就是往 `/sys/bus/i2c/devices/i2c-N/new_device` 写东西（`src/entity_manager/overlay.cpp:115-130,206-263,298-323`）。AMD AMC 那份配置里的 `EEPROM` 就会走这条路（`configurations/amd/mi455x_amc.json:4-8`）；
2. `configCache.writeJsonFiles(systemConfiguration)`：把整机配置写到 `/var/configuration/system.json`（`src/entity_manager/config_cache.cpp:10-47`，路径定义在 `src/entity_manager/config_cache.hpp:14-18`）；
3. `postToDbus(newConfiguration)`：真正发布对象。

`postToDbus` 遍历每条配置，调 `postBoardToDBus`（`src/entity_manager/entity_manager.cpp:82-98`）。`postBoardToDBus` 是"JSON → 对象路径 + 接口"的核心（`src/entity_manager/entity_manager.cpp:147-284`）：

```cpp
const sdbusplus::object_path objectPath =
    em_utils::buildInventorySystemPath(configName, configType);
```

（`src/entity_manager/entity_manager.cpp:184-185`）

`buildInventorySystemPath` 的实现是：

```cpp
const sdbusplus::object_path basePath(
    "/xyz/openbmc_project/inventory/system");
boardName = dbus_util::sanitizeForDBusPathSegment(boardName);
return basePath / toLowerCopy(boardType) / boardName;
```

（`src/entity_manager/utils.cpp:430-439`）

所以 `Name: "AMD AMC"`、`Type: "Board"` 会生成 `/xyz/openbmc_project/inventory/system/board/AMD_AMC`（名字里的空格被替换掉）。然后：

- 创建 `xyz.openbmc_project.Inventory.Item` 接口（`src/entity_manager/entity_manager.cpp:187-191`）；
- 对于 Board、Chassis、Cpu、PowerSupply 等类型，再挂一个对应的 Inventory.Item 子接口，映射表是硬编码的（`src/entity_manager/entity_manager.cpp:196-214`）；
- 把 `Probe` 本身发布到 `xyz.openbmc_project.Configuration.Probe` 接口，属性有 `Name`、`Type`、`Probe`（`src/entity_manager/entity_manager.cpp:120-145`）——这是"这个对象是怎么来的"的自述；
- 顶层那些以 `xyz.openbmc_project.` 开头的键（比如 AMD 配置里的 `Asset`）也会被挂成接口（`src/entity_manager/entity_manager.cpp:227-245`）。

接着处理 `Exposes` 数组（`src/entity_manager/entity_manager.cpp:256-281`）。每一条 Exposes 交给 `postExposesRecordsToDBus`（`src/entity_manager/entity_manager.cpp:286-391`）：

```cpp
const std::string itemName = dbus_util::sanitizeForDBusPathSegment(
    findName->second.get<std::string>());
sdbusplus::object_path ifacePath = objectPath / itemName;
...
std::shared_ptr<sdbusplus::asio::dbus_interface> itemIface =
    dbus_interface.createInterface(
        ifacePath, "xyz.openbmc_project.Configuration." + itemType,
        configNameOrig);
```

（`src/entity_manager/entity_manager.cpp:324-327,375-378`）

关键约定：**Exposes 里的 `Type` 直接拼成接口名后缀**。仓库的 `my_first_sensors.md` 给了一个看得见摸得着的例子（那里的卡叫 `Great Card`）：

```text
path: /xyz/openbmc_project/inventory/system/board/18_Great_Card/18_great_local
xyz.openbmc_project.Configuration.TMP441 interface -         -               -
.Address                                 property  t         76               emits-change
.Bus                                     property  t         18               emits-change
.Name                                    property  s         "18 great local" emits-change
.Type                                    property  s         "TMP441"         emits-change
```

（`docs/my_first_sensors.md:189-209`）

注意 `Address` 从 JSON 里的字符串 `"0x4c"` 变成了 D-Bus 上的整数 `76`——中间有一步 schema 驱动的类型转换（`src/entity_manager/dbus_interface.cpp:263-303`，`convertSchemaTypes`），后面"机制讲解"那篇会展开。

一个额外的细节：每个由 Exposes 生成的对象还会挂一个 `xyz.openbmc_project.Configuration.Object` 接口，它没有属性，作用是"不管这条记录是什么类型，服务都能靠这个接口找到它"（`docs/entity_manager_dbus_api.md:98-118`，发布点在 `src/entity_manager/entity_manager.cpp:274-280`）。

### 1.7 第 7 步：dbus-sensors 把配置变成真正的传感器

到这里 entity-manager 的活就干完了。README 把这类消费方叫 **reactor**（反应器）："拿 entity-manager 的配置去执行、启用它描述的功能"，并点名 dbus-sensors 是最典型的一个（`README.md:75-81`）。

`docs/my_first_sensors.md:211-250` 完整展示了 dbus-sensors 的动作：

1. hwmon 温度守护进程找 `xyz.openbmc_project.Configuration.TMP441` 接口（`docs/my_first_sensors.md:211-213`）；
2. 按 `Bus`、`Address` 找到设备，看到有 hwmon 实例，把 `temp1_input` 映射到 `Name`、`temp2_input` 映射到 `Name1`（`docs/my_first_sensors.md:215-216`）；
3. 最终在 `/xyz/openbmc_project/sensors/temperature/18_great_local` 上发布 `xyz.openbmc_project.Sensor.Value`，属性 `Value` 是真正的温度读数（`docs/my_first_sensors.md:218-249`）。

整条链路画成一张图：

```text
 ┌──────────────┐   I2C 扫描, 读 FRU EEPROM
 │  fru-device  │ ─────────────────────────────┐
 └──────────────┘                              │ 发布 xyz.openbmc_project.FruDevice
                                               │ 对象 /FruDevice/Super_Great
                                               ▼
 ┌──────────────────────────────────────────────────────────┐
 │ D-Bus                                                    │
 │   InterfacesAdded / NameOwnerChanged / PropertiesChanged │  ← 触发信号
 └──────────────────────────────────────────────────────────┘
                                               │
                       ┌───────────────────────┘
                       ▼
 ┌──────────────────────────────────────────────────────────┐
 │ entity-manager                                           │
 │  加载 configurations/*.json  →  EMConfig                 │
 │  500ms 防抖  →  PerformScan  →  GetSubTree / GetAll      │
 │  Probe 求值  →  模板替换  →  systemConfiguration         │
 │  postBoardToDBus  →  postExposesRecordsToDBus            │
 └──────────────────────────────────────────────────────────┘
                                               │ 发布 Configuration.TMP441 等接口
                                               ▼
 ┌──────────────────────────────────────────────────────────┐
 │ dbus-sensors (hwmon 温度守护进程)                         │
 │  匹配 Configuration.TMP441 → 读 temp1_input              │
 │  发布 xyz.openbmc_project.Sensor.Value (Value = 31.938)  │
 └──────────────────────────────────────────────────────────┘
                                               │
                                               ▼
                              bmcweb / IPMI 读出这个温度
```

（链路各步的依据：探测 `src/fru_device/fru_device.cpp:884-910`；信号 `src/entity_manager/entity_manager.cpp:730-775`；防抖 `src/entity_manager/entity_manager.cpp:619-631`；扫描 `src/entity_manager/perform_scan.cpp:752-763`；匹配 `src/entity_manager/perform_probe.cpp:174-182`；发布 `src/entity_manager/entity_manager.cpp:530-557`；dbus-sensors `docs/my_first_sensors.md:211-250`）

---

## 2. 真实场景二：配置改动后如何"热更新"，以及 dbus-sensors 怎么消费

这一节要纠正一个常见的想当然：**entity-manager 并不会在运行时盯着 `configurations/` 目录看文件有没有改**。

### 2.1 配置文件什么时候生效

配置只在进程启动时读一次（`src/entity_manager/configuration.cpp:20-28`），而且 `Configuration` 对象构造完就再也不重新扫目录了——`loadConfigurations()` 只在构造函数里被调用（`src/entity_manager/configuration.cpp:20-28`）。整个 `src/entity_manager/` 目录里没有任何对配置目录的 inotify 或定时重扫。

对比一下：真正用 inotify 盯着文件系统的是 fru-device，它盯的是 `/sys/bus/i2c/devices`，I2C 总线增删时重新扫描（`src/fru_device/fru_device.cpp:1619-1620,1638-1664`）。盯的是硬件 sysfs，不是 JSON 配置目录。

所以：

- **改了 JSON 配置**：要重启 entity-manager 服务才会生效。systemd 单元文件里写了 `Restart=always`（`service_files/xyz.openbmc_project.EntityManager.service:10-16`），重启即可重新加载。
- **硬件热插拔**：不需要重启。D-Bus 上的 `InterfacesAdded`/`InterfacesRemoved`/`NameOwnerChanged` 信号会触发一次新的扫描（`src/entity_manager/entity_manager.cpp:730-775`）。

> Spring 类比：改配置文件要重启容器；但容器里的 Bean 可以按事件动态注册/注销。entity-manager 就是这种"配置启动加载 + 事件驱动更新"的混合体。

### 2.2 一次"重新扫描"里，谁增、谁删、谁不动

重新扫描并不把整棵树推倒重来，而是做差量。回到 `propertiesChangedCallbackDebounced`：

```cpp
for (const auto& [name, device] : missingConfigurations->items())
{
    pruneConfiguration(name, device);
}
nlohmann::json newConfiguration = systemConfiguration;
deriveNewConfiguration(oldConfiguration, newConfiguration);
```

（`src/entity_manager/entity_manager.cpp:592-599`）

- **删**：`missingConfigurations` 里剩下的，就是"上次匹配过、这次没匹配到"的设备，逐个 `pruneConfiguration`（`src/entity_manager/entity_manager.cpp:513-528`）：移除它的 D-Bus 接口、从 `systemConfiguration` 里擦掉、清理 topology，并发出 Inventory Removed 事件日志。
- **增**：`newConfiguration` 是当前全量，`deriveNewConfiguration` 把 `oldConfiguration` 里已有的键从里面删掉（`src/entity_manager/configuration.cpp:126-144`），剩下的就是本次新增。只有新增的才发 Inventory Added（`src/entity_manager/entity_manager.cpp:600-606`）。
- **不动**：旧的、仍然存在的配置保持原样，不重新发布。

README 把这条列为需求之一："支持多次探测，并且在发现新组件或组件消失时，只做最少的改动"（`README.md:107-110`）。代码里的 `cachedBaseline` 就是干这个的：它记下启动以来哪些 key 已经报告过，避免重复报"新增"（`src/entity_manager/entity_manager.hpp:45-48`，`src/entity_manager/entity_manager.cpp:658-704`）。

还有一条 10 秒延时的"收割"逻辑：扫描完不马上判定设备消失，而是起一个 10 秒定时器再检查（`src/entity_manager/entity_manager.cpp:476-511`）。这是为了避免扫描过程中的竞态——设备只是暂时没被看到，就被误判为拔掉了。

### 2.3 运行期改属性：从 D-Bus 写回 JSON

有一类"热更新"是运行时的：外部通过 D-Bus 的 `Set` 改配置属性。README 说得很清楚——entity-manager 发布的数据要能让别的守护进程"创建它描述的功能的实例"（`README.md:104-105`），而有些类型是允许读写的。

允许写的接口是白名单里的六个：`FanProfile`、`Pid`、`Pid.Zone`、`Stepwise`、`Thresholds`、`Polling`（`src/entity_manager/entity_manager.cpp:41-42`），`getPermission` 据此返回 `readWrite` 或 `readOnly`（`src/entity_manager/entity_manager.cpp:44-50`）。

写回链路在 `addProperty` 里（`src/entity_manager/dbus_interface.hpp:169-200`）：D-Bus 属性被 `Set` 时，回调先更新内存里的值，再用 `ConfigPointer` 定位到 JSON 里的位置写回去（`src/entity_manager/dbus_interface.hpp:187-192`），最后调 `configCache.writeJsonFiles` 落盘到 `system.json`（`src/entity_manager/dbus_interface.hpp:193-198`）。`ConfigPointer` 这个结构相当于一个手写的 JSON 指针，用 boardId + exposesIndex + propertyName + arrayIndex 定位（`src/entity_manager/config_pointer.hpp:9-50`），代码注释说明它是为了避免强制使用通用 json 指针数据结构（`src/entity_manager/config_pointer.hpp:9-10`）。

还有 `AddObject` / `Delete` 两个 D-Bus 方法，允许运行时往配置里加记录、删记录（`src/entity_manager/dbus_interface.cpp:120-163,402-428`）。

> 这又是一处 Spring 类比：`ConfigPointer` 像是一个带路径的 `PropertyPath`，`Set` 回调像是带 `@PostPersist` 的 setter——改内存的同时持久化。

### 2.4 dbus-sensors 是怎么消费的

dbus-sensors 不在本仓库里，但它怎么消费 entity-manager 的数据，在本仓库的文档和代码里都留了痕：

- **它靠接口类型找配置**：hwmon 温度守护进程找的是 `xyz.openbmc_project.Configuration.TMP441`（`docs/my_first_sensors.md:211-213`）。接口名就是 Exposes 的 `Type` 拼出来的（`src/entity_manager/entity_manager.cpp:375-378`）。
- **它靠属性知道去哪读硬件**：`Bus`、`Address`、`Name`、`Name1` 这些属性都在 Configuration 接口上（`docs/my_first_sensors.md:203-208`）。
- **它靠 `Configuration.Object` 做关联**：这个无属性接口让消费方不用关心记录的具体类型也能找到它（`docs/entity_manager_dbus_api.md:98-108`）。
- **设备实例化由 entity-manager 先做**：`Exposes` 里的 `Type` 若在 `devices::exportTemplates` 白名单里，entity-manager 会先往 sysfs 写 `new_device` 把内核驱动挂上（`src/entity_manager/overlay.cpp:265-296`），dbus-sensors 才能看到 hwmon 节点。

"Type 想在 D-Bus 上出现，光在 schema 里加不够"这一点在代码注释里写得很直白：顶层 Inventory 接口只给白名单里的类型（`src/entity_manager/entity_manager.cpp:196-199`）。而 Exposes 的类型若想触发内核设备创建，得在 `devices.hpp` 的 `exportTemplates` 里登记（`src/entity_manager/overlay.cpp:290-295`）。

### 2.5 顺带说一句：关联（Associations）

entity-manager 还会自动建立对象之间的关联，比如"哪个电源给哪块板供电"。配置里靠 `Name` 字段把两个端口连起来（`docs/associations.md:5-11`），代码里是 `Topology` 类：记录每个板子的类型和探测路径（`src/entity_manager/topology.hpp:41-56,95-100`），计算关联后用 `xyz.openbmc_project.Association.Definitions` 接口发布（`src/entity_manager/entity_manager.cpp:100-117`）。这是配置驱动思想的延伸——关联关系也是写 JSON，不是写死在 C++ 里。

---

## 3. 结尾：它和几个邻居的分工

把几个仓库放在一起看，边界就清楚了：

| 角色 | 谁来做 | 负责什么 | 依据 |
| --- | --- | --- | --- |
| 探测（事实来源） | fru-device、devicetree-vpd-parser、gpio-presence 等 | 发现硬件，把它读到的东西发到 D-Bus；不认识"传感器"概念 | `README.md:47-63`、`src/fru_device/fru_device.cpp:884-910`、`src/gpio-presence/README.md:79-91` |
| 翻译（本文主角） | entity-manager | 加载 JSON，用 Probe 匹配事实，把 JSON 翻译成 D-Bus 对象树 | `README.md:65-73`、`src/entity_manager/entity_manager.cpp:147-284` |
| 执行（reactor） | dbus-sensors | 读 Configuration 接口，建传感器、跑扫描循环 | `README.md:75-81`、`docs/my_first_sensors.md:211-250` |
| 呈现 | bmcweb / intel-ipmi-oem | 用 inventory 信息做 Redfish REST API / IPMI SDR、FRU 命令 | `README.md:128-133` |
| 接口字典 | phosphor-dbus-interfaces | 定义 D-Bus 接口长什么样（属性、类型） | `meson.build:34-37`、`src/entity_manager/entity_manager.cpp:200-214` |

一句话记住分工：

> **探测进程说"我看到了什么"，entity-manager 说"这块板按配置应该有什么、怎么读"，dbus-sensors 说"那我去读，并把读数放到 D-Bus 上"，bmcweb 说"我把读数变成网页/Redfish"。**

entity-manager 处在这条链的中间，自己既不碰硬件寄存器，也不画网页。它存在的意义，就是把"硬件差异"收敛到一份份 JSON 里（`README.md:65-73`、`CONFIG_FORMAT.md:3-7`），让上下游都能少写代码。

下一篇《EntityManager机制讲解》会钻进内部：一份 JSON 是怎么被逐层解析、匹配、转换类型、最后变成 D-Bus 接口上的属性的。
