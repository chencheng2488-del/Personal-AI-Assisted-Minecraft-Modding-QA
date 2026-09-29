# Infinity What Mod —— 问答学习记录

> 记录规则：从 2026-09-30 起持续记录问答。当用户说"终止记录"时停止整篇记录；说"这一段不记录"时跳过当段内容。

---

## 2026-09-30

### Q1：`ModBlockEntities.java` 是干什么的？教程说的每个 mod 都要搞的"注册"是什么？

**A：**

#### 1. 什么是"注册（Registration）"

Minecraft **不会自动扫描你写的类**。游戏内部为每一类东西维护着一张全局表（注册表 Registry），表里是"字符串 ID → 对象"的映射，例如：

- 方块注册表：`infwhat:inf_fluid_generator` → 那个 Block 实例
- 物品注册表：`infwhat:inf_fluid_generator` → 对应的 BlockItem
- 方块实体类型注册表：`infwhat:inf_fluid_generator` → BlockEntityType

你 `new` 出来的对象必须在游戏加载的指定时机放进这张表，否则游戏根本不知道它存在。原因是游戏处处用 ID 引用内容：

- 存档里写的是 ID（坐标 (x,y,z) 的方块是 `infwhat:inf_fluid_generator`）；
- 客户端与服务端网络同步传的是 ID；
- `/give` 指令、`assets/infwhat/models/...`、语言文件、战利品表等 JSON 全部按 ID 找对象。

没注册就 `new` 的方块：放不进世界、存档时崩溃或丢失，属于"黑户"。

#### 2. 1.20.1 Forge 的注册方式：DeferredRegister（延迟注册）

自己抓时机注册容易早于/晚于游戏加载，所以官方模式是"先排队，到点统一注册"：

1. `DeferredRegister.create(某种注册表, modid)` —— 建一个队列；
2. `队列.register("名字", () -> new XXX())` —— 往队列里放一个"以后再创建"的供应商，返回 `RegistryObject<T>` 包装（没注册前不能 `.get()`）；
3. mod 主类构造时调用 `队列.register(modEventBus)` —— 把队列挂到 mod 事件总线；
4. 游戏加载到注册阶段时触发事件，Forge 按正确顺序调用这些供应商，完成真正注册。

#### 3. 本文件逐行解读

注意：**被注册的不是方块实体（BlockEntity）实例本身**。世界里每放置一个方块，BE 实例由 `Block#newBlockEntity()` 现场 new 一个，成千上万、各不相同，不可能注册。注册的是 **BlockEntityType（方块实体类型）**，它是一张"类型说明书"：

- 工厂方法：这种方块的 BE 怎么创建（`InfFluidGeneratorBlockEntity::new`）；
- 绑定哪些方块：哪种方块允许挂这个 BE（`ModBlocks.INF_FLUID_GENERATOR.get()`）。

```java
// 第 12-13 行：在"方块实体类型"注册表上建队列，归属 modid = infwhat
DeferredRegister<BlockEntityType<?>> BLOCK_ENTITIES =
        DeferredRegister.create(ForgeRegistries.BLOCK_ENTITY_TYPES, InfinityWhat.MOD_ID);

// 第 15-20 行：注册一个类型，ID = infwhat:inf_fluid_generator
RegistryObject<BlockEntityType<InfFluidGeneratorBlockEntity>> INF_FLUID_GENERATOR =
        BLOCK_ENTITIES.register("inf_fluid_generator", () ->
                BlockEntityType.Builder.of(
                        InfFluidGeneratorBlockEntity::new,        // 工厂：BE 的构造器引用
                        ModBlocks.INF_FLUID_GENERATOR.get()       // 绑定的方块（此时才get，时序安全）
                ).build(null));                                   // null = 不提供存档升级用的 DataFixer

// 第 29-31 行：主类 InfinityWhat 调用，把队列挂到 mod 事件总线
public static void register(IEventBus eventBus) {
    BLOCK_ENTITIES.register(eventBus);
}
```

#### 4. 一张表理解本 mod 的四套注册

| 文件 | 注册进哪张表 | 类比 |
|---|---|---|
| `ModBlocks` | 方块表 | 给游戏增加一个"方块品种" |
| `ModItems` | 物品表 | 给它一个能拿在手里/放进背包的物品形态 |
| `ModBlockEntities` | 方块实体类型表 | 给方块配"档案与出生证"：挂哪个 BE 类、如何创建 |
| `ModMenuTypes` | 菜单类型表 | 给 GUI 容器一个网络同步用的类型 ID |

这就是为什么所有教程都要写注册——它是**样板代码（boilerplate）**，每个加内容的 mod 都逃不掉，但不含玩法逻辑。真正的玩法在 BlockEntity 里。

---

### Q2：有一个现成的同版本同加载器的 `ModBlockEntities.java`，就能完全复制粘贴吗？

**A：骨架可以复制，内容必须改。** 同 MC 版本（1.20.1）+ 同加载器（Forge 47.x）保证的是 **API 签名不变**——`DeferredRegister.create(...)`、`Builder.of(...).build(null)` 这套写法所有 mod 通用；但文件里凡是"属于具体 mod"的部分都得替换：

**必须改的 4 处：**

1. **`package` 声明**——换成自己的包路径；
2. **modid**——`DeferredRegister.create(..., 别人的MOD_ID)` 改成自己的（如本项目的 `InfinityWhat.MOD_ID`，值为 `infwhat`），否则注册进别人的命名空间，资源全对不上；
3. **注册条目的类引用**——构造器方法引用（`InfFluidGeneratorBlockEntity::new`）和绑定方块（`ModBlocks.INF_FLUID_GENERATOR.get()`）必须指向**你自己已写好的类**，类不存在直接编译不过；
4. **条目数量和 ID 字符串**——对方有 2 个方块实体你就写 2 条，ID 名字自己定（`"inf_fluid_generator"` 等）。

**不用改的：** 整体结构、方法调用写法、`.build(null)` 的 null（DataFixer，绝大多数 mod 用不到）、`register(IEventBus)` 方法。

**两个容易漏的配套点（不在本文件里）：**

- 主类构造器里必须调用 `ModBlockEntities.register(modEventBus)`，复制了文件但忘了挂事件总线 = 注册不生效；
- BlockEntity **服务端 tick**（如本项目物品发生器每 8 tick 推送）不在这里注册，而是在 Block 类的 `getTicker()` 里绑定。

**反面情况（什么时候连骨架都不能抄）：**

- 版本不同：1.20.1（Forge）→ 1.21（NeoForge）时包名和注册 API 有迁移；Fabric 完全是另一套（`BlockEntityType.Builder` 类似但事件模型不同）；
- 对方的 BlockEntity 构造器签名特殊（带额外参数），`::new` 方法引用就对不上。

**一句话总结：** 注册类是"填空题"不是"抄送件"——格式 100% 可复用，空格里必须填自己的类名、方块名、modid。

---

### Q3：是不是不用深究原理，只要知道"不能没有它、空格自己填、换版本/加载器换骨架"？具体要写的地方有没有清单？

**A：对，这个阶段这样理解完全正确。** 注册代码是"上车先系安全带"式的存在——刚开始不需要研究织带工艺，知道必须系、卡扣对准就行。

#### 清单 A：`ModBlockEntities.java` 必填的 6 个空（以本项目为例）

| # | 位置 | 本项目的值 | 填错的后果 |
|---|---|---|---|
| 1 | `package` 行 | `com.bandybus.infwhat.block.entity` | 编译不过 |
| 2 | `import` 的自己的类 ×N | 主类、BE 类、Block 类 | 编译不过 |
| 3 | `DeferredRegister.create(注册表, ①)` 的 ① | `InfinityWhat.MOD_ID`（= `infwhat`） | 注册进错误命名空间，资源/存档全乱 |
| 4 | 每条目的 ID 字符串 | `"inf_fluid_generator"` / `"inf_item_generator"` | 必须与方块 ID 对应，建议直接照抄 ModBlocks 里的名字 |
| 5 | 每条目的 `::new` | `InfFluidGeneratorBlockEntity::new` | BE 创建不出来 |
| 6 | 每条目绑定的方块 | `ModBlocks.INF_FLUID_GENERATOR.get()` | BE 与方块挂不上，放方块不生成 BE |

固定不动的部分：`DeferredRegister`/`RegistryObject`/`Builder.of(...).build(null)` 的写法、`register(IEventBus)` 整个方法。

#### 清单 B：整个 mod 的"注册四件套 + 主类接线"总清单

新增一个带 GUI 的方块实体机器时，按顺序填：

- [ ] `ModBlocks`：加 1 条方块注册（名字、方块属性：硬度/声音/材质）
- [ ] `ModItems`：BlockItem 通常由 ModBlocks 内联动注册（本项目就是这样），确认堆叠数等属性
- [ ] 写好 `XxxBlock` 类（`newBlockEntity`、需要 tick 就写 `getTicker`、右键 `use`）
- [ ] 写好 `XxxBlockEntity` 类（构造器签名必须是 `(BlockPos, BlockState)`）
- [ ] **`ModBlockEntities`：加 1 条（清单 A 的 6 个空）**
- [ ] 有 GUI → 写 `XxxMenu` + `XxxScreen`，在 `ModMenuTypes` 加 1 条，并在客户端事件类注册 Screen
- [ ] **主类构造器里确认每个 `Xxx.register(modEventBus)` 都调用了**（漏一个就整套不生效）
- [ ] 资源：lang 中英文、textures 贴图；blockstate/模型/配方/战利品表/标签走 DataGen

#### 清单 C：什么时候必须换"骨架"（而不只是填空）

- MC 大版本变了（1.20.1 → 1.21）；
- 加载器变了（Forge → NeoForge / Fabric）；
- 判据：抄来的代码第一行 `import net.minecraftforge...` 就报红 = 骨架过期，去对应版本的官方 MDK 示例拿新骨架，再把自己的空填回去。

---

### Q4：项目的文件结构为什么这样安排？各自什么功能？是否必须？

**A：结构由"两套规则"决定，理解这两条就不用死记：**

1. **Java 代码区（`src/main/java/...`）：布局自由，按功能分包是社区惯例。** 只要 `package` 与目录一致、类在 classpath 里，放哪 Forge 都能加载。本项目用的是"按功能分包（package by feature）"：block 的东西归 block，client 的东西归 client。
2. **资源区（`src/main/resources/` 与 `generated/resources/`）：路径是 Minecraft 用死的约定，一个目录都不能乱放。** 游戏按固定路径查找：`assets/<modid>/lang/`、`assets/<modid>/textures/block/`、`data/<modid>/recipes/`……放错 = 游戏找不到 = 紫黑块/英文键/没配方。

#### 一、工程根目录（构建与工程元数据）

| 文件/目录 | 功能 | 是否必须 |
|---|---|---|
| `build.gradle` / `settings.gradle` / `gradle.properties` | 构建脚本、工程名、MC/Forge/mod 版本属性 | **必须** |
| `gradlew` / `gradlew.bat` / `gradle/wrapper/` | Gradle Wrapper，保证任何人用同一 Gradle 版本构建 | **必须**（随模板带，勿手改 jar） |
| `src/main/templates/META-INF/mods.toml` | mod 元数据（id、版本、作者、依赖），构建时替换变量 | **必须，没有它游戏不认这是 mod** |
| `src/main/templates/pack.mcmeta` | 资源包格式声明（1.20.1 = pack_format 15） | **必须** |
| `.github/workflows/build.yml` | GitHub Actions 自动构建 | 可选，删了不影响开发 |
| `README.md` / `TEMPLATE_LICENSE.txt` | 文档与许可证 | 可选（发布时建议有） |
| `.build-target-props.json` | 生成平台残留 | **可删** |

#### 二、Java 包 `com.bandybus.infwhat`（布局可自由，下表是本项目的选择）

| 包/类 | 功能 | 是否必须 |
|---|---|---|
| `InfinityWhat.java` | `@Mod` 入口，构造器里调各注册类 + 配置 | **必须有一个**（类名任意） |
| `block/ModBlocks.java` + 两个 Block 类 | 方块品种注册；Block 是薄壳：右键开 GUI、创建 BE、破坏掉落 | 加方块则**必须** |
| `block/entity/ModBlockEntities.java` + 两个 BE 类 | BE 类型注册；**玩法大脑**：存数据、tick、Capability、NBT | 需要存数据/每刻运行才**必须**（纯装饰方块不需要） |
| `item/ModItems.java` | 物品注册队列 | 本项目**实际为空**——BlockItem 在 ModBlocks 里联动注册了；保留队列是惯例，方便以后加独立物品 |
| `menu/`（2 Menu + ModMenuTypes） | 服务端容器逻辑：槽位、Shift 点击、桶交互；菜单类型注册 | 有 GUI 才**必须** |
| `client/screen/`（2 Screen） | 客户端画 GUI（贴图、液槽、文字） | 有 GUI 才必须，**只在客户端加载** |
| `client/ModClientEvents.java` | 把 MenuType 与 Screen 绑定（`MenuScreens.register`） | 有 GUI 才必须；`Dist.CLIENT` 保证服务端不碰客户端类 |
| `config/InfinityWhatConfig.java` | 无限开关、有限容量、流体黑名单（ForgeConfigSpec） | 可选，不需要配置文件可删 |
| `event/ModCommonEvents.java` | 把两个方块挂进创造模式"功能方块"物品栏 | 可选但强烈建议，否则玩家只能 `/give` 获取 |
| `datagen/`（5 个类） | 开发期运行 `runData` 生成 JSON，运行游戏时不参与逻辑 | **可选的开发工具**；不写它就手写 JSON |

#### 三、资源区（路径写死，modid 段 = `infwhat`）

| 路径 | 功能 | 是否必须 |
|---|---|---|
| `assets/infwhat/lang/en_us.json`、`zh_cn.json` | 翻译键 → 显示名 | 必须有（缺了游戏内显示 `container.infwhat.xxx` 原始键） |
| `assets/infwhat/textures/block/*.png` | 方块贴图 | 有方块必须 |
| `assets/infwhat/textures/gui/container/*.png` | GUI 背景 | 有 GUI 必须 |
| `assets/infwhat/textures/gui/*_screenshot.png` | 验收截图残留，游戏不引用 | **可删** |
| `assets/infwhat/blockstates/*.json`（generated） | 方块状态 → 用哪个模型 | 有方块必须（可手写可生成） |
| `assets/infwhat/models/...`（generated） | 方块/物品模型形状 | 必须（同上） |
| `data/infwhat/recipes/*.json`（generated） | 合成配方 | 没有则无法合成（可接受则可不要） |
| `data/infwhat/loot_tables/...`（generated） | 挖方块掉什么 | 必须有，否则挖了不掉落 |
| `data/minecraft/tags/blocks/mineable/pickaxe.json` | 归类为"镐子可挖" | 可选（影响挖掘速度与掉落条件） |
| `data/infwhat/advancements/...`（generated） | 获得进度/配方书解锁 | 可选 |

#### 四、为什么 client 要单独分包？

Minecraft 分**物理服务端**和**客户端**，服务端没有 GUI 渲染类。凡是 `net.minecraft.client.*` 的代码若在服务端被加载会直接 `ClassNotFoundException` 崩服。所以 Forge 约定：客户端专属代码放 `client/` 包，用 `@Mod.EventBusSubscriber(value = Dist.CLIENT)` 隔离。这不是洁癖，是物理隔离要求。

#### 五、一句话记住

- **最小可运行 mod** = `mods.toml` + `pack.mcmeta` + 一个 `@Mod` 主类 + 至少一个注册类 + lang 文件；
- **本项目多出的部分**全是为两件事服务：能玩（block/entity/menu/config/event）+ 能看（client/resources）+ 开发便利（datagen）。

---

### Q5：Gradle 到底是什么？和 Maven 什么关系？为什么 MC mod 开发都用它？国内为什么这么麻烦？

**A：**

#### 1. Gradle / Maven 是"构建工具 + 依赖管家"

它们解决同一类问题：

- **依赖管理**：你代码里 `import net.minecraftforge...`，这些 jar 从哪来？构建工具按坐标（如 `net.minecraftforge:forge:1.20.1-47.4.10`）自动从远程仓库（Maven 仓库）下载并放进编译 classpath；
- **构建流程自动化**：编译 Java → 处理资源（替换 mods.toml 变量）→ 打成 jar → 跑测试；
- **可复用插件**：别人把复杂流程封装成插件，你一行 `id 'xxx'` 引入。

两者是**竞品**：Maven 用 XML（`pom.xml`）描述，约定固定、死板；Gradle 用 Groovy/Kotlin 脚本（`build.gradle`），灵活、可编程，是 Android 官方构建工具。

#### 2. 为什么 MC mod 圈几乎只用 Gradle

不是 Gradle 比 Maven 好多少，而是 **Forge/Fabric/NeoForge 官方只提供 Gradle 插件**：

- Forge 老插件：ForgeGradle（FG）；本项目：`net.neoforged.moddev.legacyforge`（ModDevGradle 的 Forge 兼容版）；
- Fabric：`fabric-loom`；NeoForge：ModDevGradle。

这些插件干的是 Maven 普通流程干不了的 MC 专属脏活：

1. 下载原版 client/server jar 与混淆映射（mappings/parchment）；
2. **反编译 + 重映射** Minecraft（把混淆名 `a()b()c()` 还原成 `getBlockState()` 等可读名），你才能写出能编译的 mod；
3. 配置 `runClient`/`runData` 等运行项；
4. 处理 AccessTransformer、混开发环境的 jar 重映射（reobf）等。

Maven 没有官方等价插件，技术上有人硬搞过但全社区工具链、教程、MDK 模板都默认 Gradle，**没有选择的必要**。

#### 3. 本项目里 Gradle 的关键部件

- `gradle/wrapper/gradle-wrapper.properties`：指定 Gradle 版本，`gradlew.bat` 会自动下载该版本——**不依赖你本机装没装 Gradle**；
- `build.gradle`：构建脚本（插件、仓库、依赖、runData 输出目录等）；
- `gradle.properties`：版本变量与 JVM 参数；
- 首次构建最慢：下载 Gradle 本体 → 下载 MC/Forge → 反编译（日志里 `createMinecraftArtifacts`、`neoformruntime` 阶段）；之后有缓存（`C:\Users\<你>\.gradle\caches`）秒级完成。

#### 4. 国内为什么麻烦

所有默认源都在境外且部分被墙/抖动：

| 下载内容 | 默认源 |
|---|---|
| Gradle 本体 | `services.gradle.org` / `downloads.gradle.org` |
| 插件与普通库 | `repo.maven.apache.org`（Maven Central）、`plugins.gradle.org`、Google Maven |
| Minecraft 本体与元数据 | `launchermeta.mojang.com`、`piston-data.mojang.com`（Mojang CDN） |
| Forge / NeoForge | `maven.minecraftforge.net`、`maven.neoforged.net` |

Maven Central 与 Gradle 本体有国内镜像；**Mojang 和 Forge 的 maven 基本没有稳定国内镜像**，这是麻烦的根源。

#### 5. 国内加速方案

**(1) Gradle 本体：手动下载或用镜像**
编辑 `gradle/wrapper/gradle-wrapper.properties`，把 `distributionUrl` 换成腾讯云镜像：
`https://mirrors.cloud.tencent.com/gradle/gradle-8.14.4-bin.zip`（版本号照原文件）。或用代理下载对应 zip 放进 `C:\Users\<你>\.gradle\wrapper\dists\...` 让 wrapper 校验复用。

**(2) 普通 Maven 依赖：配置全局 init 脚本（推荐，一劳永逸，不动项目文件）**
在 `C:\Users\<你>\.gradle\init.gradle` 写入：

```groovy
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        maven { url 'https://maven.aliyun.com/repository/google' }
    }
}
settingsEvaluated { settings ->
    settings.pluginManagement {
        repositories {
            maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
            maven { url 'https://maven.aliyun.com/repository/public' }
            gradlePluginPortal()
            mavenCentral()
        }
    }
}
```

**(3) Mojang / Forge maven：没有镜像时**
- 给 Gradle 配代理（在 `gradle.properties` 追加）：
  `systemProp.http.proxyHost=127.0.0.1` / `systemProp.http.proxyPort=端口` / `systemProp.https.proxyHost=...` / `systemProp.https.proxyPort=...`；
- 或开全局 TUN/系统代理后构建；
- 这部分**只需成功一次**，之后全部走本地 `.gradle/caches`，离线也能构建（`--offline`）。

**(4) 验证缓存是否就绪**：本项目首次构建已成功跑过一遍，说明本机缓存已齐全；之后断网也基本可用，不要再随便删 `~/.gradle/caches`。

#### 6. 一句话总结

Gradle 对 mod 开发者而言是"**带 MC 专属插件的依赖下载器+编译器+启动器**"；国内 80% 的痛苦在首次下载，对策是"能镜像的走阿里云/腾讯云，镜像不了的 Mojang/Forge 走一次代理，之后靠缓存"。

---

### Q6：现在还需要再下载什么吗？代码里的 `extends` 和"方块注册"是一回事吗？

**A：**

#### 1. 还需要下载吗？—— 不需要了

首次 `compileJava` 已成功，意味着：Gradle 本体、Forge/MC 依赖、反编译产物全部进了 `C:\Users\Mccoy\.gradle\caches`，项目的 `build/` 下也有产物。以后：

- 日常改代码、编译、`runClient` 都不再重复下载；
- 只有三种情况要重新下载：① 改了 `gradle.properties` 里的 MC/Forge 版本；② 手动删了 `.gradle/caches` 或 `build/` 里的 moddev 产物（删了会重新走一遍流水线）；③ 在 build.gradle 里新加了第三方 mod 依赖。
- 运行游戏本身（`runClient`）首次会下载原版资源文件（音效、语言等，走 Mojang CDN），那是游戏资产不是构建依赖，属于另一回事。

#### 2. "方块注册"和 `extends` 是两件完全不同的事

代码里有两个容易混淆的动作：

**动作 A：写一个类 `extends BaseEntityBlock`（Java 继承）——回答"这个方块是什么、会怎么反应"**

`extends` 是 Java 语言的继承，跟 Forge 无关。原版提供了 `Block`、`BaseEntityBlock` 等基类，里面已经写好了方块作为方块该有的一切（碰撞箱、渲染、状态管理……）。你继承它，只需**覆写自己关心的几个方法**：

```java
public class InfFluidGeneratorBlock extends BaseEntityBlock {
    public BlockEntity newBlockEntity(...)      // 这里放方块时创建哪个 BE
    public RenderShape getRenderShape(...)     // 用普通模型渲染
    public InteractionResult use(...)          // 右键时打开 GUI
    // playerWillDestroy(...)（物品发生器覆写）    // 破坏时把模板物品掉出来
}
```

不继承、从零实现一个方块要重写成百上千个方法，继承 = 站在原版类肩膀上只改差异。
- 普通方块：`extends Block`；
- 带方块实体的机器：`extends BaseEntityBlock`（它是 Block 的子类，多了 BE 相关契约）；
- 特殊需求再选别的父类（如 `HorizontalDirectionalBlock` 带朝向）。

**动作 B：`ModBlocks` 里的方块注册 ——回答"怎么让游戏知道这个类、用什么 ID"**

`new InfFluidGeneratorBlock(properties)` 只是在内存里 new 了一个普通 Java 对象，游戏不认。必须通过 DeferredRegister 把它登记进方块注册表并赋予 ID `infwhat:inf_fluid_generator`：

```java
INF_FLUID_GENERATOR = registerBlock("inf_fluid_generator",
    () -> new InfFluidGeneratorBlock(BlockBehaviour.Properties.of()
        .mapColor(MapColor.METAL).strength(3.5F, 10.0F)
        .sound(SoundType.METAL).requiresCorrectToolForDrops()));
```

注意这里还顺带传了 `Properties`：硬度、颜色、音效、挖掘工具要求——**方块的"属性"不是靠覆写方法，而是靠 Properties 链式配置**。

#### 3. 两者的关系（一句话）

- `extends BaseEntityBlock` = **定义**一种新方块（写它的行为代码）；
- `DeferredRegister.register(...)` = **登记**这种方块（给它 ID、放进游戏全局表）；
- 同一个注册条目还联动 `new BlockItem(...)`，让它拥有物品形态；
- 再配合 `ModBlockEntities` 把"这种方块"和"哪种 BlockEntity"绑定。

三者合起来，一个带 GUI 的机器方块才真正"活"在游戏里。

---

### Q7：开发时需要时刻拿着 API 表/方块属性手册吗？为什么没见过成体系的 MC mod 开发指南？

**A：不需要背 API，但确实要接受一个事实：MC mod 开发没有 Spring 那种成体系官方教程，知识是"文档 + 源码 + 范例"三件套拼出来的。专业开发者也不背，他们靠 IDE 现查。**

#### 1. 为什么没有成体系指南（先消除焦虑）

- Minecraft/Forge 不是商业公司维护的稳定 API，每个大版本都在改包名、改签名；大部头书出版即过时；
- Forge 官方文档只覆盖"骨架概念"（注册、事件、BE、Capability、DataGen、网络包等），不是 API 字典；
- 最权威的"API 说明"其实是**带注释的源码本身**。

#### 2. 真正的开发姿势：三个不用翻书的查法

**(1) 本项目已经配了 Parchment 映射**（gradle.properties 里的 parchment 配置，build.gradle 也开了 `downloadSources = true`）。所以在 IDE 里：

- **Ctrl/左键点任意原版类/方法名**（如 `BlockBehaviour.Properties`、`BaseEntityBlock`）→ 直接跳进反编译但带可读参数名和注释的源码；
- 鼠标悬停方法 → 看 JavaDoc；
- 在 `.of()` 后打个 `.` → 自动补全列出**全部**可链式调用的属性，比任何手册全：`strength`、`sound`、`mapColor`、`lightLevel`、`noOcclusion`、`randomTicks`、`requiresCorrectToolForDrops`、`instabreak`、`friction`、`speedFactor`……

**(2) 常用方块属性不用背，记住分类即可：**

| 类别 | 代表方法 |
|---|---|
| 物理属性 | `strength(硬度, 抗爆)`、`friction`、`speedFactor`、`jumpFactor`、`noOcclusion` |
| 外观 | `mapColor`、`sound`、`lightLevel(s -> 15)`、`ignitedByLava` |
| 掉落/工具 | `requiresCorrectToolForDrops()`、`noLootTable()`、`instabreak()` |
| 行为开关 | `randomTicks()`、`dynamicShape()`、`isRedstoneConductor`、`isSuffocating` |

**(3) 不知道某个玩法怎么实现 → 抄同类开源 mod**：GitHub 搜 `1.20.1 forge blockentity` 等关键词，看别人怎么调 API；这是圈内最主要的学习方式（注意许可证，只学写法）。

#### 3. 值得收藏的资源（按优先级）

1. **Forge 官方文档（1.20.1 专版，与本项目完全对应）**：https://docs.minecraftforge.net/en/1.20.1/ —— 重点读 Concepts（blocks、blockentities、items、capabilities、datagen、events、network）；
2. **IDE 里的原版+Forge 源码**（Ctrl+点击），最权威；
3. NeoForge 文档 https://docs.neoforged.net/ —— 只讲新版，概念解释比 Forge 文档清楚，很多章节可对照理解；
4. Forge 官方 GitHub 的测试 mod（MinecraftForge 仓库 `src/test`）——各种功能的官方最小示例；
5. 中文：B 站搜 "Forge 1.20.1 教程" 系列视频入门；MCBBS/论坛存档贴；注意 **mcmod.cn（MC百科）是 mod 内容百科，不是开发文档**，别搜错地方；
6. 提问：Forge Forums、NeoForged Discord。

#### 4. 推荐学习顺序（在这个项目上）

1. 先把本项目当黑盒跑通 runClient，能放方块、开 GUI；
2. 改一个小属性（如 `strength`）→ runClient 看效果，建立"Properties 是配置"的手感；
3. 照 Q3 清单新增一个最简单方块（无 BE），补 lang/贴图/DataGen；
4. 再读 InfFluidGeneratorBlockEntity 的 fill/drain，理解 Capability；
5. 遇到具体 API 再 Ctrl+点源码或查文档——**按需检索，不要前置背诵**。
