# Untitled1 插件框架设计文档

## 文档说明

本文档基于 `Untitled1` 插件的最终代码，详细描述每个类的职责、每个方法的功能、实现逻辑以及开发/维护时需要注意的关键点。适用于新手开发者理解插件架构，也作为后续扩展的参考手册。

---

## 目录

1. [主类 `Untitled1`](#1-主类-untitled1)
2. [常量类 `Constants`](#2-常量类-constants)
3. [命令类 `Untitled1Command`](#3-命令类-untitled1command)
4. [事件监听类 `Untitled1EventListener`](#4-事件监听类-untitled1eventlistener)
5. [配置管理器 `ConfigManager`](#5-配置管理器-configmanager)
6. [配置注册表 `ConfigRegistry`](#6-配置注册表-configregistry)
7. [消息管理器 `MessageManager`](#7-消息管理器-messagemanager)
8. [经济管理器 `VaultManager`](#8-经济管理器-vaultmanager)
9. [资源文件设计](#9-资源文件设计)
   - `plugin.yml`
   - `config.yml`
   - `lang.yml` / `lang_en.yml`

---

## 1. 主类 `Untitled1`

**包路径**：`net.untitled1.Untitled1`  

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `instance` | `Untitled1` | 单例实例 |
| `configRegistry` | `ConfigRegistry` | 配置注册表 |
| `messageManager` | `MessageManager` | 消息管理器 |
| `vaultManager` | `VaultManager` | 经济管理器 |
| `commandManager` | `PaperCommandManager` | ACF 命令管理器 |

### 方法

#### `onEnable()`

#### `onDisable()`

#### `setupCommands()`

- **功能**：初始化 ACF 命令管理器并注册命令类。
- **注意**：
  1. 加载 ACF 语言文件：设置默认语言为 `Locale.ENGLISH`，从 `lang_en.yml` 加载本地化键值对，开启 `usePerIssuerLocale(true)`（根据玩家客户端语言自动切换）。

#### `reload()`

- **功能**：重载插件目录下的所有文件（配置和语言）。

#### `getInstance()` / 各 Getter

- **功能**：提供全局访问点。
- **注意**：单例模式便于工具类或非注入类获取插件实例，但应避免过度使用。

---

## 2. 常量类 `Constants`

**包路径**：`net.untitled1.Constants`  
**职责**：集中管理字符串常量，避免魔法值。

| 常量 | 值 | 用途 |
|------|---|------|
| `ACF_BASE_KEY` | `"commands"` | ACF 语言文件的基础键（未实际使用，保留） |
| `ADMIN_PERMISSION` | `"untitled1.admin"` | 管理员权限 |
| `RELOAD_PERMISSION` | `"untitled1.admin.reload"` | 重载权限 |
| `HELP_PERMISSION` | `"untitled1.help"` | 帮助命令权限 |
| `PLAYER_PERMISSION` | `"untitled1.player"` | 玩家基础权限 |
| `COMMAND_ALIAS` | `"untitled1"` | 主命令名称 |
| `COMMAND_ALIAS_SHORT` | `"ut1"` | 命令短别名 |

**注意**：
- 所有权限节点必须在 `plugin.yml` 中声明，否则权限检查可能失效。
- 修改常量值后需重新编译，并同步更新 `plugin.yml` 中的对应权限名。

---

## 3. 命令类 `Untitled1Command`

**包路径**：`net.untitled1.command.Untitled1Command`  
**继承**：`BaseCommand`  
**职责**：定义所有插件命令，使用 ACF 注解驱动。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `plugin` | `Untitled1` | 插件主类引用 |
| `msg` | `MessageManager` | 消息管理器 |

### 构造方法

- **功能**：初始化字段。
- **注意**：依赖注入，便于单元测试。

### 方法

#### `onHelp(CommandHelp help)`

- **注解**：`@HelpCommand` `@Subcommand("help")` `@CommandPermission(Constants.PLAYER_PERMISSION)` `@Description("{@@commands.descriptions.help}")`
- **功能**：显示命令帮助。
- **实现**：`help.showHelp()`，ACF 自动收集所有子命令的描述和语法。
- **注意**：`@Description` 中的 `{@@...}` 是从 ACF 语言文件读取本地化描述，若未加载成功，则显示注解中的原始字符串。权限使用 `PLAYER_PERMISSION`，所有玩家均可查看帮助。

#### `onReload(CommandSender sender)`

- **注解**：`@Subcommand("reload|r")` `@CommandPermission(Constants.RELOAD_PERMISSION)` `@Description("{@@commands.descriptions.reload}")`
- **功能**：重载插件配置。
- **实现**：调用 `plugin.reload()`，然后发送 `reload_success` 消息。
- **注意**：权限节点为 `untitled1.admin.reload`，需 OP 或单独授权。

#### `onBalance(CommandSender sender, Player player)`

- **注解**：`@Subcommand("balance|bal")` `@CommandPermission(Constants.PLAYER_PERMISSION)` `@Description("查询自己的金币余额")`
- **功能**：显示玩家余额。
- **实现**：
  1. 检查 `plugin.getVaultManager().isEnabled()`，若不可用则发送 `vault_not_available`。
  2. 获取余额并格式化：`plugin.getVaultManager().format(balance)`。
  3. 发送 `balance_query` 消息，替换 `{balance}` 占位符。
- **注意**：该方法接收 `Player player` 参数，ACF 自动将命令执行者转为 `Player`，若控制台执行会抛出异常。实际应使用 `CommandSender` 并通过 `instanceof` 判断，当前设计只允许玩家执行（权限已限制但类型安全不够）。可优化为 `CommandSender` 并手动获取玩家。

#### `onGive(CommandSender sender, Player target, @Optional Integer amount)`

- **注解**：`@Subcommand("give|g")` `@CommandPermission(Constants.ADMIN_PERMISSION)` `@CommandCompletion("@players @range:1-64")`
- **功能**：给予目标玩家锋利 III 钻石剑，并可选扣款。
- **实现逻辑**：
  1. 计算最终数量 `finalAmount`（默认1，最大64，最小1）。
  2. 若 `finalAmount < 1`，发送 `invalid_amount` 错误。
  3. 经济扣款部分（仅当发送者是玩家且 Vault 可用）：
     - 计算费用 `cost = finalAmount * 10.0`。
     - 检查玩家余额 `has(player, cost)`，不足则发送 `insufficient_funds` 并返回。
     - 扣款 `withdraw(player, cost)`，发送 `payment_deducted` 消息。
  4. 创建物品 `createSharpnessSword()`，设置数量，加入目标玩家背包。
  5. 发送成功消息 `give_success` 给发送者，发送 `received_sword` 给目标玩家。
- **注意**：
  - `@Optional Integer amount` 允许不输入数量，此时为 `null`。
  - 经济扣款发生在物品给予之前，避免扣款成功但背包满导致物品丢失（但背包满时 `addItem` 会掉落在地上，逻辑仍可接受）。
  - 控制台执行该命令时跳过经济扣款（因为 `sender instanceof Player` 为 false），可直接给予物品。

#### `createSharpnessSword()`

- **功能**：创建一个锋利 III 的钻石剑物品。
- **实现**：`new ItemStack(Material.DIAMOND_SWORD)` 并添加附魔 `Enchantment.SHARPNESS, 3`。
- **注意**：该方法可扩展为从配置读取物品属性（如名称、Lore、自定义模型数据）。

---

## 4. 事件监听类 `Untitled1EventListener`

**包路径**：`net.untitled1.event.Untitled1EventListener`  
**实现**：`Listener`  
**职责**：监听玩家加入事件，发送自定义欢迎消息。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `msg` | `MessageManager` | 消息管理器 |

### 构造方法

- **功能**：注入 `MessageManager`。
- **注意**：通过 `plugin.getMessageManager()` 获取，解耦插件主类。

### 方法

#### `onPlayerJoin(PlayerJoinEvent event)`

- **注解**：`@EventHandler`
- **功能**：处理玩家加入事件。
- **实现**：
  1. `event.setJoinMessage(null)` 取消服务器默认的加入消息。
  2. `msg.send(event.getPlayer(), "welcome_message", Placeholder.parsed("player", player.getName()))` 发送自定义欢迎消息。
- **注意**：
  - 若 `welcome_message` 键缺失，`MessageManager` 会显示 "Missing message: welcome_message"。
  - 占位符 `{player}` 会被替换为玩家名。
  - 此事件是同步的，操作简单，无需考虑异步问题。

---

## 5. 配置管理器 `ConfigManager`

**包路径**：`net.untitled1.manager.ConfigManager`  
**职责**：管理单个 YAML 配置文件，提供类型安全的读写和自动保存。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `plugin` | `JavaPlugin` | 插件实例 |
| `fileName` | `String` | 完整文件名（含 `.yml`） |
| `config` | `YamlConfiguration` | 内存中的配置对象 |
| `configFile` | `File` | 磁盘文件对象 |

### 构造方法

- **注意**：文件扩展名自动添加，调用方只需传入基础名（如 `"config"`）。

### 方法

#### `reload()`

- **功能**：从磁盘重新加载配置。

#### `save()`

- **功能**：将当前配置保存到磁盘。
- **注意**：该方法由 `set()` 自动调用，也可手动调用。

#### 类型安全读取方法

| 方法 | 说明 |
|------|------|
| `getString(path, defaultValue)` | 返回字符串 |
| `getInt(path, defaultValue)` | 返回整数 |
| `getFloat(path, defaultValue)` | 通过 `getDouble` 转型返回浮点数 |
| `getLong(path, defaultValue)` | 返回长整数 |
| `getBoolean(path, defaultValue)` | 返回布尔值 |
| `getDouble(path, defaultValue)` | 返回双精度浮点数 |
| `getList(path, defaultValue)` | 返回 `List<?>` |
| `getStringList(path, defaultValue)` | 返回 `List<String>`，空列表时返回默认值 |
| `getIntegerList(path, defaultValue)` | 返回 `List<Integer>` |
| `getSection(path)` | 返回 `Map<String, Object>` 表示配置段，不存在时返回空 `HashMap` |
| `get(path, defaultValue)` | 返回 `Object`，需调用者自行转型 |
| `get(path, defaultValue, Class<T>)` | 泛型读取，支持基本类型自动转换 |

**注意**：
- 所有方法都接受默认值，避免返回 `null`。
- `getFloat` 依赖 `getDouble`，可能存在精度损失，但对于配置值通常可接受。
- `getStringList` 和 `getIntegerList` 在列表为空时返回默认值，而不是空列表（取决于业务需求）。

#### `set(path, value)`

- **功能**：设置配置值并自动保存。
- **注意**：频繁调用 `set` 会频繁写磁盘，批量修改时应使用 `getHandle()` 直接操作，最后手动 `save()`。

#### `getHandle()`

- **功能**：返回原始 `YamlConfiguration` 对象。
- **注意**：仅在需要调用原生方法时使用，破坏封装性。

---

## 6. 配置注册表 `ConfigRegistry`

**包路径**：`net.untitled1.manager.ConfigRegistry`  
**职责**：管理多个 `ConfigManager` 实例，提供统一的重载和保存。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `plugin` | `Untitled1` | 插件实例 |
| `managers` | `Map<String, ConfigManager>` | 已加载的配置管理器缓存 |

### 构造方法

### 方法

#### `get(String fileName)`

- **功能**：获取或创建指定名称的 `ConfigManager`。
- **实现**：`managers.computeIfAbsent(fileName, name -> new ConfigManager(plugin, name))`。
- **注意**：首次调用会创建新实例并自动加载文件，后续调用返回缓存实例。

#### `reloadAll()`

- **功能**：重载所有已加载的配置文件。
- **实现**：遍历 `managers.values()`，调用每个 `ConfigManager.reload()`。

#### `saveAll()`

- **功能**：保存所有已加载的配置文件。
- **实现**：遍历 `managers.values()`，调用每个 `ConfigManager.save()`。

**设计注意**：
- 该注册表仅在主类中创建一次，其他类通过 `plugin.getConfigRegistry()` 获取。
- 未通过 `get` 加载的配置文件不会出现在缓存中，因此不会被 `reloadAll` 重载。

---

## 7. 消息管理器 `MessageManager`

**包路径**：`net.untitled1.manager.MessageManager`  
**职责**：基于 MiniMessage 加载语言文件，发送带前缀的彩色消息。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `plugin` | `Untitled1` | 插件实例 |
| `langConfig` | `ConfigManager` | 语言文件配置管理器（`lang.yml`） |
| `miniMessage` | `MiniMessage` | MiniMessage 解析器 |
| `prefix` | `Component` | 解析后的前缀组件 |

### 构造方法

- **参数**：`Untitled1 plugin`
- **注意**：要求 `ConfigRegistry` 已准备好，且 `lang` 配置已预加载（主类中已做）。

### 方法

#### `reload()`

- **功能**：重新加载语言文件并更新前缀。
- **实现**：
  1. 从 `langConfig` 读取 `prefix` 键，默认值 `"<gray>[<gold>Untitled1</gold>]</gray> "`。
  2. `this.prefix = miniMessage.deserialize(prefixRaw);`
- **注意**：调用 `reload()` 不会重新读取整个 `langConfig`，因为 `langConfig.reload()` 需要单独调用。实际重载时应先调用 `langConfig.reload()`，再调用此方法。主类的 `reload()` 已通过 `configRegistry.reloadAll()` 重载了 `lang`，因此只需调用 `messageManager.reload()` 刷新前缀。

#### `getRaw(String key)`

- **功能**：获取原始消息字符串（不含前缀，不解析占位符）。
- **实现**：`langConfig.getString(key, key)`，若键不存在返回键名本身。
- **用途**：日志输出。

#### `getMessage(String key, TagResolver... resolvers)`

- **功能**：获取带前缀的完整消息组件。
- **实现**：
  1. 从 `langConfig` 读取原始字符串，若键不存在返回 `"<red>Missing message: " + key + "</red>"`。
  2. `Component msg = miniMessage.deserialize(raw, resolvers);`
  3. 若 `key.equals("prefix")` 直接返回 `msg`，否则返回 `prefix.append(msg)`。
- **注意**：占位符通过 `resolvers` 传入，例如 `Placeholder.parsed("player", name)`。

#### `sendFormat(CommandSender sender, String key, Object... args)`

- **功能**：发送格式化消息，自动处理命名占位符和位置占位符。
- **实现逻辑**：
  - 若 `args.length` 为偶数，视为键值对（`key1, val1, key2, val2...`），构造命名占位符。
  - 若为奇数，视为位置占位符值列表，自动转为 `{0}`, `{1}` 等。
  - 调用 `send(sender, key, resolvers)`。
- **注意**：命名占位符方式推荐，更清晰；位置占位符用于兼容旧代码。

#### `send()` 系列方法

| 方法签名 | 说明 |
|----------|------|
| `send(Audience audience, String key, TagResolver... resolvers)` | 发送给任意 `Audience`（控制台、玩家等） |
| `send(CommandSender sender, String key, TagResolver... resolvers)` | 发送给 `CommandSender`（Paper 中实现了 `Audience`） |
| `send(Player player, String key, TagResolver... resolvers)` | 发送给 `Player` |

**实现**：统一调用目标对象的 `sendMessage(getMessage(key, resolvers))`。

#### `success()` / `error()`

- **功能**：语义化便捷方法，实际调用 `send`。
- **注意**：颜色样式由语言文件中的消息本身决定，方法名仅用于代码可读性。

#### 日志方法

| 方法 | 说明 |
|------|------|
| `log(String key)` | 输出 `INFO` 级别日志，内容为 `getRaw(key)` |
| `logFormat(String key, Object... args)` | 格式化日志，支持 `{0}` `{1}` 位置占位符 |
| `log(Level level, String key, Object... args)` | 指定日志级别 |

**实现**：通过 `plugin.getLogger().info(...)` 输出。

**注意**：日志消息不会添加前缀，因为 `getRaw` 返回原始字符串。

---

## 8. 经济管理器 `VaultManager`

**包路径**：`net.untitled1.manager.VaultManager`  
**实现**：`Listener`  
**职责**：封装 Vault Economy API，动态绑定经济插件。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `plugin` | `JavaPlugin` | 插件实例 |
| `economy` | `Economy` | Vault 经济服务实例 |

### 构造方法

- **参数**：`JavaPlugin plugin`
- **实现**：
  1. `this.plugin = plugin;`
  2. 调用 `setupEconomy()` 尝试获取服务。
  3. `plugin.getServer().getPluginManager().registerEvents(this, plugin);` 注册自身为事件监听器。
- **注意**：监听 `ServiceRegisterEvent` 是为了处理经济插件在 `Untitled1` 之后加载的情况。

### 方法

#### `setupEconomy()`

- **功能**：尝试获取 Vault Economy 服务。
- **实现**：
  1. 检查 Vault 插件是否存在，不存在则警告并返回。
  2. 通过 `ServicesManager` 获取 `Economy` 服务注册。
  3. 若成功，保存到 `economy` 并输出信息日志。
- **注意**：该方法可能被调用多次（构造时和事件中），需做好幂等性。

#### `onServiceRegister(ServiceRegisterEvent event)`

- **注解**：`@EventHandler`
- **功能**：监听经济服务注册事件，当经济插件后加载时重新尝试绑定。
- **实现**：若注册的服务是 `Economy.class` 且当前 `economy == null`，则调用 `setupEconomy()`。
- **注意**：防止重复绑定，因为 `setupEconomy` 内部有检查。

#### `isEnabled()`

- **功能**：返回经济功能是否可用。
- **实现**：`return economy != null;`

#### 经济操作方法

| 方法 | 实现逻辑 |
|------|----------|
| `getBalance(OfflinePlayer player)` | 若 `economy == null` 返回 `0.0`，否则调用 `economy.getBalance(player)` |
| `has(OfflinePlayer player, double amount)` | 返回 `economy != null && economy.has(player, amount)` |
| `withdraw(OfflinePlayer player, double amount)` | 返回 `economy != null && economy.withdrawPlayer(player, amount).transactionSuccess()` |
| `deposit(OfflinePlayer player, double amount)` | 返回 `economy != null && economy.depositPlayer(player, amount).transactionSuccess()` |
| `getCurrencyName()` | 单数货币名称，默认 `"coins"` |
| `getCurrencyNamePlural()` | 复数货币名称 |
| `format(double amount)` | 格式化金额，默认 `amount + " coins"` |

**注意**：所有方法在 `economy == null` 时返回安全默认值，避免 `NullPointerException`。

---

## 9. 资源文件设计

### 9.1 `plugin.yml`

**作用**：插件描述文件，服务器读取此文件识别插件。

| 键 | 值 | 说明 |
|----|---|------|
| `name` | `untitled1` | 插件名称 |
| `version` | `0.1` | 版本号 |
| `main` | `net.untitled1.Untitled1` | 主类全限定名 |
| `api-version` | `1.21` | 声明 API 版本 |
| `authors` | `- PAG_Pillager` | 作者列表 |
| `website` | `www.baiyu.fun` | 网站 |
| `description` | `描述内容` | 插件描述 |
| `commands` | 见下 | 命令声明（非 ACF 必需，但可提供元信息） |
| `permissions` | 见下 | 权限节点声明 |

**命令声明**：
```yaml
commands:
  untitled1:
    description: admin command
    aliases: [ut1]
    permission-message: §c你没有权限！
    usage: /<command> reload
```
- `aliases` 必须与 `Constants.COMMAND_ALIAS_SHORT` 一致。
- `permission-message` 是当玩家没有命令权限时显示的默认消息，ACF 会覆盖此行为。

**权限声明**：
```yaml
permissions:
  untitled1.admin:
    description: Allows admin command
    default: op
  untitled1.admin.reload:
    description: Allows reload
    default: op
  untitled1.player:
    description: Allows player command
    default: true
```
- `untitled1.help` 权限未声明，需补充，否则非 OP 玩家可能无法使用帮助命令。
- `default: op` 表示只有 OP 有此权限；`default: true` 表示所有玩家默认拥有。

### 9.2 `config.yml`

**当前内容**：
```yaml
enable: true
messages_enable: true
```
**用途**：预留配置项，尚未在代码中使用。可扩展为功能开关（如是否启用经济扣款、是否发送欢迎消息等）。

**注意事项**：若添加新配置项，需在代码中通过 `ConfigManager` 读取。

### 9.3 `lang.yml` / `lang_en.yml`

**格式**：MiniMessage 标记语言。  
**设计原则**：
- `prefix` 键为全局前缀，自动添加到除自身外的所有消息前。
- 消息键命名采用小写加下划线，如 `welcome_message`。
- 占位符使用 `{name}` 形式，在代码中通过 `Placeholder.parsed("name", value)` 替换。

**核心消息键列表**：

| 键 | 中文示例 | 英文示例 |
|----|---------|---------|
| `prefix` | `<gray>[<gold>untitled1</gold>]</gray> ` | 相同 |
| `no_permission` | `<red>你没有权限执行此命令！</red>` | `<red>You do not have permission to use this command!</red>` |
| `welcome_message` | `<green>欢迎 {player} 加入服务器！</green>` | `<green>Welcome {player} to the server!</green>` |
| `plugin_on_enabled` | `<green>插件已加载！</green>` | `<green>Plugin enabled!</green>` |
| `plugin_on_disabled` | `<red>插件已卸载！</red>` | `<red>Plugin disabled!</red>` |
| `reload_success` | `<green>插件配置已重载！</green>` | `<green>Plugin configuration reloaded!</green>` |
| `help` | 多行帮助文本 | 多行帮助文本 |
| `commands.descriptions.help` | `显示帮助信息` | `Show help information` |
| `commands.descriptions.reload` | `重载插件配置` | `Reload plugin configuration` |
| `commands.descriptions.give` | `给玩家添加锋利三钻石剑` | `Give Sharpness III Diamond Sword to a player` |
| `received_sword` | `<yellow>你收到了 <green>{amount}</green> 把锋利三钻石剑！</yellow>` | `<yellow>You received <green>{amount}</green> Sharpness III Diamond Swords!</yellow>` |
| `give_success` | `<green>已成功给 <yellow>{player}</yellow> 添加了 <gold>{amount}</gold> 把锋利三钻石剑！</green>` | `<green>Successfully gave <yellow>{player}</yellow> <gold>{amount}</gold> Sharpness III Diamond Swords!</green>` |
| `invalid_amount` | `<red>无效数量: <white>{amount}</white>，数量必须在 1-64 之间！</red>` | `<red>Invalid amount: <white>{amount}</white>, must be between 1 and 64!</red>` |
| `vault_not_available` | `<red>经济系统未启用，请确保已安装 Vault 和经济插件。</red>` | `<red>Economy system not available. Please install Vault and an economy plugin.</red>` |
| `vault_hooked` | `<green>已成功挂钩经济插件！</green>` | `<green>Successfully hooked into economy plugin!</green>` |
| `vault_not_hooked` | `<yellow>未检测到经济插件，经济相关功能不可用。</yellow>` | `<yellow>No economy plugin detected, economy features disabled.</yellow>` |
| `balance_query` | `<green>你的余额: {balance}</green>` | `<green>Your balance: {balance}</green>` |
| `insufficient_funds` | `<red>余额不足！需要 {cost}</red>` | `<red>Insufficient funds! Need {cost}</red>` |
| `payment_deducted` | `<green>已扣除 {cost}</green>` | `<green>Deducted {cost}</green>` |

**注意事项**：
- 消息中不能出现未转义的 `<` 或 `>`，否则 MiniMessage 解析异常。
- 多行消息使用 `|` 符号，每行缩进必须一致。
- 修改语言文件后需执行 `/untitled1 reload` 生效。

---

## 附录：开发与维护注意事项

### 通用原则
1. **避免静态工具类**：所有管理器应通过实例访问，便于模拟和替换。
2. **依赖注入**：通过构造函数传递依赖（如 `MessageManager` 传给命令类）。
3. **配置与代码分离**：所有用户可修改的文本和数值应放入配置文件。
4. **日志记录**：关键操作（如配置加载失败、经济挂钩）应记录日志，方便排查。

### ACF 命令框架
- 务必调用 `enableUnstableAPI("help")`，否则 `@HelpCommand` 不生效。
- 参数类型自动转换：`Player` 参数自动解析在线玩家，若玩家不在线会抛出异常，可自定义异常处理。
- `@CommandCompletion` 内置完成器：`@players`、`@worlds`、`@range:min-max` 等。

### MiniMessage 消息
- 使用 `Placeholder.parsed("name", value)` 替换占位符，`value` 中的 MiniMessage 标签会被解析，注意防止注入。
- 若需要纯文本（如玩家名可能包含颜色代码），应使用 `Placeholder.unparsed("name", value)` 或先清理。

### Vault 集成
- 软依赖：不要在 `plugin.yml` 中 `depend: [Vault]`，而是通过 `softdepend` 或动态检查。
- `OfflinePlayer` 的余额操作可能不被所有经济插件支持，生产环境需测试。

### 性能考量
- `ConfigManager.set()` 自动保存，频繁调用会 IO 负担，批量操作应手动控制。
- 事件监听中避免耗时操作（如数据库查询），必要时使用异步线程。

---

*文档版本 1.0，对应插件版本 0.1。*