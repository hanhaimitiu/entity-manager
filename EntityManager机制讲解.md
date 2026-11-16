# EntityManager机制讲解：一份 JSON 是怎么变成 D-Bus 接口的

> 业务背景（这块板子插上电之后发生了什么、谁负责什么）请看同目录的《EntityManager业务详解》。本文只钻一件事：**entity-manager 内部到底怎么转**。
>
> 仓库路径：`E:/openbmc/entity-manager`。下文所有 `路径:行号` 都相对这个根目录，行号按当前检出的源码逐条核对过。
>
> 读者背景：Java/Spring 出身。能用 Spring 类比的我会类比，但**代码的原话优先**——类名、字段名、接口名都按代码写，不生造译名。

---

## 0. 先讲一个真实例子：一份 JSON 走完最后一公里

我们挑仓库里真实存在的一份配置 `configurations/acbel/rica_psu.json`（26 行），它描述一款 ACBEL 电源：

```json
{
    "Exposes": [
        {
            "Address": "$address",
            "Bus": "$bus",
            "Name": "PSU$ADDRESS % 4 + 1 FRU",
            "Type": "EEPROM"
        },
        {
            "Address": "$ADDRESS % 4 + 88",
            "Bus": "$bus",
            "Name": "PSU$ADDRESS % 4 + 1",
            "PowerScaleFactor": 1000000,
            "Type": "pmbus"
        }
    ],
    "Name": "ACBEL RICA PSU$ADDRESS % 4 + 1",
    "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'R1CA2551B*'})",
    "Type": "PowerSupply",
    "xyz.openbmc_project.Inventory.Decorator.Asset": {
        "Manufacturer": "$PRODUCT_MANUFACTURER",
        "Model": "$PRODUCT_PRODUCT_NAME",
        "PartNumber": "$PRODUCT_PART_NUMBER",
        "SerialNumber": "$PRODUCT_SERIAL_NUMBER"
    }
}
```

（`configurations/acbel/rica_psu.json:1-26`）

把它从磁盘上的字节变成 D-Bus 上的属性，中间要过七道关。下面每一步都给出代码位置。

### 0.1 第一关：扫描目录，逐文件解析

`main()` 把两个配置目录和一个 schema 目录传给 `EntityManager`（`src/entity_manager/main.cpp:14-22`）。`EntityManager` 构造函数再转传给 `Configuration`（`src/entity_manager/entity_manager.cpp:52-65`），`Configuration` 的构造函数直接调用 `loadConfigurations()`（`src/entity_manager/configuration.cpp:20-28`）。

`loadConfigurations()` 用正则 `.*\.json` 递归找出所有配置文件（`src/entity_manager/configuration.cpp:73-79`）。`findFiles` 的实现里用了 `recursive_directory_iterator`，并且用文件名去重——两个目录里同名文件只留一个（`src/utils.cpp:44-70`）。这就是"系统配置目录可以覆盖打包配置"的实现原理。

找到文件后逐个 `loadSingleConfigFile`（`src/entity_manager/configuration.cpp:111-114`）：

```cpp
auto data = nlohmann::json::parse(jsonStream, nullptr, false, true);
if (data.is_discarded()) { /* 记日志并跳过 */ }
if (ENABLE_RUNTIME_VALIDATE_JSON && !validateJson(schema, data)) { /* 跳过 */ }
if (data.type() == nlohmann::json::value_t::array) {
    for (auto& d : data) { /* EMConfig::fromJson → configurations.emplace_back */ }
} else { /* 同样处理单个对象 */ }
```

（压缩自 `src/entity_manager/configuration.cpp:30-71`）

注意 `parse` 的第四个参数 `true` 允许忽略注释——所以配置文件里可以写 C 风格注释（`CONFIG_FORMAT.md:260`）。

### 0.2 第二关：变成 EMConfig

`EMConfig::fromJson` 先确认顶层是 object，失败返回 `nullopt`（`src/entity_manager/em_config.cpp:35-56`）。真正的解析在私有构造函数里（`src/entity_manager/em_config.cpp:190-219`）：

```cpp
result = getConfigProp<std::string>(config, "Name", name);
result = getConfigProp<std::string>(config, "Type", type);
if (!parseProbe(probeStmt, isArrayProbeStmt, config)) throw ...;
if (!parseExposes(exposesRecords, config)) throw ...;
if (!parseExtraInterfaces(extraInterfaces, config)) throw ...;
```

字段落到 `EMConfig` 的成员上（`src/entity_manager/em_config.hpp:20-29`）：

| 成员 | 类型 | 从哪来 |
| --- | --- | --- |
| `name` | `std::string` | 顶层 `Name` |
| `type` | `std::string` | 顶层 `Type` |
| `probeStmt` | `vector<string>` | 顶层 `Probe`，数组或单字符串都接受（`src/entity_manager/em_config.cpp:58-104`） |
| `exposesRecords` | `vector<json::object_t>` | 顶层 `Exposes`，每条必须有 `Name` 和 `Type`（`src/entity_manager/em_config.cpp:106-165`） |
| `extraInterfaces` | `flat_map<string, json::object_t>` | 顶层以 `xyz.openbmc_project.` 开头的键（`src/entity_manager/em_config.cpp:167-188`） |

我们这份电源配置的 `Asset` 就进了 `extraInterfaces`；两条 Exposes 进了 `exposesRecords`。

### 0.3 第三关：把 Probe 编成可求值的 token 流

配置文件解析完后，`Configuration` 还会做一件容易被忽略的事：`filterProbeInterfaces()`（`src/entity_manager/configuration.cpp:26-28,158-186`）。它把每份配置的 Probe 解析成 token，把其中 D-Bus 接口名收进 `probeInterfaces` 这个 set（`src/entity_manager/configuration.cpp:171-180`）。这个 set 后面用来过滤 D-Bus 信号——不关心的接口变化不触发扫描。

Probe 语法的词法分析器是 `probe::lexProbe`（声明在 `src/entity_manager/probe_lexer.hpp:46-49`），token 一共七种（`src/entity_manager/probe_lexer.hpp:16-25`）：

```cpp
enum class TokenType {
    boolTrue, boolFalse, opAnd, opOr, matchOne, found, dbusProbe,
};
```

`DbusProbe` 结构存接口名和属性匹配条件（`src/entity_manager/probe_lexer.hpp:27-33`）。完整语法（EBNF）在 `docs/probe_grammar.md:20-44`。

> Spring 类比：这一步相当于把 `@ConditionalOnProperty("...")` 的字符串表达式编译成一套可执行的 `Condition` 对象。

### 0.4 第四关：拿 token 去 D-Bus 上找对象

`PerformScan::run()` 先 `processConfigurations`，为每份配置生成一个 `PerformProbe`，并收集它关心的 D-Bus 接口（`src/entity_manager/perform_scan.cpp:714-763`）。收集逻辑在 `collectDbusProbes`，只挑 `token.dbus` 有值的（`src/entity_manager/perform_scan.cpp:698-712`）。

然后 `findDbusObjects` 调 object mapper：

```cpp
object_mapper::getSubTree(*scan->_em.systemBus, "/", 0, interfaces,
                          std::move(cb));
```

（`src/entity_manager/perform_scan.cpp:198-200`）

`GetSubTree` 返回的是"路径 → 服务名 → 接口列表"的树。对树上的每一项：

1. 对该路径注册 `PropertiesChanged` 监听（`src/entity_manager/perform_scan.cpp:102-105`）；
2. 跳过 entity-manager 自己，防止自激（`src/entity_manager/perform_scan.cpp:109-113`）；
3. 跳过 `org.freedesktop` 开头的三个默认接口（它们没有属性），其余调 `GetAll`（`src/entity_manager/perform_scan.cpp:116-124`），结果存进 `dbusProbeObjects[path][interface]`（`src/entity_manager/perform_scan.cpp:90-94`）。

`GetAll` 失败会重试 5 次，间隔 2 秒（`src/entity_manager/perform_scan.cpp:48-54,80-86`）；但如果错误是 `EBADR`，说明对象在 GetSubTree 和 GetAll 之间被删了，属于并发删除，直接跳过不重试（`src/entity_manager/perform_scan.cpp:62-74`）。这种细节很能说明这份代码是给真实热插拔场景写的。

### 0.5 第五关：求值、替换模板、生成运行时配置

`PerformProbe` 的析构函数做求值（`src/entity_manager/perform_probe.cpp:174-182`）——用析构函数当"回调"是这个实现的一个特别之处，因为 `shared_ptr` 引用计数归零时它自然触发，从而保证所有 `PerformProbe` 都完成探测后，`PerformScan` 的析构函数才继续（`src/entity_manager/perform_scan.cpp:765-780`）。

`doProbe` 的求值循环在 `src/entity_manager/perform_probe.cpp:69-161`。两个要点：

- `probeDbus` 遍历 `dbusProbeObjects`，对每个候选对象逐属性比对；字符串按正则，其他值按相等（`src/entity_manager/perform_probe.cpp:19-65`）。匹配上的对象连接口带路径存进 `foundDevs`（`src/entity_manager/perform_probe.cpp:56-62`）；
- `AND`/`OR` 严格左结合，没有优先级（`src/entity_manager/perform_probe.cpp:127-142`）。为什么不加优先级？`docs/probe_grammar.md:70-88` 给了答案：仓库里有一批 `A OR B AND FOUND('...')` 形式的 Probe，按左结合是 `(A OR B) AND FOUND(...)`，符合作者意图；加优先级会改变语义，而且语法没有分组符号，作者没法改写。

我们这份电源配置匹配上后，`$bus`、`$address`、`$ADDRESS % 4 + 1` 这些模板会被替换。替换逻辑分两层：

- 结构层：`replaceTemplateFields` 处理 `type`、`probeStmt`、`exposesRecords`（`src/entity_manager/perform_scan.cpp:516-538`）；
- 属性层：`applyTemplateAndExposeActions` 对每条 Exposes 的每个属性调 `em_utils::templateCharReplace`，并应用 `Bind*`/`DisableNode` 动作（`src/entity_manager/perform_scan.cpp:465-478`）。

`$ADDRESS % 4 + 1` 里带算术运算，这类表达式由 `src/entity_manager/expression.cpp` 处理（见 `src/entity_manager/utils.cpp:6` 的引用）；`$bus` / `$address` 这类简单替换的占位符常量是 `"$"`（`src/entity_manager/utils.cpp:17`）。

最终 `systemConfiguration[recordName] = record.toJson()`（`src/entity_manager/perform_scan.cpp:636-638`）。`recordName` 是哈希值（`src/entity_manager/perform_scan.cpp:203-225`），所以两条同型号电源会有两个不同的 key。

### 0.6 第六关：JSON → 对象路径 + 接口

`publishNewConfiguration` 依次做：加载 overlay（真正创建内核设备）、写配置缓存、发布 D-Bus（`src/entity_manager/entity_manager.cpp:530-557`）。

发布主流程：

```text
postToDbus                       src/entity_manager/entity_manager.cpp:82-118
 └─ postBoardToDBus              :147-284
     └─ postExposesRecordsToDBus :286-391
         ├─ postConfigurationRecord :393-462
         └─ createInterface/populateInterfaceFromJson（dbus_interface.cpp）
```

`postBoardToDBus` 里：

- `buildInventorySystemPath(name, type)` 生成路径基址：`/xyz/openbmc_project/inventory/system/<小写type>/<清洗后的name>`（`src/entity_manager/utils.cpp:430-439`）；
- `typeToInterface` 硬编码表决定顶层 Inventory 接口——`PowerSupply` → `xyz.openbmc_project.Inventory.Item.PowerSupply`（`src/entity_manager/entity_manager.cpp:200-214`）。注释提醒：光在 schema 里加类型不够，必须在这里登记，entity-manager 才会认领接口（`src/entity_manager/entity_manager.cpp:196-199`）；
- `Probe` 原文发到 `xyz.openbmc_project.Configuration.Probe`（`src/entity_manager/entity_manager.cpp:120-145,221-225`）；
- 顶层以 `xyz.openbmc_project.` 开头的键在这里变成接口，用 `populateInterfaceFromJson` 填属性（`src/entity_manager/entity_manager.cpp:227-245`）。

`postExposesRecordsToDBus` 里：

- `Name` 清洗后作为子路径段：`ifacePath = objectPath / itemName`（`src/entity_manager/entity_manager.cpp:324-327`）；
- `Type` 拼接成接口名：`"xyz.openbmc_project.Configuration." + itemType`（`src/entity_manager/entity_manager.cpp:375-378`）。所以 `Type: "pmbus"` 会得到 `xyz.openbmc_project.Configuration.pmbus`；
- 值类型不对就整条不发布：`convertSchemaTypes` 失败直接 `return nullopt`（`src/entity_manager/entity_manager.cpp:332-337`）；
- `Status: "disabled"` 的记录被跳过（`src/entity_manager/entity_manager.cpp:300-308`）。

Exposes 里如果还有嵌套的 object 或 array，由 `postConfigurationRecord` 处理：object 生成一个子接口，array 按索引生成多个带下标的接口（`src/entity_manager/entity_manager.cpp:393-462`），对应文档里的 `xyz.openbmc_project.Device.{Object}.{index}`（`docs/entity_manager_dbus_api.md:120-138`）。

### 0.7 第七关：属性落地，schema 决定 D-Bus 类型

最后一公里在 `populateInterfaceFromJson`（`src/entity_manager/dbus_interface.cpp:305-367`）。先看一眼它是怎么把 JSON 值变成 D-Bus 属性的：

```cpp
std::optional<SchemaTypedValues> dbusValues = convertSchemaTypes(dict);
...
for (const auto& [key, value] : dict) {
    if (schema 指定了类型) { addProperty(key, publishedValue, ...); continue; }
    // 否则按 JSON 自身类型推断：bool / int64 / uint64 / double / string
    populateInterfacePropertyFromJson(...);
}
```

（压缩自 `src/entity_manager/dbus_interface.cpp:312-361`）

`convertSchemaTypes` 是"schema 驱动类型"的关键（`src/entity_manager/dbus_interface.cpp:263-303`）：遍历记录的属性，用 `schemaDBusTypes.find(recordType, key)` 查 schema 里声明的 D-Bus 类型（`src/entity_manager/dbus_interface.cpp:282-289`），用 `toDBusValue` 转换（`src/entity_manager/dbus_interface.cpp:290-301`）。任何一个属性转不了，整个接口都不发布（`src/entity_manager/dbus_interface.cpp:293-299,312-322`）——这样消费方永远不会看到一个"缺属性或类型不对"的半成品接口。

`SchemaDBusTypes` 在启动时递归读取 `schemas/` 下所有 `.json`，从 `$defs` 里提取每个属性的类型（`src/entity_manager/schema_dbus_types.cpp:137-178,180-204`）。`$ref` 可以跨文件，最多跟 8 层以防止循环（`src/entity_manager/schema_dbus_types.cpp:17-19,29-58`）。

属性最终通过 `addValueToDBus` → `addProperty` 注册到 sdbusplus 的接口上（`src/entity_manager/dbus_interface.hpp:169-220`），类型分派在 `populateInterfacePropertyFromJson` 的 switch 里（`src/entity_manager/dbus_interface.cpp:212-261`）。到这里，我们那份电源配置的 `Bus`、`Address`、`PowerScaleFactor` 就真的出现在 D-Bus 上了。

### 0.8 用 Java 把整条链子翻译一遍

```text
Java/Spring 世界                entity-manager 世界
──────────────────────────────────────────────────────────────
application.yml / @Configuration → configurations/*.json
@Bean 方法                        → Exposes 数组里的一条条记录
@ConditionalOnProperty            → Probe 表达式
Environment 占位符 ${bus}          → 模板变量 $bus / $address
BeanDefinitionRegistry            → std::vector<EMConfig> configurations
BeanFactory 实例化 Bean           → postExposesRecordsToDBus 创建对象+接口
Bean 的属性绑定                    → populateInterfaceFromJson 注册 D-Bus 属性
按类型查 Bean getBeansOfType()    → dbus-sensors 按 Configuration.TMP441 找对象
事件驱动刷新 @RefreshScope         → InterfacesAdded 信号 + 500ms 防抖重扫
```

这个类比只到"像"为止。真正的差别是：Spring 的 Bean 还在 JVM 里，entity-manager 产出的对象是**跨进程可见**的——任何 D-Bus 客户端（bmcweb、IPMI、dbus-sensors）都能读。所以接口名和属性类型是硬约束，写错一个字母，消费方就找不到。

---

## 1. 先看构建关系

entity-manager 不是一个单一程序。`meson.build` 里能看出它由几个可执行程序、几个静态库、多少测试组成。

### 1.1 进程、库、测试总表

| 目标 | 类型 | 源文件要点 | 开关 / 出处 |
| --- | --- | --- | --- |
| `entity-manager` | 可执行程序 | `main.cpp` + `entity_manager_lib` | 总是构建，`src/entity_manager/meson.build:45-53` |
| `entity-manager` 静态库 | 静态库 | `entity_manager.cpp`、`configuration.cpp`、`perform_scan.cpp`、`perform_probe.cpp`、`dbus_interface.cpp` 等 18 个 cpp | `src/entity_manager/meson.build:21-43` |
| `utils` 静态库 | 静态库 | `src/utils.cpp`、`src/dbus_util.cpp` | `src/meson.build:5-10` |
| `fru-device` | 可执行程序 | `fru_device.cpp`、`fru_reader.cpp`、`fru_utils.cpp`、`gzip_utils.cpp`、`neard_dbus.cpp`、`nfc.cpp` | 选项 `fru-device`，`src/fru_device/meson.build:14-37` |
| `devicetree-vpd-parser` | 可执行程序 | `machine_context.cpp`、`devicetree_vpd_parser.cpp` | 选项 `devicetree-vpd`，`src/devicetree_vpd_parser/meson.build:2-10` |
| `gpio-presence-sensor` | 可执行程序 | `main.cpp` + `gpio_presence_lib` | 选项 `gpio-presence`，`src/gpio-presence/meson.build:18-31` |
| `gpio_presence_lib` | 静态库 | `device_presence.cpp`、`gpio_presence_manager.cpp`、`config_provider.cpp` | `src/gpio-presence/meson.build:1-16` |
| 测试 | 可执行程序 × 10+ | `test_fru_utils`、`test_topology`、`test_gpio_presence`、`test_utils`、`test_dbus_util`、`test_gpio_presence_state` 等 | `test/meson.build:21-101`，entity-manager 内部测试见 `test/entity_manager/meson.build` |

顶层 `meson.build` 用 `subdir()` 把各目录串起来：`configurations` → `schemas` → `service_files` → `src` → `test`（`meson.build:81-126`）。配置 JSON 和 schema 是作为数据安装的（`meson.build:88-119`），不是编译进二进制——这点呼应了"配置驱动"。

### 1.2 几个值得注意的编译开关

| 选项 | 默认值 | 作用 | 出处 |
| --- | --- | --- | --- |
| `validate-json` | `true` | 构建时用 `scripts/validate_configs.py` 校验所有配置 | `meson.options:32-37`、`meson.build:100-109` |
| `runtime-validate-json` | `false` | 运行时校验 JSON（映射为 `ENABLE_RUNTIME_VALIDATE_JSON`） | `meson.options:38-43`、`src/entity_manager/meson.build:3-7` |
| `new-device-detection` | `true` | 映射为 `EM_CACHE_CONFIGURATION`，控制是否缓存配置用于新设备检测 | `meson.options:50-55`、`src/entity_manager/meson.build:9-10` |
| `fru-device` | 由配方决定 | 是否构建 fru-device，同时决定是否查找 `libi2c` | `meson.build:30-32`、`src/meson.build:14-16` |
| `gpio-presence` | `true` | 是否构建 gpio-presence 及依赖 `libgpiodcxx` | `meson.options:44-49`、`meson.build:45-51` |

`service_files/meson.build` 里有个对应关系表：FruDevice 服务对应 `fru-device` 选项，EntityManager 服务无条件安装，另两个对应各自选项（`service_files/meson.build:1-18`）。

> 从 C++ 角度看，这个项目用到了不少 C++23 的东西：编译标准在 `meson.build:4` 里的 `cpp_std=c++23` 定死；代码里有 `std::expected`（`src/entity_manager/em_config.cpp:7,10-33`）、`std::move_only_function`（`src/entity_manager/perform_scan.cpp:189-196`）、`std::flat_map`（`src/entity_manager/entity_manager.hpp:16`）、`std::ranges::to`（`src/fru_device/fru_device.cpp:1274`）等。Java 程序员读代码时遇到这些类型别慌，它们对应的就是"带错误返回的 Result"、"更省内存的 Map"这类概念。

---

## 2. 配置目录：`configurations/`

这个目录目前有 375 个 `.json` 文件，按厂商/平台分子目录（`configurations/` 下的一级目录有 3y-power、acbel、amd、ampere、ibm、intel、nvidia、supermicro 等几十个）。

几个关键事实：

- **一个设备一份配置**。README 把它列为需求："每种支持的设备型号恰好有一个配置文件"，宁可重复也不合并（`README.md:112-116`）。`CONFIG_FORMAT.md:24-47` 解释了原因：硬件的 bug 和怪癖是慢慢发现的，一物一文件能把改动的影响面缩到最小。
- 配置文件可以是单个对象，也可以是数组（`schemas/README.md:7-11`）。
- 文件命名必须是 `lower_snake_case`，这是为了和 OpenBMC C++ 源文件命名规范一致（`schemas/README.md:13-15`）。
- 新增/移动/删除配置文件后要跑 `scripts/generate_config_list.sh`，让 meson 能正确引用（`CONFIG_FORMAT.md:109-112`）。
- 配置不是长期稳定的 ABI，schema 会演进，所以配置文件不应该提交到 meta 层（`CONFIG_FORMAT.md:49-58`）。
- `blacklist.json` 在安装时被单独安装（`meson.build:79`），用于屏蔽特定设备；相关说明见 `docs/blacklist_configuration.md`。

`Exposes` 里能写的东西由 schema 约束——`Type` 决定语义。比如 `Status: "disabled"` 让记录默认不生效（`CONFIG_FORMAT.md:91`），`Bind*` 表示这条记录不完整、要跟别处记录合并（`CONFIG_FORMAT.md:92`），`DisableNode` 把另一个 Entity 设为 disabled（`CONFIG_FORMAT.md:93`）。

---

## 3. schema：`schemas/`

schema 目录解决两个问题：**配置写对没有**，以及**每个属性在 D-Bus 上是什么类型**。

### 3.1 校验发生在两个时机

- **构建时**：`validate-json` 默认开着，meson 定义 `custom_target('check_syntax', ...)` 跑 `scripts/validate_configs.py -v -k`（`meson.build:100-109`）。这保证进镜像的配置一定是合法 JSON。
- **运行时**：`runtime-validate-json` 默认关（`meson.options:38-43`）。打开后，`loadSingleConfigFile` 会用 valijson + `schemas/global.json` 校验每份配置（`src/entity_manager/configuration.cpp:46-50,89-109`）。`validateJson` 函数本体在 `src/entity_manager/configuration.cpp:146-156`。

### 3.2 `global.json` 定义了配置的骨架

顶层 `oneOf` 允许：数组、单个 `EMConfig`、单个 `EMPlatformConfig`（`schemas/global.json:189-212`）。

`EMConfig` 要求四个必填字段：`Exposes`、`Name`、`Probe`、`Type`（`schemas/global.json:158`）。`Type` 是枚举，包含 Board、Cable、Chassis、Cpu、Dimm、NVMe、PowerSupply、Valve（`schemas/global.json:83-96`）。注意 schema 里的类型集合和代码里的 `typeToInterface` 白名单并不完全一样（`src/entity_manager/entity_manager.cpp:200-213`）：schema 允许 Dimm，但 `typeToInterface` 里没有 Dimm，所以 Dimm 不会有顶层 Inventory 接口。这类"schema 允许但代码不认领"的情况，读代码时要留意。

`Probe` 的定义支持单字符串或字符串数组，数组元素可以是 `ProbeTerm`、`ProbeOperator`（`AND`/`OR`）、`ProbeModifier`（`MATCH_ONE`）（`schemas/global.json:4-31,53-62`）。`FOUND(...)` 和 D-Bus 接口探测分别用正则约束（`schemas/global.json:42-50`）。

`Exposes` 每条记录的 schema 在 `exposes_record.json`，通过 `$ref` 引入（`schemas/global.json:69-75`）。新增 Exposes 子 schema 要放进 schema 目录，并从 `global.json` 的 `EMExposesElement` 处引用（`schemas/README.md:59-65`）。

### 3.3 schema 还决定了 D-Bus 类型

这是这一版代码里比较特别的一层。`SchemaDBusTypes` 启动时把所有 schema 读进来（`src/entity_manager/schema_dbus_types.cpp:137-178`），建立"记录类型 + 属性名 → D-Bus 类型"的映射；`convertSchemaTypes` 用它来决定属性以什么类型发布（`src/entity_manager/dbus_interface.cpp:263-303`）。

为什么需要？因为 JSON 里的数字看不出它是 `uint8` 还是 `uint64`，字符串 `"0x4c"` 看起来是字符串但实际要发成整数。`my_first_sensors.md` 里的例子印证了这一点：`Address` 在 JSON 里是 `"0x4c"`（`docs/my_first_sensors.md:134`），到 D-Bus 上变成了 `t`（uint64）类型的 `76`（`docs/my_first_sensors.md:204`）。

---

## 4. "finder" 探测机制：其实代码里分成了两半

团队里常说"finder"，但**这个仓库里并没有一个叫 Finder 的类**。源码里的真实分工是：

1. **探测守护进程**（detection daemon）负责发现硬件事实；
2. **entity-manager 的 Probe 机制**负责拿配置去匹配这些事实。

两者通过 D-Bus 接口解耦。README 明确把 entity-manager 自己排除在探测之外（`README.md:118-122`）。

### 4.1 探测这一半：谁在发"事实"

| 探测源 | 发出什么 | 出处 |
| --- | --- | --- |
| fru-device | 扫 I2C，FRU 内容发布为 `xyz.openbmc_project.FruDevice` 接口，路径 `/xyz/openbmc_project/FruDevice/<产品名>` | `src/fru_device/fru_device.cpp:780-849,884-910` |
| devicetree-vpd-parser | 解析设备树 VPD，申请 `xyz.openbmc_project.MachineContext` | `src/devicetree_vpd_parser/devicetree_vpd_parser.cpp:5-8,25-38` |
| gpio-presence | 检测器件在位，发布 `xyz.openbmc_project.Inventory.Source.DevicePresence` | `src/gpio-presence/README.md:79-91` |
| peci-pcie / smbios-mdr | 通过 CPU 总线 / SMBIOS 表探测 PCIe 设备（外部仓库） | `README.md:50-55` |

fru-device 的 I2C 扫描细节：遍历 `/sys/bus/i2c/devices` 下的总线（`src/fru_device/fru_device.cpp:368-427,780-849`），读 FRU EEPROM，用产品名做对象路径段；同名时追加 `_1`、`_2`（`src/fru_device/fru_device.cpp:898-907`）。

### 4.2 匹配这一半：PerformScan / PerformProbe

这一半完全在 entity-manager 进程内，前面 0.3～0.5 节已经走过，这里只归纳调用关系：

```text
propertiesChangedCallback()                       防抖入口
  → propertiesChangedCallbackDebounced()          src/entity_manager/entity_manager.cpp:559-617
     → PerformScan::run()                         src/entity_manager/perform_scan.cpp:752-763
        → processConfigurations()                 收集 D-Bus 接口，生成 PerformProbe
        → findDbusObjects()                       GetSubTree + GetAll
        → PerformProbe::~PerformProbe()           doProbe 求值
           → probeDbus()                          属性正则匹配
           → updateSystemConfiguration()          模板替换 + 写 systemConfiguration
```

`PerformScan` 的析构函数有个巧妙设计：只要本轮有配置匹配成功（`_passed == true`），它就再造一个新的 `PerformScan` 跑下一轮，并把 `passedProbes` 和 `dbusProbeObjects` 传下去（`src/entity_manager/perform_scan.cpp:765-775`）。为什么要多轮？因为 `FOUND('...')` 依赖"前面的配置已经匹配成功"这个事实——第一轮匹配出的设备名，第二轮才能被 `FOUND` 命中。直到某一轮没有任何配置匹配，`_passed` 为假，才调最终回调（`src/entity_manager/perform_scan.cpp:776-779`）。

> 这相当于一个不动点迭代：反复跑"条件成立就注册 Bean"，直到没有新 Bean 可注册为止。Spring 的 `@ConditionalOnBean` 依赖顺序解析，和这个是同一类问题。

---

## 5. D-Bus 对象组装的细节

### 5.1 对象路径和接口名的生成规则

- 板级对象：`/xyz/openbmc_project/inventory/system/<type 小写>/<name>`（`src/entity_manager/utils.cpp:430-439`）。
- Exposes 子对象：在板级路径后追加清洗过的 `Name`（`src/entity_manager/entity_manager.cpp:324-327`）。
- 接口名：`"xyz.openbmc_project.Configuration." + Type`（`src/entity_manager/entity_manager.cpp:375-378`）；嵌套 object 再加 `.` + 属性名（`src/entity_manager/entity_manager.cpp:401-406`）；数组加数字下标（`src/entity_manager/entity_manager.cpp:446-452`）。

清洗动作有两个函数：`sanitizeForDBusPathSegment`（用于路径段）和 `validateDBusInterfaceSegments`（校验接口段是否合法），都在 `src/dbus_util.cpp`。`postExposesRecordsToDBus` 会先校验 `Type`，非法就整条跳过（`src/entity_manager/entity_manager.cpp:309-322`）。

文档里给了一条限制：Exposes 里的字典/数组最多一层，不支持两层嵌套（`docs/entity_manager_dbus_api.md:140-180`）。代码里的 `postConfigurationRecord` 也只处理"object 或 object 数组"，不递归更深（`src/entity_manager/entity_manager.cpp:393-462`）。

### 5.2 谁来创建接口、谁负责销毁

`EMDBusInterface::createInterface` 封装了 `objServer.add_interface`，并按 parent 把接口指针存进 `inventory` map，便于整批删除（`src/entity_manager/dbus_interface.cpp:55-78`）。`tryIfaceInitialize` 统一捕获初始化异常并记日志，不让一个坏接口拖垮进程（`src/entity_manager/dbus_interface.cpp:40-53`）。

删除时 `pruneConfiguration` 调 `removeDeviceInterfaces`（`src/entity_manager/entity_manager.cpp:513-528`），按之前记录的 parent 把接口全部移除。

`xyz.openbmc_project.Configuration.Object` 接口用引用计数管理：多条 Exposes 记录共享一个 Name 时，只在第一个上创建接口，最后一个被删时才移除（`src/entity_manager/dbus_interface.cpp:80-118`）。

### 5.3 可变属性与持久化

前面说过，六个白名单接口的属性可写（`src/entity_manager/entity_manager.cpp:41-50`）。`addProperty` 的 setter 回调里做三件事：更新内存值、`setJsonFromPointer` 写回 JSON、`configCache.writeJsonFiles` 落盘（`src/entity_manager/dbus_interface.hpp:182-199`）。可读写的接口还会被挂上 `Delete` 方法（`src/entity_manager/dbus_interface.cpp:362-365`，方法本体在 `src/entity_manager/dbus_interface.cpp:120-163`）。

开机时 `handleCurrentConfigurationJson()` 会检查固件版本哈希是否一致；一致就把 `/var/configuration/system.json` 拷到 `/tmp/configuration/last.json` 当作 `lastJson`，并把其中的 key 放进 `cachedBaseline`（`src/entity_manager/entity_manager.cpp:658-704`）。版本哈希用 `/etc/os-release` 的内容算（`src/entity_manager/utils.cpp:19-37`）。这就是为什么升级固件后配置会重新算——版本变了，旧缓存不认。

---

## 6. 热插拔监听

这一节把"事件从哪来"说透。

### 6.1 entity-manager 这一侧：三类 D-Bus match

`initFilters`（`src/entity_manager/entity_manager.cpp:730-775`）注册：

| match | 触发场景 | 处理 |
| --- | --- | --- |
| `nameOwnerChanged` | 有服务申请/释放 well-known name | 一律触发重扫；忽略 `:` 开头的唯一连接名（`src/entity_manager/entity_manager.cpp:733-747`） |
| `interfacesAdded` | D-Bus 上新增接口 | 只在新增接口属于 `probeInterfaces` 时触发（`src/entity_manager/entity_manager.cpp:751-759`） |
| `interfacesRemoved` | D-Bus 上移除接口 | 属于关注接口时触发，并清掉该路径的 match（`src/entity_manager/entity_manager.cpp:761-774`） |

另外扫描时对每个命中的路径注册 `PropertiesChanged` match（`src/entity_manager/entity_manager.cpp:706-723`），这样属性值变化也会触发重扫。

所有信号汇入 `propertiesChangedCallback()`，用 500ms 定时器防抖（`src/entity_manager/entity_manager.cpp:619-631`）。防抖期间又来新信号会重置定时器，所以连续的硬件事件只触发一次扫描。

### 6.2 fru-device 这一侧：inotify + 电源状态

fru-device 用 `inotify_add_watch` 盯 `/sys/bus/i2c/devices`，关注 `IN_CREATE | IN_MOVED_TO | IN_DELETE`（`src/fru_device/fru_device.cpp:1619-1620`）。事件里名字以 `i2c` 开头的，解析出总线号，重新扫描对应总线（以及它的根总线）（`src/fru_device/fru_device.cpp:1638-1664`）。此外还监听 host0 的电源状态，开机时才重扫（`src/fru_device/fru_device.cpp:1600-1611`）。

### 6.3 10 秒延迟"收割"

扫描结束后不立刻判定设备消失。`startRemovedTimer` 起一个 10 秒定时器，到期后才对 `lastJson` 里仍然缺失的设备发 Inventory Removed（`src/entity_manager/entity_manager.cpp:476-511`）。`scannedPowerOn` / `scannedPowerOff` 两个标志避免在上电/掉电阶段反复收割（`src/entity_manager/entity_manager.cpp:482-490`）。

---

## 7. 校验与错误处理

把散落各处的"防御性代码"集中列一下，便于建立整体印象：

| 场景 | 处理 | 出处 |
| --- | --- | --- |
| JSON 语法错 | 记日志，跳过该文件 | `src/entity_manager/configuration.cpp:33-44` |
| 配置缺 `Name`/`Type`/`Probe`/`Exposes` | `EMConfig` 构造函数抛异常，`fromJson` 捕获返回 `nullopt` | `src/entity_manager/em_config.cpp:190-219,35-56` |
| Probe 解析失败 | 跳过该配置并记日志 | `src/entity_manager/perform_scan.cpp:727-736` |
| Exposes 记录类型不能转成 schema 声明的 D-Bus 类型 | 整条记录不发布 | `src/entity_manager/entity_manager.cpp:332-337` |
| 接口里有一个属性类型不对 | 整个接口不发布，避免半成品 | `src/entity_manager/dbus_interface.cpp:312-322` |
| 数组元素类型不一致 | 记日志跳过该属性 | `src/entity_manager/dbus_interface.cpp:340-351` |
| D-Bus 接口名不合法 | 校验后跳过该 Exposes 记录 | `src/entity_manager/entity_manager.cpp:309-322` |
| 对象在扫描中被删（`EBADR`） | 跳过，不重试 | `src/entity_manager/perform_scan.cpp:62-74` |
| 连不上 object mapper | 重试 5 次，仍失败则 `std::exit(EXIT_FAILURE)` | `src/entity_manager/perform_scan.cpp:138-162` |
| FRU 内容没变 | 跳过重新发布，避免 InterfacesAdded/Removed 抖动 | `src/fru_device/fru_device.cpp:1234-1248` |
| 接口初始化抛异常 | `tryIfaceInitialize` 捕获并记日志 | `src/entity_manager/dbus_interface.cpp:40-53` |
| schema 目录读不了 | 记日志，不让它把启动过程抛异常打断 | `src/entity_manager/schema_dbus_types.cpp:139-145,168-172` |

一个总的原则能从代码里读出来：**宁愿少发布一个接口，也不发布一个类型错误的接口**。因为消费方（dbus-sensors 等）是按接口类型找配置的，一个半对的接口比没有接口更难排查。

---

## 8. 几个特殊要点

1. **Probe 没有运算符优先级**，严格左到右。原因写在 `docs/probe_grammar.md:70-88`，是为了兼容已有的平台配置。改 Probe 时别按常规优先级理解。

2. **`PerformProbe` 用析构函数触发求值**。这不是常规写法，阅读时要知道 `shared_ptr` 释放即求值（`src/entity_manager/perform_probe.cpp:174-182`）。

3. **多轮扫描**是为 `FOUND('...')` 服务的，直到没有新配置匹配才结束（`src/entity_manager/perform_scan.cpp:765-780`）。

4. **类型白名单有两套**：顶层 Inventory 接口看 `typeToInterface`（`src/entity_manager/entity_manager.cpp:200-213`），内核设备创建看 `devices::exportTemplates`（`src/entity_manager/overlay.cpp:281-295`）。两套都登记了，配置才完整生效。

5. **`Board` 和 `Platform` 是特殊类型**。Board 等类型挂 Inventory 接口，而 Platform 不挂 Inventory 接口，而是用 `xyz.openbmc_project.Configuration.Platform`（`src/entity_manager/entity_manager.cpp:205-213`）。Platform 配置的 schema 在 `schemas/global.json:160-187`，Exposes 用另一套平台 schema（`schemas/platform/platform_exposes_record.json`，见 `schemas/global.json:170`）。

6. **overlay 需要真的往 sysfs 写设备**。对 dbus-sensors 来说，entity-manager 不只是"发布配置"，还提前把内核驱动挂好了（`src/entity_manager/overlay.cpp:115-130,148-204`）。`my_first_sensors.md` 里的 `echo "24c02 0x50" > /sys/bus/i2c/devices/i2c-18/new_device` 就是手动模拟这一步（`docs/my_first_sensors.md:167-172`）。

7. **事件日志也是配置驱动的产物**。设备增删会发 `InventoryAdded` / `InventoryRemoved` 的 D-Bus event 和 journal 日志（`src/entity_manager/log_device_inventory.cpp:81-133`），而且受 `EM_CACHE_CONFIGURATION` 开关控制（`src/entity_manager/log_device_inventory.cpp:83-86`）。

8. **启动顺序在 systemd 单元里定死**。entity-manager 明确声明 `After=` 三个探测服务，避免 BMC 重启时探测服务已停、entity-manager 误报设备移除（`service_files/xyz.openbmc_project.EntityManager.service:3-8`）。

---

## 9. 读代码路线建议

如果你想自己顺着源码走一遍，按这个顺序最省力：

1. `src/entity_manager/main.cpp:12-30` — 看进程入口，认识两个配置目录；
2. `src/entity_manager/configuration.cpp:20-124` — 看配置怎么加载成 `EMConfig`；
3. `configurations/acbel/rica_psu.json:1-26` — 对着真实配置看字段；
4. `src/entity_manager/perform_scan.cpp:714-780` 和 `src/entity_manager/perform_probe.cpp:19-182` — 看匹配如何发生；
5. `src/entity_manager/entity_manager.cpp:82-391` — 看 JSON 如何变成 D-Bus 对象和接口；
6. `src/entity_manager/dbus_interface.cpp:305-367` 和 `src/entity_manager/dbus_interface.hpp:169-220` — 看属性如何落地；
7. `docs/my_first_sensors.md:124-250` — 看一个完整的、带 `busctl` 输出的例子收尾。

走完这七步，你会对"配置驱动硬件描述"这句话有具体的、能落到行号上的理解。
