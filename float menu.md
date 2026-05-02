# 悬浮菜单插件设计文档
（BY float menu，简称 BYFM）

### 引言
以下是一个使用 Paper 1.21+ Display 实体 API 直接编写的悬浮菜单演示设计，不依赖 YAML 解析，仅通过代码快速实现一个初始菜单demo。

---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---

## 一、demo设计

### 1. 引言
以下是一个使用 Paper 1.21+ Display 实体 API 直接编写的悬浮菜单演示设计，不依赖 YAML 解析，仅通过代码快速实现一个初始菜单demo。

### 2. 演示目标
玩家使用胡萝卜钓竿右键时，整体相对玩家会有偏移、旋转，在面前生成一个悬浮菜单。
菜单包含两个图标：一个“钻石”图标（点击给予钻石）、一个“关闭”图标（点击关闭菜单）。
玩家通过视线瞄准图标时，图标放大并发光，用TextDisplay显示文字提示；空手或手持胡萝卜钓竿左键点击图标执行对应动作。

### 3. 实现细节
#### 坐标
**底座**：(0, 0, 1.5)，分为x轴、y轴、z轴三轴。并不是绝对坐标，而是相对玩家的偏移量，且x、y、z为相对玩家的水平横向偏移量、垂直偏移量、深度偏移量。
菜单创建时记录玩家的初始 yaw，之后菜单保持该朝向不变。玩家移动时菜单跟随位置，但朝向锁定为创建时的方向。

**普通图标**：(x, y, z)，分为x轴、y轴、z轴三轴。并不是绝对坐标，而是基于底座的偏移量，且同样是相对玩家的偏移量，且x、y、z为相对玩家的水平横向偏移量、垂直偏移量、深度偏移量。
同理，旋转量也是相对玩家的旋转量。玩家移动、旋转时，悬浮菜单会跟随玩家移动，但不跟随玩家旋转。
表现例子：
有xyz空间直角坐标系（y为垂直轴），玩家的眼睛位于(0, 0, 0)，方向为(1, 0, 1)，底座设置为(0, 0, 1.5)，普通图标设置为(-0.5, 0, 0)。则普通图标全局位置为(1.5-√2/4, 0, 1.5+√2/4)。

#### 创建菜单底座
在玩家眼睛前方处（坐标）生成一个 item_display 实体作为底座（无实际物品，仅用于定位），再生成一个黑色背景板。
**黑色背景板**：使用 text_display 实体，设置其文本为空格或透明，并设置 background 为黑色，同时调整 transformation 使其覆盖图标区域。

#### 图标
在底座相对位置生成两个 item_display：
钻石图标：位置 (-0.5, 0, 0)，物品为钻石，名称“&b领取钻石”。
关闭图标：位置 (0.2, 1.0, 0)，物品为屏障，名称“&c关闭菜单”。
空手或手持胡萝卜钓竿左键点击图标执行对应动作。
交互实体设置标签floatmenu_interact，并存储所属菜单ID到PersistentDataContainer。

#### 高亮图标
缩放至1.2倍并设置发光，插值时长4 tick。
**文字**：为每个图标预先创建一个隐藏的TextDisplay，位置 = 图标位置 + hover_offset。当图标被瞄准时，显示该文本实体，并更新文本内容（解析变量）。当失去瞄准时，隐藏该实体。
**图标**：图标放大1.2倍，并发对应文字颜色光。当失去瞄准时，隐藏该实体。

#### 性能与清理
每tick更新所有活跃菜单的位置和射线检测。
玩家退出、切换世界、死亡时自动关闭菜单，将菜单状态设置为关闭。
实体设置setPersistent(false)，避免保存到区块。
使用标签快速筛选实体，避免全局搜索。
**关闭菜单**：点击关闭图标时，或玩家退出、切换世界、死亡时，将菜单状态设置为关闭。状态为关闭时，移除所有实体。

---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
## 二、插件实现设计

```
net.untitled1.floatmenu/
├── FloatMenuPlugin.java              # 主插件类
├── Constants.java                    # 常量定义
│
├── menu/
│   ├── FloatMenu.java               # 悬浮菜单核心类
│   ├── FloatMenuManager.java        # 菜单管理器（全局）
│   ├── MenuHolder.java              # 玩家菜单持有者
│   └── MenuState.java               # 菜单状态枚举
│
├── icon/
│   ├── MenuIcon.java                # 图标接口
│   ├── AbstractMenuIcon.java        # 图标抽象基类
│   ├── ItemIcon.java                # 物品图标实现
│   └── IconAction.java              # 图标点击动作接口
│
├── entity/
│   ├── DisplayEntityFactory.java    # Display实体工厂
│   ├── BasePlate.java               # 底座实体封装
│   ├── BackgroundBoard.java         # 背景板实体封装
│   └── IconEntity.java              # 图标实体封装
│
├── transform/
│   ├── RelativePosition.java        # 相对位置计算
│   ├── MenuTransform.java           # 菜单变换（位置、旋转）
│   └── InterpolationHelper.java     # 插值动画辅助
│
├── interaction/
│   ├── RaycastDetector.java         # 射线检测
│   ├── HoverHandler.java            # 悬停处理
│   ├── ClickHandler.java            # 点击处理
│   └── InteractionResult.java       # 交互结果
│
├── task/
│   ├── MenuUpdateTask.java          # 菜单更新任务（每tick）
│   └── MenuCleanupTask.java         # 清理任务
│
├── listener/
│   ├── PlayerInteractListener.java  # 玩家交互监听
│   ├── PlayerMovementListener.java  # 玩家移动监听
│   ├── PlayerStateListener.java     # 玩家状态监听（退出/死亡/切换世界）
│   └── MenuTriggerListener.java     # 菜单触发监听（胡萝卜钓竿）
│
└── action/
    ├── GiveItemAction.java          # 给予物品动作
    ├── CloseMenuAction.java         # 关闭菜单动作
    └── CommandAction.java           # 执行命令动作
```

---

### 1. 菜单设计

#### 1.1 FloatMenu.java - 悬浮菜单核心类

```
封装单个悬浮菜单的所有数据和状态

字段：
- UUID menuId                    # 菜单唯一ID
- Player owner                   # 所属玩家
- MenuState state                # 当前状态（OPEN/CLOSED）
- BasePlate basePlate            # 底座实体
- BackgroundBoard background     # 背景板
- List<IconEntity> icons         # 图标实体列表
- RelativePosition baseOffset    # 底座相对偏移
- long creationTick              # 创建时间

方法：
+ open()                         # 打开菜单
+ close()                        # 关闭菜单
+ update()                       # 更新位置（每tick调用）
+ getIconByEntity(Entity)        # 根据实体获取图标
+ isInteractable()               # 是否可交互
```

#### 1.2 FloatMenuManager.java - 菜单管理器

```
管理所有活跃菜单，提供全局访问

字段：
- Map<UUID, FloatMenu> activeMenus    # 玩家UUID -> 菜单
- Map<UUID, MenuHolder> holders       # 玩家持有者
- MenuUpdateTask updateTask           # 更新任务

方法：
+ createMenu(Player, MenuConfig)      # 创建菜单
+ getMenu(Player)                     # 获取玩家菜单
+ closeMenu(Player)                   # 关闭玩家菜单
+ closeAllMenus()                     # 关闭所有菜单
+ tick()                              # 全局tick更新
```

#### 1.3 MenuHolder.java - 玩家菜单持有者

```
存储玩家相关的菜单状态和数据

字段：
- Player player
- FloatMenu currentMenu
- IconEntity hoveredIcon              # 当前悬停的图标
- long lastInteractTick               # 上次交互时间
- boolean menuVisible                 # 菜单可见性
```

---

### 2. 实体层设计

#### 2.1 BasePlate.java - 底座

```
管理底座ItemDisplay实体

字段：
- ItemDisplay entity                  # 底座实体（无物品）
- RelativePosition offset             # 相对玩家偏移

方法：
+ spawn(Location)                     # 生成底座
+ updatePosition(Player)              # 更新位置
+ remove()                            # 移除
```

#### 2.2 IconEntity.java - 图标实体

```
职责：封装单个图标的Display实体

字段：
- ItemDisplay displayEntity           # 物品显示实体
- TextDisplay textDisplay             # 文字显示实体
- MenuIcon icon                       # 图标逻辑
- RelativePosition offset             # 相对底座偏移
- boolean isHovered                   # 是否被悬停
- float normalScale                   # 正常缩放
- float hoverScale                    # 悬停缩放（1.2）

方法：
+ spawn(Location)                     # 生成图标
+ updatePosition(Location)            # 更新位置
+ setHovered(boolean)                 # 设置悬停状态
+ showText()                          # 显示文字
+ hideText()                          # 隐藏文字
+ highlight()                         # 高亮效果
+ unhighlight()                       # 取消高亮
+ remove()                            # 移除
```

---

### 3. 坐标变换设计

#### 3.1 RelativePosition.java - 相对位置

```
职责：处理相对玩家的坐标计算

字段：
- double x                            # 水平横向偏移
- double y                            # 垂直偏移
- double z                            # 深度偏移

方法：
+ toAbsolute(Player)                  # 转换为绝对坐标
+ rotate(float yaw)                   # 根据玩家朝向旋转
+ clone()                             # 克隆
```

**坐标计算逻辑**：
玩家眼睛位置: eyeLoc = player.getEyeLocation()
玩家朝向: yaw = player.getLocation().getYaw()
底座绝对位置 = eyeLoc + RelativePosition.rotate(yaw)
图标绝对位置 = 底座绝对位置 + 图标偏移

---

### 4. 交互处理设计

#### 4.1 RaycastDetector.java - 射线检测

```
职责：检测玩家视线是否瞄准图标

方法：
+ detect(Player, List<IconEntity>)    # 检测瞄准的图标
  - 获取玩家视线方向
  - 遍历图标进行射线-包围盒检测
  - 返回最近的命中图标
```

#### 4.2 HoverHandler.java - 悬停处理

```
职责：处理图标悬停状态变化

方法：
+ handle(Player, IconEntity)          # 处理悬停
  - 如果图标变化：
    - 旧图标取消高亮
    - 新图标应用高亮（缩放1.2 + 发光）
    - 显示文字提示
```

#### 4.3 ClickHandler.java - 点击处理

```
职责：处理图标点击事件

方法：
+ handle(Player, IconEntity)          # 处理点击
  - 验证玩家状态
  - 执行图标动作
  - 播放反馈效果
```

---

### 5. 任务调度设计

#### 5.1 MenuUpdateTask.java - 菜单更新任务

```
职责：每tick更新所有活跃菜单

执行逻辑：
1. 遍历所有活跃菜单
2. 更新底座位置（跟随玩家移动，不跟随旋转）
3. 更新所有图标位置
4. 执行射线检测
5. 更新悬停状态
6. 检查菜单有效性（玩家是否有效）
```

---

### 6. 事件监听设计

| 监听器 | 事件 | 处理逻辑 |
|--------|------|----------|
| MenuTriggerListener | PlayerInteractEvent | 检测胡萝卜钓竿右键，触发菜单显示 |
| PlayerInteractListener | PlayerInteractEvent | 左键点击执行图标动作 |
| PlayerMovementListener | PlayerMoveEvent | 更新菜单位置（可选，主要靠tick任务） |
| PlayerStateListener | PlayerQuitEvent | 关闭菜单 |
| PlayerStateListener | PlayerChangedWorldEvent | 关闭菜单 |
| PlayerStateListener | PlayerDeathEvent | 关闭菜单 |

---

### 7. 数据流图

```
玩家右键胡萝卜钓竿
        │
        ▼
MenuTriggerListener 检测
        │
        ▼
FloatMenuManager.createMenu()
        │
        ├── DisplayEntityFactory 创建实体
        │       ├── BasePlate (ItemDisplay)
        │       ├── BackgroundBoard (TextDisplay)
        │       └── IconEntity[] (ItemDisplay + TextDisplay)
        │
        ▼
MenuUpdateTask 开始每tick更新
        │
        ├── 更新位置（不跟随旋转）
        ├── RaycastDetector 射线检测
        └── HoverHandler 更新悬停状态
                │
                ▼
        玩家左键点击图标
                │
                ▼
        ClickHandler 执行动作
                │
                ├── GiveItemAction
                ├── CloseMenuAction
                └── CommandAction
```

---

### 8. 实体标签与数据存储

```
实体标签：
- floatmenu_interact          # 可交互图标
- floatmenu_baseplate         # 底座
- floatmenu_background        # 背景板
- floatmenu_text              # 文字显示

PersistentDataContainer:
Key: floatmenu:menu_id        Value: 菜单UUID
Key: floatmenu:icon_id        Value: 图标ID
```

---

### 9. 关键实现要点

1. **位置更新策略**：每tick更新，使用 `Location.add()` 和三角函数计算旋转后的位置

2. **插值动画**：使用 `setInterpolationDelay(0)` 和 `setInterpolationDuration(4)` 实现平滑缩放

3. **性能优化**：
   - 使用标签筛选实体，避免全局搜索
   - 实体设置 `setPersistent(false)`
   - 玩家退出时立即清理

4. **线程安全**：所有实体操作在主线程执行

---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
---

## 附录

### paper entity.txt
Display entities
Added in 1.19.4, display entities are a powerful way to display various things in the world, like blocks, items and text.

By default, these entities have no hitbox, don’t move, make sounds or take damage, making them the perfect for all kinds of applications, like holograms.

Text can be displayed via a TextDisplay entity.

TextDisplay display = world.spawn(location, TextDisplay.class, entity - {
     customize the entity!
    entity.text(Component.text(Some awesome content, NamedTextColor.BLACK));
    entity.setBillboard(Display.Billboard.VERTICAL);  pivot only around the vertical axis
    entity.setBackgroundColor(Color.RED);  make the background red

     see the Display and TextDisplay Javadoc, there are many more options
});
Blocks can be displayed via a BlockDisplay entity.

BlockDisplay display = world.spawn(location, BlockDisplay.class, entity - {
     customize the entity!
    entity.setBlock(Material.GRASS_BLOCK.createBlockData());
});
Items can be displayed via an ItemDisplay entity.

Despite its name, an item display can also display blocks, with the difference being the position in the model - an item display has its position in the center, whereas a block display has its position in the corner of the block (this can be seen with the hitbox debug mode - F3+B).

ItemDisplay display = world.spawn(location, ItemDisplay.class, entity - {
     customize the entity!
    entity.setItemStack(ItemStack.of(Material.SKELETON_SKULL));
});
Displays can have an arbitrary affine transformation applied to them, allowing you to position and warp them as you choose in 3D space.

Transformations are applied to the display in this order

Diagram
The most basic transformation is scaling, let’s take a grass block and scale it up

world.spawn(location, BlockDisplay.class, entity - {
    entity.setBlock(Material.GRASS_BLOCK.createBlockData());
    entity.setTransformation(
        new Transformation(
                new Vector3f(),  no translation
                new AxisAngle4f(),  no left rotation
                new Vector3f(2, 2, 2),  scale up by a factor of 2 on all axes
                new AxisAngle4f()  no right rotation
        )
    );
     or set a raw transformation matrix from JOML
     entity.setTransformationMatrix(
             new Matrix4f()
                     .scale(2)  scale up by a factor of 2 on all axes
     );
});
Scaling example

You can also rotate it, let’s tip it on its corner

world.spawn(location, BlockDisplay.class, entity - {
    entity.setBlock(Material.GRASS_BLOCK.createBlockData());
    entity.setTransformation(
        new Transformation(
                new Vector3f(),  no translation
                new AxisAngle4f((float) -Math.toRadians(45), 1, 0, 0),  rotate -45 degrees on the X axis
                new Vector3f(2, 2, 2),  scale up by a factor of 2 on all axes
                new AxisAngle4f((float) Math.toRadians(45), 0, 0, 1)  rotate +45 degrees on the Z axis
        )
    );
     or set a raw transformation matrix from JOML
     entity.setTransformationMatrix(
             new Matrix4f()
                     .scale(2)  scale up by a factor of 2 on all axes
                     .rotateXYZ(
                             (float) Math.toRadians(360 - 45),  rotate -45 degrees on the X axis
                             0,
                             (float) Math.toRadians(45)  rotate +45 degrees on the Z axis
                     )
     );
});
Rotation example

Finally, we can also apply a translation transformation (position offset) to the display, for example

world.spawn(location, BlockDisplay.class, entity - {
    entity.setBlock(Material.GRASS_BLOCK.createBlockData());
    entity.setTransformation(
        new Transformation(
                new Vector3f(0.5F, 0.5F, 0.5F),  offset by half a block on all axes
                new AxisAngle4f((float) -Math.toRadians(45), 1, 0, 0),  rotate -45 degrees on the X axis
                new Vector3f(2, 2, 2),  scale up by a factor of 2 on all axes
                new AxisAngle4f((float) Math.toRadians(45), 0, 0, 1)  rotate +45 degrees on the Z axis
        )
    );
     or set a raw transformation matrix from JOML
     entity.setTransformationMatrix(
             new Matrix4f()
                     .translate(0.5F, 0.5F, 0.5F)  offset by half a block on all axes
                     .scale(2)  scale up by a factor of 2 on all axes
                     .rotateXYZ(
                             (float) Math.toRadians(360 - 45),  rotate -45 degrees on the X axis
                             0,
                             (float) Math.toRadians(45)  rotate +45 degrees on the Z axis
                     )
     );
});
Translation example

Transformations and teleports can be linearly interpolated by the client to create a smooth animation, switching from one transformationlocation to the next.

An example of this would be smoothly rotating a blockitemtext in-place. In conjunction with the Scheduler API, the animation can be restarted after it’s done, making the display spin indefinitely

ItemDisplay display = location.getWorld().spawn(location, ItemDisplay.class, entity - {
    entity.setItemStack(ItemStack.of(Material.GOLDEN_SWORD));
});

int duration = 5  20;  duration of half a revolution (5  20 ticks = 5 seconds)

Matrix4f mat = new Matrix4f().scale(0.5F);  scale to 0.5x - smaller item
Bukkit.getScheduler().runTaskTimer(plugin, task - {
    if (!display.isValid()) {  display was removed from the world, abort task
        task.cancel();
        return;
    }

    display.setTransformationMatrix(mat.rotateY(((float) Math.toRadians(180)) + 0.1F  prevent the client from interpolating in reverse ));
    display.setInterpolationDelay(0);  no delay to the interpolation
    display.setInterpolationDuration(duration);  set the duration of the interpolated rotation
}, 1  delay the initial transformation by one tick from display creation , duration);
Interpolation example

Similarly to the transformation interpolation, you may also want to interpolate the movement of the entire display entity between two points.

A similar effect may be achieved using an interpolated translation, however if you change other properties of the transformation, those too will be interpolated, which may or may not be what you want.

 new position will be 10 blocks higher
Location newLocation = display.getLocation().add(0, 10, 0);

display.setTeleportDuration(20  2);  the movement will take 2 seconds (1 second = 20 ticks)
display.teleport(newLocation);  perform the movement
Displays have many different use cases, ranging from stationary decoration to complex animation.

A popular, simple use case is to make a decoration that’s visible to only specific players, which can be achieved using standard entity API - Entity#setVisibleByDefault() and Player#showEntity() Player#hideEntity().

They can also be added as passengers to entities with the Entity#addPassenger() Entity#removePassenger() methods, useful for making styled name tags!

TextDisplay display = world.spawn(location, TextDisplay.class, entity - {
     ...

    entity.setVisibleByDefault(false);  hide it for everyone
    entity.setPersistent(false);  don't save the display, it's temporary
});

entity.addPassenger(display);  mount it on top of an entity's head
player.showEntity(plugin, display);  show it to a player
 ...
display.remove();  done with the display
