# `ip.yaml` 规范 v0.2.9

一个包一份 `ip.yaml`。它同时承担两件事：**描述这个包**（契约、旋钮、依赖、面积），以及在带 `instances:` 时**描述一次装配**。叶子 IP 与整颗 SoC 用的是同一份 schema，只差有没有 `instances:` 段。

元数据字段对齐 ECOS 的 `ip.yaml`；旋钮的约束语义取自 kconfiglib；依赖模型取自 cargo。三处借鉴各管一层，互不覆盖。

---

## 一、骨架

```yaml
name: gpio
version: 0.1.0          # 本包 semver
spec: "0.1"             # 遵循的本规范版本
kind: ip                # ip | library，省略即 ip

identity:   { ... }     # 身份与目录元数据，对齐 ECOS
contract:   { ... }     # 契约签名
params:     { ... }     # 数值旋钮
features:   { ... }     # 布尔与档位旋钮
constraints: [ ... ]    # 跨旋钮约束
guards:     [ ... ]     # 条件取值域
profiles:   { ... }     # 档位：一组默认值
chip:       { ... }     # 芯片级旋钮的初值（只有装配能写）
asic:       { ... }     # 可选。流片交付：顶层契约、payload 位、主频、ecc 预设
area:       { ... }     # 面积基线与模型
emit:       [ ... ]     # 交付形态
deps:       { ... }     # 依赖
test:       { ... }     # 可选。测试矩阵，省略即 auto
instances:  [ ... ]     # 可选。有它即为装配
```

---

## 一之二、包分两类

`kind` 省略时是 `ip`。

| `kind` | 会被例化 | 有契约 / 寄存器 / 面积 | 例 |
|:--|:--:|:--:|:--|
| **`ip`** | 是 | 是 | `gpio` `uart` `hart`；带 `instances:` 的即装配 |
| **`library`** | **否** | **否** | `hwcore` `amba` `bridge` |

```yaml
name: hwcore
kind: library
lang: bsv          # bsv | bh | verilog | vhdl | sv | chisel | spinal，省略即 bsv
```

**库包只贡献源码**：工具把它的源目录加进编译搜索路径，仅此而已。它不进地址图、不算面积、不出现在装配层次里。

**语言不限于 BSV。** 规范对源语言是开放的——Verilog、VHDL、SystemVerilog、Chisel、SpinalHDL 都可以，用 `lang:` 声明，工具按语言选对应的编译前端。**当前实现只支持 BSV 与 BH**，其余语言的前端等到真有包需要时再接。

我们自己写的 IP **在 BSV 与 BH 之间按 IP 择优**：看哪一种更适合这个 IP 的功能与特性，哪一种更易读、更好组件化与参数化。两者同一个编译器、同一套语义，只是语法不同。

择优的判据是**这个 IP 的主体是什么**：

| 主体形态 | 选 | 例 |
|:--|:--:|:--|
| 按拍推进的协议、规则与状态机 | BSV | `uart` `spi` `i2c` `emac` 的 RMII 两侧 |
| 引脚级总线时序 | BSV | `amba` 的 `Apb4` |
| 纯组合的映射，靠模式匹配分派 | BH | `hart` 的译码器 |
| 泛型容器与类型级参数化 | BH | `hwcore` 的 CAM |
| 变体靠**增删方程**（指令集扩展、协议族） | BH | 同上 |
| 变体靠**开关门控**一段硬件 | BSV | 各外设的特性开关 |
| 生成物（`regmap.yaml` 出来的寄存器组） | BSV | 生成器只发 BSV，人要看得懂 diff |

**一个包可以同时用两种语法**，此时 `lang` 写成序列：

```yaml
lang: [bsv, bh]
```

两种源码放**同一个源目录**（`hwsrc/`）。`bsc` 按扩展名区分（`.bsv` 是 BSV，`.bs` 是 BH），`import` 两边看不出差别，所以不分目录——分了只是让每处搜索路径都要写两遍。

因此库包**不得**出现 `contract` / `params` / `features` / `area` / `instances`，也不得有 `regmap.yaml`——写了不是无害的冗余，是误导读者以为它能被例化。工具对此报错。

`instances[].of` 指向库包同样报错。

> 工艺数据（`pdk`）不写 `ip.yaml`。它既不是 IP 也不是源码库，是**数据**，走专门的读取路径进层叠的最底层。

---

## 二、`identity` —— 对齐 ECOS

```yaml
identity:
  uid: ip-000042              # ECOS 目录编号；未收录时省略
  slug: gpio-tapeout-bsv
  display_name: Configurable GPIO
  summary: Bus-neutral GPIO with a measured area price on every feature.
  summary_zh: 总线中立的通用 IO，每个特性都带实测面积价格。
  category: peripheral
  subcategories: [gpio]
  ip_family: gpio
  compatibility: []
  integration_profile: [apb4-slave]
  maturity: simulated         # 见 maturity.md
  links:
    repository: https://github.com/Tape-Out/gpio
  upstream: { ... }           # 仅收编第三方时出现
  tracking: { ... }
```

字段名与取值域跟随 ECOS，**我们不改名**，这样 IP 将来能直接进它们的目录。注意 ECOS 目录的准入门槛是 silicon-proven，所以对齐是为了**将来能进**，不是现在就能进。

---

## 三、`contract` —— 契约签名

```yaml
contract:
  version: 1                  # 契约单独定版，与包的 version 无关
  ctrl:
    shape: flat               # flat | server | none
    aw: 8
    dw: 32
  data:
    - { name: tx, dir: out, kind: get, width: 8 }
  irq:
    - { name: irq,  kind: level }
    - { name: irqs, kind: level, width: harts }   # 多根：宽度可写旋钮名
  pins:
    - { name: gpio_in,  dir: in,  width: numPins }
    - { name: gpio_out, dir: out, width: numPins }
```

`shape` **是生成参数，不是包装层**。同一份逻辑按 `flat` 或 `server` 生成，两者代价差 2.55–3.19 倍；**规范不提供形态适配器**，因为实测适配比按目标形态重新生成贵 39.3%–152.2%。选形态的规则见 `contract.md`。

`shape: none` 说的是「这个 IP 不是总线从设备」——核就是。它的 CSR 空间是自己用的，引到顶层会让规则与外部方法抢同一个端口，而 bsc 的处置是把规则整条丢掉、只给一句告警。声明了 `none` 之后：中立顶层不出控制口，扁平化直接拒绝，装配也不把它放进地址图（它的 `regmap` 的 `base` 描述的是 CSR 空间，与片上地址无关）。

中断线的 `width` 让「每核一根」「每通道一根」表达得出来。不写就是单根。

契约版本与包版本分开（D24）：第三方只需盯契约版本，包内部改实现不打扰他们。

---

## 四、`params` 与 `features` —— 旋钮

### 数值旋钮

```yaml
params:
  numPins:
    type: int
    default: 32
    range: [1, 64]
    desc: 引脚数
```

### 布尔与档位旋钮

```yaml
features:
  irq:
    type: bool
    default: true
    area: { fixed: 0, per: numPins, k: 37.23 }

  debounce:
    type: bool
    default: false
    depends: [bidir]                  # kconfiglib 的 depends on
    area: { fixed: 29.7, per: numPins, k: 30.41 }

  drive:
    type: choice                      # kconfiglib 的 choice
    values: [none, two-level, four-level]
    default: none
    area:
      none:       { fixed: 0, per: numPins, k: 0 }
      two-level:  { fixed: 0, per: numPins, k: 8.0 }
      four-level: { fixed: 0, per: numPins, k: 16.0 }
```

**旋钮分两类，求解时机不同：**

| 类 | 例 | 求解 |
|:--|:--|:--|
| **实例级** | 位宽、中断、去抖、驱动强度 | 每次例化独立求解，**可以互斥** |
| **芯片级** | 复位风格、时钟树、全局总线后端 | 全局唯一，由顶层装配定 |

之所以敢让旋钮互斥——cargo 不敢——是因为硬件**每次例化各自展开**，不像 crate 全图只编译一次取并集。同一颗 SoC 里 `gpio` 一处八针、一处三十二针，各生成各的。

### 约束

```yaml
constraints:
  - when: { debounce: true }
    then: { bidir: true }
    msg: 去抖作用在输入通路上，需要 bidir 打开
```

借 kconfiglib 的 `depends on`、`choice`、`range`；**不借 `select`**。`select` 能强开一个依赖并不满足的符号，是 kconfig 公认最烂的部分。这里的 `constraints` 在不满足时**报错并指出是哪条**，不静默改值。


### 守卫：随别的字段变的取值域

约束回答「这组取值合不合法」，守卫回答「在当前这组取值下，这个字段还能填什么」。前者是事后判定，后者是事前收窄——网页表单里选了国家、省份下拉框跟着重填，是同一件事。

```yaml
guards:
  - when:   { xlen: 32 }
    narrow: { rvd: [false] }
    why: RV32 开 D 扩展时 FLen 推成 64，XLEN - FLen 变负，位拼接编不过
```

`when` 全部相等则守卫成立，把 `narrow` 里各字段的取值域收窄到所列的值。**`why` 必填**：说不出为什么的限制，过两个月没人敢动它。

守卫成立时按取值的来历分三种处置，与 `depends on` 同一套规矩：

| 这个值从哪来 | 怎么处置 |
|:--:|:--|
| 包默认、上层层叠 | **就地挪进允许的取值域**，并在 `config --why` 里标出是哪条守卫改的 |
| 实例 `with:` 或命令行 | **报错**，并把 `why` 原样说出来 |
| 矩阵派生的点 | 先修正联动字段；被试的那个字段本身不合法时才**不提供**，并说明理由 |

修正是会串的：一条守卫改了 A，可能让另一条守卫开始约束 B。求解一轮轮做到不动为止，成环则报错。

**点号键跨包**。装配可以收窄它装进来的那些包的旋钮，嵌套装配同样有效——子包管得了自己的字段，管不了「这颗 CPU 配成 32 位时那座桥只能是某几档」，那是装配才知道的事。

```yaml
guards:
  - when:   { cpu.xlen: 32 }
    narrow: { bridge0.dataWidth: [32] }
    why: 桥的数据通路比 XLEN 宽时高位恒零，白占面积
```

取值域归被指向的那个包管，装配这一侧只核到实例名为止。

**投影到 Kconfig**。守卫不是我们发明的语义，kconfig 原本就有，所以按它的写法发出去，`menuconfig` 看到的收窄与这里说的一致：

| 守卫 | Kconfig |
|:--:|:--|
| 布尔收窄成单值 | `depends on !(条件)` 加 `default … if (条件)` |
| 数值收窄成区间 | `range 低 高 if (条件)`，排在无条件 `range` 前面 |
| 档位排除若干档 | 被排除的那几档各自 `depends on !(条件)` |

kconfig 里看不见的符号照样取默认值，所以「不可选」与「只能是这个」本来就是一件事。判据也交给 `kconfiglib` 去判：生成的 Kconfig 读回来，问它在这组取值下那个符号还可不可选、有效区间是多少。

### 条件默认值

用 `generate.expose: all` 从上游的常量文件取旋钮时，同一个常量在不同模板里取值往往不同。这时它的默认值是条件的：哪份模板当选，就取哪份的值。

照搬先扫到的那份会出事。cva6 的 32 位模板走 hpdcache、要 `MemTidWidth = 4`，而 64 位那份写的是 2；把 2 套过去，写缓冲的 mem ID 编号不够用，展开当场 `$fatal`。这一类错在清单里看不出来，只有真去展开才会掉出来。


### 档位：一组默认值，不是另一种配置方式

外来的核常常随仓带来十几份现成配置，每份是一整个文件。**把「选哪份文件」做成一个维度，就会长出第二种配置方式**——一套设字段，一套换文件——于是每个旋钮都要回答「我现在起不起作用」，而答案取决于另一个旋钮。这条路走下去全是特例。

档位把它收回一种：模板永远只有我们生成的那一份，上游那些现成配置退化成**一组默认值**。

```yaml
params:
  cfg: { type: choice, values: [base, small, big], default: base }
profiles:
  by: cfg
  sets:
    small: { icacheByteSize: 2048, rvd: 0 }
    big:   { icacheByteSize: 32768 }
```

`by` 指一个已有的档位旋钮，`sets` 的每个键必须在它的取值域里，每个字段必须是已有的旋钮。档位落到**条件默认值**上——与 `generate.expose: all` 从多份模板读出来的那种是同一套东西，不另起一套机制。所以：

- 档位只改默认值，命令行与实例 `with:` 照样盖得过去；
- `config --why` 答得出「这个值来自哪一档」；
- 投影到 Kconfig 就是 `default … if (档位)`。

**档位必须是可核验的**：选了 `small`，生成出来的东西应当与上游那份 `small` 逐字段相同。接外来的核时这是一条真判据——cva6 的十二档就是这么逐档比过来的，比的是**字段的实际取值**（引用常量的先解开一层），不是文件文本。

### 旋钮源：常量与结构体字面量

`generate.syntax` 可以给一个列表，一份模板里两种写法一起扫：

| 写法 | 长什么样 |
|:--:|:--|
| `localparam` | `localparam Depth = 8;` |
| `parameter` | `parameter Depth = 8;` |
| `field` | 结构体字面量的一项：`Depth: unsigned'(8),` |

**只认字面量**。引用别的常量的那些项不暴露——它们是派生的，跟着源头走才对；把派生项也做成旋钮，改了源头反倒不动，那是错的。枚举（`config_pkg::WT`）认得出形状但猜不出有几档，所以只有清单在 `domain` 里给了取值域才暴露。

只扫常量往往不够。cva6 的十四份配置里，**差异有一半写在结构体字面量里**（`VLEN`、`NOCType`、`ZKN`、`SvnapotEn`），只读常量做出来的档位会是个名不副实的核。


### 芯片作用域：值沿实例树扩散

`xlen` 一颗芯片只该有一个说法；`aw` 则是同一个 IP 一处八位一处三十二位各配各的。D47 早就把旋钮分了实例级与芯片级两类，这里给它机制。

**照 CSS：不是所有属性都继承，而「继不继承」是属性自己的性质。**

```yaml
params:
  xlen: { type: choice, values: [32, 64], default: 64, scope: chip }   # 沿实例树向下扩散
  aw:   { type: choice, values: [8, 32], default: 8 }                  # 默认 local，不扩散
```

顶层装配给它们定初值，且**只有装配能写**——半路冒出来的全局变量正是「这个值谁定的」答不出来的那个坑：

```yaml
chip:
  xlen: 32
```

层叠次序只是在原有的基础上插一层，不改已有语义：

```
命令行 > 实例 with: > 装配 > 工作区 > 继承 > 包默认
```

**就近生效**：祖先扩散下来的值输给后代自己写的，但赢过包默认——否则继承永远不起作用。装配套装配时，孙子跟的是**中间层**解出来的值，不是顶层的，而中间层的改动不连累它的兄弟。

**`lock: true` 是 `!important`**，写在被继承的那一侧：后代就近覆盖时当场报错，并说清是哪一层在锁。

**投影到 Kconfig**：芯片级的旋钮在顶上各占一个符号，后代 `default` 到它身上——`menuconfig` 打开就是「父值自动取到、自己想改也改得了」。锁住的那些不给提示语，于是不可编辑但照样取默认值。判据交给 `kconfiglib`：改顶上的符号，后代要跟着变；改后代自己的，兄弟不能受影响。

**边界要说清**：这套只适用于**实例树**。

| 情形 | 适不适用 | 为什么 |
|:--|:--:|:--|
| 装配 → 实例、装配套装配 | 适用 | 就是一棵树，每个实例只有一个父 |
| 库包 | **不适用** | 库包从不被例化，树上根本没有它。它的旋钮是**测试期**的，装配真正用的宽度来自契约。硬给它一条扩散路径，来历就又断了 |
| 同一层里的多个总线段 | **不适用** | 继承只向下，表达不了「这几个兄弟共用一个值」。**这给 `segments:` 一条设计约束：段必须是树上的一个结点，不能是平级的标签** |
| NoC 网格 | 适用 | 网格是图，但它的配置是树上一个实例，拓扑在实例内部 |

---

## 五、`area` —— 面积价目表

```yaml
area:
  base:  { fixed: 37.6, per: numPins, k: 32.05 }
  model: linear-additive
  error: { bound: 0.07, sign: over }
  corner: { tool: ecc, pdk: ics55, freq_mhz: 100, measured: "2026-09-08",
            toolchain: "bsc 2026.01 · yosys 0.68+195 · ecc 0.1.0a11" }
```

价目表有**两种形态**，用在不同的旋钮上，各有各的保证：

| 形态 | 写法 | 用于 | 保证 |
|:--|:--|:--|:--|
| **仿射** | `{fixed, per, k}` | **特性**叠加 | 恒为高估 |
| **实测点** | `{points, per, margin}` | **参数**缩放 | 恒为高估，靠余量 |

```yaml
area:
  base:
    per: fifoDepth
    points: {1: 1689.24, 2: 1897.56, 4: 2532.60, 8: 3375.96, 16: 5000.52, 32: 8062.60}
    margin: 0.04          # 插值后上抬，由实测残差定
  model: measured-points
  error: { bound: 0.06, sign: over }
```

**为什么参数不能用仿射**：参数曲线可能有**结构断点**——`uart` 的 FIFO 在深度 3 以上换了实现，一条直线怎么拟都会把中间低估 4.2%。

**为什么存了点还要余量**：曲线是凹的，**弦恒在曲线之下**，所以点间线性插值系统性偏低（实测四个未测深度，三个低估，最多 3.54%）。加余量之后四个全部高估，最紧的只多 0.32%。

**`margin` 必须由实测定**。没测过就别写——写了是假的保证，比没有保证更危险。

**每个特性存「固定项 + 每单位斜率」两个数，不按 2ⁿ 组合存表。** 实测依据：三特性八组合 × 两位宽，线性叠加的偏差恒在 −0.6% ~ −6.9%，**符号一律为负**（特性共享逻辑被优化器合并）。

`error.sign: over` 明写模型**恒为高估**，因此预算工具只会偏保守，不会承诺不了。

`corner` 记录这组数是在什么工具、什么工艺、什么频率下量的。换任何一项，数就作废。`toolchain` 记的是**那几版工具**——只记版本号那一截，不记 git sha1 与编译日期，因为后两者变了数未必变。**本版只记不判**：历史数据没有版本可比，等全库重测一轮之后再把「版本变了即失效」变成门禁。它还必须带 `gen_digest`——**生成产物的摘要**。手工维护版本号两头不讨好：忘了升是漏报，升了没改输出是误报。对产物取摘要两种错都没有。

### 基线曲线、特性曲线与参数增量曲线（`points-additive`）

有多个参数与特性的包用这一种。预测等于**基线曲线 + 每个参数的增量曲线 + 每个开着的特性的曲线**，再乘 `1 + margin`：

```yaml
area:
  base:     { per: pins, points: { 8: 612.36, 16: 1183.84, 32: 2413.60, 64: 4522.84 } }
  params:
    funcs:  { points: { 2: -410.20, 4: 0.0, 8: 768.32, 16: 2234.96 } }
  margin: 0.008
  model: points-additive
features:
  padctl:
    area: { per: pins, points: { 8: 867.44, 16: 1497.44, 32: 2782.36, 64: 5830.72 } }
```

- **基线曲线**沿 `per` 指的那个参数走，其余参数取默认、特性全关。
- **参数增量曲线**是这个参数相对默认值的差，**默认值处必须为 0**，否则基线被算两次；工具对此报错。取值低于默认时增量可以为负。
- **特性曲线**是这个特性开与关的差，沿 `per` 指的参数走。
- 三种曲线出了实测格点：基线沿最近一段外推且不低于最近的实测点；增量曲线（参数与特性）一律取整条曲线的最大值——大规模下增量被综合噪声主导，下降的尾巴不是趋势。

`recal` 从 `measured` 里取行来算这几条曲线，**取行的配置是固定的**：基线与参数曲线要「特性全关、其余参数取默认」的行，特性曲线要「只开这一个特性（连同它依赖的特性）」的开关两行。缺哪一行就报错、不写回。

**`when` 只用在参数增量曲线上，意思是「这个参数的代价挂在某个特性上」**：特性关着时增量记 0，`recal` 也在那个特性开着时取行。`mbox` 的锁数就是这样——自旋锁关掉之后一把锁也不例化，曲线照加会高估三倍多。**不能拿它凑行**：参数的代价与特性无关时写 `when`，特性关着的配置会被少算——`pinmux` 为了让 `recal` 找到行给 `funcs` 加过 `when: padctl`，于是 `padctl` 关着、16 路功能的配置少算了整条 mux 的增量，正确做法是补量特性全关的那几行。

### 实测过的配置直接查表

扫描本来就要把整张网格量一遍，那就把这些数存下来：

```yaml
area:
  measured:
    - { at: { numPins: 8, irq: true, bidir: true, debounce: false }, um2: 646.52 }
    - { at: { numPins: 8, irq: false, bidir: false, debounce: false }, um2: 211.12 }
```

落在表里的配置**精确报出，不走模型也不抬余量**；表外才走模型。`gpio` 因此从 −14.58% 收到 +0.34%。

### 价目表量的是**中立顶层**，不是包装层

包装层含一个总线绑定器，而装配里整颗芯片只有一个。按包装层计价等于每个实例都多算一份绑定器，译码开销会算出**负数**。所以 `area` 描述的是 `build --neutral` 综合的那一层。

绑定器的钱记在实现它的包上（`amba`），交换网的钱记在 `hwcore`，**按包记一次**，不随实例数翻倍。库包因此可以有 `area`，但只能是定值——它没有旋钮可依。

### 叶子是上界，装配是估计

两层的承诺不同，写清楚免得误用：

| 层 | 承诺 | 怎么保证 |
|:--|:--|:--|
| 叶子 IP | **恒不低估** | 全部合法特性组合 × 多个参数值的证伪网格 |
| 装配 | **带双侧误差的估计** | 实测比例在 0.94~1.29 之间，取 1.15、界 ±15% |

装配的比例写在 `hwcore` 的 `area.assembly.factor`。为什么不能也做成上界：独立综合时端口必须保住、装配里综合器能并掉，而嵌套的握手又比独立边界贵，两股力方向相反，比例取决于配置的组合。硬要做上界就得留 30% 余量，那样的数字没法拿来比较配置。

**装配里不叠叶子的上界余量**——两道保守叠起来会到 26%。

### 证伪网格必须覆盖全部合法组合

只测「单特性」与「全开」会漏掉两两组合。`gpio` 的 `irq+bidir` 在 8 针处欠估 **9.57%**、32 针处又高估 4.68%——**符号两边都有**。所以「叠加恒为高估」那条只在包装层成立，换到 IP 本体就不成立。

### 关掉一个特性，面积必须真的掉下来

只挡逻辑不挡例化的开关一分钱都不省。`hart` 的 M 扩展开关原先只让译码器不产生乘法指令，模块照样例化——实测两侧只差 **98 µm²**，而那个单元本身要六千。改成按开关选一个退化实现之后，真实代价是 7,452。

**这是一条可执行的判据**：写完一个特性，量一次开与关，掉不下来就是没关掉。

---

## 六、`emit` —— 交付形态

照 Rust 的 `crate-type`：同一份源码，几种交付边界。

```yaml
emit:
  - kind: bsv                 # BSV 包，给 BSV 消费者
    package: GpioGen          # BSV 包名
    module: mkGpio            # 例化时调的模块
    config_type: GpioCfg      # 特性结构体的类型名
    interface: GpioIfc        # 接口类型名
    ctrl: regs                # 契约子接口叫什么，默认 regs
    ctrl_slow: slow           # 会停顿的那个控制口（可选，与 slow_when 成对）
    slow_when: sync           # 哪个特性开着时用会停顿的那个口
    pins:                     # 透传的子接口，装配与包装层照它生成
      - { name: pins, type: GpioPins, targs: [numPins] }
  - kind: verilog-flat        # 扁平端口顶层，可独立综合与独立流片
    bus: apb4                 # 必选：扁平化必须选一种总线
```

`pins` 里 `type` 为 `RegManager` 的那些是**发起口**（核的取指与访存、DMA 的搬运）。它们不引到装配顶层，而是接进交换网与外部总线一起排队——见下面「装配」一节。其余的原样透传到顶层。

`targs` 写旋钮名或数字，求解后取值。

`kind: bsv` 那四个字段**都是必填**：装配器要靠它们生成 `import` 与例化语句，光有 `kind` 生成不出东西。这一条是竖切逼出来的——规范先写漏了，写生成器时才发现。

**`ctrl_slow` 与 `slow_when` 管的是同一个 IP 的两副面孔。** 同步存储（SRAM 宏、ROM 宏、缓存）答不了同一拍，它的控制口是 `RegTarget` 而不是 `RegIf`；而同一个包在异步形态下仍然是 `RegIf`。两个口都在接口上不随配置变形，用哪个由 `slow_when` 指的那个特性决定：包装层按它选绑定器，装配按它决定这个实例进零等待那条向量还是会停顿那条。

两个键**必须成对**，且 `slow_when` 必须指向这个包真有的特性——只给一个，选口的条件就没了；指向不存在的特性，选出来的永远是同一个口而没人报错。**这两条都是硬错误，不是警告。**

如果没有 `ctrl_slow`，这个 IP 就只有一副面孔，一切照旧。

`verilog-flat` 是三件事的组合：**契约 + 选一种总线 + 扁平化**。它必须带 `bus`，因为外人不讲我们的契约，只讲 APB4 或 AXI。

### `kind: foreign` —— 别人的 RTL

我们自己只写 BSV 与 BH，但要接的东西是 Verilog、SystemVerilog、VHDL、Chisel 与 SpinalHDL 出来的网表。这些一律按**黑盒**收：不解析它的源码，只收一份声明，把它当成一个已知形状的组件放进装配。

```yaml
emit:
  - kind: foreign
    lang: verilog             # 与顶层 lang 同一张表：verilog | sv | vhdl | chisel | spinal
    top: uart_core            # 确切的顶层模块名，大小写照抄
    rtl: [rtl/uart_core.v, rtl/uart_rx.v]   # 综合视图，路径相对包根
    sim: [sim/uart_model.v]                 # 仿真视图，不写就同 rtl
    params:                   # 它的参数 <- 我们的旋钮，或一个字面值
      DATA_BITS: dataBits
      FIFO_DEPTH: 8
    clock: { port: clk }
    reset: { port: rst_n, active: low, sync: true }
    ports:
      - { endpoint: ctrl, kind: transaction, role: target, profile: apb4,
          prefix: p }                       # psel penable pwrite …
      - { endpoint: pins, kind: physical, type: UartPins,
          map: { tx: o_tx, rx: i_rx } }
      - { endpoint: irq, kind: event, role: source, map: { irq: o_irq } }
    limits:
      - 数据位固定 8
```

**每一项都是「接得上」必需的，缺一项就接不上**：顶层名与文件给出源码闭包；`params` 说清我们的旋钮投影到它的哪个参数；`clock` 与 `reset` 说清域与复位极性（写反了仿真能过、上板不工作）；`ports` 把它的端口名对到我们的端点上——整组协议用 `profile` 加 `prefix`，零散的线逐根 `map`；`limits` 写明它做不到什么。

**两份视图是有意分开的。** 仿真那份常带 `initial`、带延时、带只有仿真器认的模型，综合那份不能有。合成一份的代价是：要么仿真不准，要么综合报一堆看不懂的错。

**回执**：`ran build` 真正例化它时，要拿**展开之后的**端口表回来核对——声明里写的每个端口名与位宽都必须对得上。`ran check` 只做静态核对（文件在不在、`params` 的键有没有、端点形状全不全），因为那一步还没有工具去展开参数。

**反例**：把 `o_tx` 写成 `o_txd`，回执那一步必须报「黑盒 `uart_core` 没有端口 `o_txd`」并指到 `ip.yaml` 的那一行；`params` 里写一个它没有的参数，同样报错。两条都对不上还能构建，说明这份声明形同虚设。

**它不生成任何东西。** `kind: foreign` 的包没有我们生成的顶层，装配把它当既成事实接线。所以它也**不进价目表的预测**：面积只能实测，`area.base` 只许写定值，写曲线是假的。

**生成器类的上游**（先跑它自己的生成器，才有 RTL 或头文件）写 `setup: <任务名>`，指向 `tasks:` 里的那条任务。缺文件时报「它是生成物，先跑 `ran run <包> <任务>`」，**但不代跑**：任务不是隐藏的构建步骤。

#### 源码从哪来：五种写法

`rtl`（以及下面各视图里的 `rtl`）是一张条目表，每条是下面五种之一，可以混用：

```yaml
rtl:
  - rtl/core.sv                                        # 1. 一个文件
  - { path: rtl/fpu_dpi.sv, when: { fpu: DPI } }       # 2. 带条件的文件
  - { glob: "hw/rtl/**/*.sv", exclude: ["hw/rtl/afu/**"] }   # 3. glob
  - { regex: "^hw/rtl/(core|cache)/VX_.*\\.sv$" }      # 4. 正则，匹配相对包根的路径
  - { flist: core/Flist.cva6, env: { CVA6_REPO_DIR: third_party/cva6 } }   # 5. Flist
```

五种都可以带 `when:`，旋钮不满足时整条跳过。命令行 `--flist <文件>` 与第 5 种同义，可重复写，追加在清单条目之后（`-f` 已是 `--format`，不复用）。

**glob 与正则扫到什么就编什么**，不替用户挑「只编用到的」。扫多了是配置的事：把整个目录扫进来，用不上的文件照样要编得过。

**Flist 照工具惯例读**：认源文件、`+incdir+`、`+define+`、`-f`／`-F` 嵌套、`-y` 与 `+libext+`。`-f` 引用的相对路径以当前目录为基准，`-F` 以那份清单自己所在目录为基准。`${变量}` 由 `env:` 给值。Flist 里的 `+define+`、`+incdir+` 与清单的 `defines`、`includes` 冲突时**清单优先**，被覆盖的逐条写进回执。

**顺序**：展开出来的文件去重，由工具按 slang 算出的依赖把包排到前面，结果记进锁。slang 不在乎顺序，yosys 与 Verilator 在乎；**各工具拿的是同一份排好的清单**，不各排各的。

**反例**：
- glob 或正则一个文件都没匹配到：`XR-SRC-001`，error，指到那一条。
- Flist 里的 `${变量}` 在 `env:` 里没有值：`XR-SRC-002`，error。静默展开成空串会把路径接到根目录上。
- Flist 里的 `+define+X=1` 被清单的 `X: 2` 覆盖：`XR-SRC-003`，info，两个值都写进回执。

#### 视图：`rtl`、`sim`、`syn`

同一个包，综合与仿真常常要不同的宏、目录与文件：Vortex 综合要 `SYNTHESIS ASIC YOSYS` 加一个存储空壳库，仿真要 `SIMULATION SV_DPI` 加 DPI 头文件的目录。视图就是一份叠在基础配置上的覆盖：

```yaml
- kind: foreign
  top: Vortex
  rtl: [{ glob: "hw/rtl/**/*.sv", exclude: ["hw/rtl/afu/**"] }]
  defines: [VX_CFG_XLEN=32]            # 基础：三个视图共用
  includes: [hw/rtl]
  views:
    sim: { defines: [SIMULATION, SV_DPI], includes: [hw/dpi] }
    syn: { defines: [SYNTHESIS, ASIC, YOSYS], rtl: [{ glob: "hw/syn/libs/no_mem/*.v" }] }
```

- **名字只有三个**：`rtl` 是基础，`sim`、`syn` 叠在它上面。不许自造视图名，免得名字泛滥。
- **照层叠覆盖**：列表追加在基础之后；宏与基础同名时，视图里的值为准。要整份换掉，写 `replace: [rtl]`。
- **旧写法照读**：`sim: [文件…]` 等价于 `views: { sim: { rtl: [文件…], replace: [rtl] } }`，也就是「仿真视图，不写就同 rtl」原来的意思。
- **重流程只跑它需要的那一份**：综合与面积走 `syn`，仿真走 `sim`，**不跟旋钮做叉乘**。展开与回执是秒级，每个视图各跑一次无所谓；综合与全量仿真可能是几天，翻倍不行。

**反例**：写 `views: { fpga: … }`，`XR-VIEW-001`，error，并列出三个合法的名字。

#### 宏的投影：三种写法都有

`defines` 可以是列表（字面宏），也可以是表。表里每个宏由旋钮投影出来，**三种写法都支持，用户任选**：

```yaml
defines:
  VX_CFG_NUM_CORES: cores               # 值宏：-DVX_CFG_NUM_CORES=<cores>
  VX_CFG_L2_ENABLE: { when: l2 }        # 开关宏：l2 开着才定义，不带值
  VX_CFG_FPU_TYPE: fpu                  # 档位的值宏：-DVX_CFG_FPU_TYPE=DPI
  "VX_CFG_FPU_TYPE_{{fpu}}": true       # 名字带值的选择宏：-DVX_CFG_FPU_TYPE_DPI
  VX_CFG_TCU_IMPL: { from: tcu, map: { TFR: tfr_core, DSP: dsp_core, BHF: null } }   # 逐档映射
```

- **开关宏只能跟布尔旋钮**：`` `ifdef `` 判的是「定义没定义」，给它 `=0` 反而是打开。
- **名字带值的选择宏**复用 `{{…}}` 占位符，与 `tasks:`、`ran new` 同一套，不另起语法。它解决的是这一类上游：RTL 按 `` `ifdef VX_CFG_FPU_TYPE_DPI `` 选实现，只给值宏选不中。
- **逐档映射**给每一档一个宏值；`null` 表示这一档不定义。**每个合法档位都要写到**，漏一档就报错。
- 同一个档位旋钮可以同时出值宏与选择宏：上游两种都认时就两种都给。

**反例**：
- `{{…}}` 里写了不存在的旋钮，或者写的不是档位旋钮：`XR-MACRO-001`，error。
- 逐档映射漏了一个合法档位：`XR-MACRO-002`，error，列出漏的那几档。
- `{when: …}` 指向非布尔旋钮：error，这一条从前就有。

#### 回执的两层检查

回执拿展开之后的设计回来核对，除了上面的端口与参数，还有两层：

**通用检查**：顶层的非打包数组端口，长度解出 0 就是 `XR-RCPT-001`，error。「一个存储口都没有」这种配置会展开成功，Verilator 也收零长度数组，不拦就一路静默下去。

**探针**：旋钮若不落在顶层参数上（比如落在包的 localparam 上），光比顶层看不出它生效没有。写探针：

```yaml
receipt:
  - { symbol: "VX_gpu_pkg::NUM_SOCKETS", expect: { eq: "{{cores}}" } }
  - { symbol: "VX_gpu_pkg::VX_MEM_PORTS", expect: { ge: 1 } }
```

`expect` 认 `eq`、`ne`、`ge`、`le`、`in`，值可以用 `{{旋钮}}` 取当前配置。**只查通用检查不够**：打包端口写成 `[N-1:0]`，N 为 0 时是 `[-1:0]`，两位宽，看宽度发现不了；这类要靠探针。

**反例**：
- 用错了生成器的求值方式，存储口数成了 0：`XR-RCPT-001` 必须报；写了 `ge: 1` 的探针同时报 `XR-RCPT-002`。
- 上游把 `VX_MEM_PORTS` 改了名：`XR-RCPT-003`，error，「探针找不到符号」。探针失效时要明确报错，不许当成通过。

#### 编译单元

IEEE 1800 把「几个文件组成几个编译单元」留给工具定：iverilog 把全部文件并成一个，slang 默认一个文件一个。上游若把声明写在 `$unit` 层（文件里、模块外的 `parameter`），别的文件直接引用，就只在并成一个单元时编得过：

```yaml
- kind: foreign
  top: gpu_die
  unit: single                 # file（默认）| single
  rtl: [src/const.sv, src/core.sv, src/gpu_die.sv]
```

- `single` 时文件顺序有意义，`$unit` 层的声明要排在用它的文件前面。
- 这是语言语义的选择，不是诊断，所以不走 `diagnostics`。

**反例**：VeriGPU 不写 `unit: single`，回执报 `slang:UndeclaredIdentifier`，指到用 `data_width` 的那一行；写了就过。写 `unit: many`，清单校验报错并列出两个合法值。

#### 上游自己的测试

`test.upstream` 列上游自带的测试，矩阵的每一点都跑。两种写法：

```yaml
test:
  upstream:
    # 测试台：息壤用 iverilog 编译运行，参数按这一点的旋钮覆盖
    - { name: hello, files: [bench/tb.v, rtl/core.v], dut: tb,
        params: { width: w }, fixed: { memsize: 8192 },
        expect: "Hi", timeout: 120, when: { w: 1 } }
    # 任务：上游有自己的脚本（先汇编再仿真、cocotb、Makefile），退出码就是判据
    - { name: examples, task: examples, timeout: 600 }
```

- 测试台那一种，`name`、`files`、`dut` 必写；`expect` 给了就要在输出里找到它，只看「仿真结束了」不算过。
- 任务那一种只认 `name`、`task`、`timeout`、`when`，`task` 必须是 `tasks:` 里的一条。任务按这一点解出的配置填占位符。
- `when` 不满足的点跳过，并在报告里写明跳过的原因，不算通过。

**反例**：
- 任务那一种再写 `dut`：清单校验报错，两种写法不混。
- `task` 指向 `tasks:` 里没有的名字：清单校验报错。
- 任务退出码非零：这一点标红，报告里带退出码与输出末尾几行（`XR-TASK-004`）。

#### 本版实现到哪

规范一次写全，工具分档实现。**用到还没实现的写法，报 `XR-SPEC-001`「已声明，本版还不能构建」**，不报语法错：

| 写法 | 状态 |
|:--:|:--:|
| 单个文件、带 `when` 的文件、glob、正则、Flist 与 `--flist` · `sim` 列表与 `views` · `defines` 的五种写法 · `includes` · `setup` · `unit` · 回执（端口、参数、`XR-RCPT-001`、探针） · `slang:` 诊断与 `allow`／`deny` · `test.upstream` 的两种写法 | 已实现 |
| 文件按 slang 算出的依赖排序并记进锁 · 配置导入与补全 | 已声明，未实现 |

**扁平化不是免费的**：把综合边界推到契约层，对外线位实测从 475 涨到 762（+60%），多出来的全是方法变端口后的 `RDY`/`EN` 握手线。代价随 `contract.ctrl.shape` 变——`flat` 形态近乎恒等变换，`server` 形态要把两组握手都摊成端口。**这笔钱在配置时就该看得见**，所以它进价目表。

---

## 六之二、`targets` —— 怎么构建

`emit` 说的是**交付什么形态**，`targets` 说的是**怎么构建它**。两者不是一回事：`uart` 一个包既出寄存器组也出扁平端口顶层，是两个目标共用一份 `emit`。

```yaml
targets:
  regs: { driver: regs }
  flat: { driver: flat }
```

| driver | 做什么 |
|:--:|:--|
| `regs` | 从 `regmap.yaml` 生成寄存器组与一致性测试台 |
| `flat` | 生成扁平端口顶层，按 `emit` 里那一段选定的总线 |
| `bsv` | 只编这个包的 BSV 模块并跑它自己的测试台——没有寄存器图也不扁平化的那类 |
| `assembly` | 按 `instances` 生成顶层 |
| `library` | 只编源码，不例化；面积走 `area.probe` |
| `foreign` | 什么都不生成，黑盒是既成事实 |
| `none` | 没有可构建的东西 |

**不写就按树上有什么推断**，与从前一样。推断本身没问题，**把推断藏在别处才有问题**：在此之前这件事是流水线 `grep ip.yaml` 猜的——`kind: library` 归库、有 `instances:` 归装配、有 `verilog-flat` 归扁平、其余一律算「有寄存器图」。于是 `rvdbg` 这种只出 BSV 的调试模块落进最后一档，跑的是它根本没有的寄存器一致性测试。

**写出来的要对得上树上真有的东西**：说 `assembly` 就得真有 `instances`，说 `regs` 就得真有 `regmap.yaml`，说 `flat` 就得真有 `verilog-flat` 段。对不上当场报——**一句对不上的声明比猜还糟**，因为读的人会信它。

**工具要答得出**：`xirang inspect <包> -f json` 的 `targets` 字段出这张表，流水线问工具，不再自己猜。

---

## 六之三、`tasks` —— 清单里写命令

```yaml
tasks:
  vp: "perl scripts/gen.pl -o build/vp"
  smoke:
    run: bash htest/smoke.sh
    needs: [spinal]
    env: { CROSS: loongarch64-linux-gnu- }
    timeout: 5400
    desc: 开缓存跑一段小程序
```

- 一条任务写成一行命令，或带 `run` 的表；表里只认 `run`、`needs`、`env`、`cwd`、`desc`、`timeout`。名字以小写字母开头，只含小写字母、数字与短横，不能与内建阶段 `check`、`gen`、`build`、`test` 同名。
- `needs:` 只能指向内建阶段或别的任务，成环在检查期报错。
- `timeout` 是正整数秒，不写是 1800。`test.upstream` 那一条自己写了 `timeout` 时以它为准。超时连同任务起的子进程一起停，不留孤儿。
- **命令只读已解出的配置，不许回头改它**：只有下表这几种占位符，任务的输出不被解析成配置。
- 默认不进构建图，`ran build` 不替你跑，要 `ran run <包> <任务>`。工具会调用任务的只有两处：黑盒的 `setup`，与 `test.upstream` 的任务形式。

| 占位符 | 值 |
|:--:|:--:|
| `{{name}}` | 包名 |
| `{{root}}` | 包的根目录 |
| `{{out}}` | 这次的输出目录 |
| `{{knob.<旋钮>}}` | 这一点解出的取值，旋钮名照清单原样，驼峰也一样 |
| `{{defines}}` | 黑盒的宏投影成的 `-D` 串，用到才算 |
| `{{resolved}}` | 解出的全部取值落成的 JSON 文件的路径 |

占位符与宏投影、`ran new` 同一套 `{{…}}`，花括号内侧可以留空格。

**反例**：认不得的名字（`{{nope}}`）与写法不对的（`{{Name}}`、`{{knob.num-cores}}`）都报 `XR-TASK-003`，不原样留在命令里照跑；任务名与内建阶段同名、`needs` 成环、`timeout` 不是正整数，同样报错。

## 六之四、`asic` —— 流片交付

`targets` 说怎么构建，`asic` 说**这一颗怎么交出去**：顶层长什么样、payload 的每一位接谁、跑多快、走 ecc 的哪套流程。只有要流片的包写它，`ran asic <包>` 读它。本版收装配与黑盒，叶子 IP 先套一层装配。

```yaml
asic:
  top: to2610_switch        # 交付的模块名与文件名；省略则包名的 - 换成 _
  frame: mpc                # mpc：MPC-Frame 五口；none：设计的端口原样当芯片端口
  mhz: 50                   # 目标主频，进 ecc 的 frequency_mhz，由它生成 create_clock
  flow: rtl2gds             # ecc 预设：syn_sta | rtl2gds | harden | rcx，省略即 syn_sta
  pads:                     # payload 位从 0 起按序分配
    - sw0_pins_tx_0_*       # 端口名，可带 *；匹配到的按声明次序依次占位
    - { in: mdio0_pins_mdio_i, out: mdio0_pins_mdio_o, oe: mdio0_pins_mdio_oe }
    - { in: i2c0_pins_sda_i, oe: i2c0_pins_sda_pull }     # 开漏：oe 为 1 时拉低
    - { port: spis0_pins_miso, at: 60 }                   # at 钉起始位
  tie:                      # 不出芯片的输入接成常量
    sw0_pins_rx_*_rx_er: 0
  unused: [irqs]            # 不出芯片的输出，明写
```

`frame: none` 时不写 `pads`、`tie`、`unused`，时钟端口名写在 `clock`；`frame: mpc` 的时钟固定叫 `clock`，不写 `clock`。

### 五口顶层

`frame: mpc` 时息壤生成这一层，契约照 MPC-Frame 的 `docs/cn/io-map.md`：

```verilog
module to2610_switch (
  input  wire        clock,
  input  wire        reset,      // 高有效
  input  wire [65:0] io_in,
  output wire [65:0] io_out,
  output wire [65:0] io_oe
);
```

| 条目 | 每位占 | `io_out[n]` | `io_oe[n]` |
|:--:|:--:|:--:|:--:|
| 单向输入 | 一位 | 0 | 0 |
| 单向输出 | 一位 | 设计的输出 | 1 |
| 三态组 `{in, out, oe}` | 一位 | `out` | `oe` |
| 开漏 `{in, oe}` | 一位 | 0 | `oe` |
| 没分配的位 | — | 0 | 0 |

三态组里写了的几个端口位宽必须相同。设计的时钟与复位不进 `pads`：`clock` 直接接；`reset` 按设计的复位极性接，不再同步——FrameTop 的 `FrameDesignControl` 已按时钟放开复位（两拍 `release_count`），再同步一次是多余的——装配出的顶层是低有效的 `rst_n`，黑盒照 `emit` 里的 `reset.active`。**端口怎么排只看这里，不按名字猜**：gpio 叫 `gpio_dir`、spi 叫 `io_oe`、i2c 是开漏的 `scl_pull`，猜不齐。

### 展平

交付的是**一个 `.v`**：bsc 出的模块、bsc 的库模块、黑盒展开后的模块、五口顶层全在一个文件里。参数按解出的配置展开，层次保留；除顶层外的模块名一律加 `<top>_` 前缀，参数化出来的变体按原名排序编号——同一片 MPC 上的几颗设计都带着 bsc 的 `FIFO2`，不改名就撞。bsc 的 `initial` 块按 `BSV_NO_INITIAL_BLOCKS` 去掉，流片的触发器没有初值。

写完要自检：只有顶层不带前缀，yosys 只读这一个文件按顶层展开不缺模块。

### 连线闭合

除时钟与复位，设计的每个端口必须落在 `pads`、`tie`、`unused` 三处之一，且只落一处。**漏了就报错，不自动接地**：悄悄接地的输入在仿真里是一个永远不来的请求，在片上是一个永远不响的口。

### 报告

`ran asic` 写 `report.json`，流片说明从它生成，不手填：

| 节 | 内容 |
|:--:|:--:|
| `config` | 每个实例的每个旋钮：取值、赢的那一层、写在哪（与 `config --why` 同源） |
| `pads` | payload 位表、接成常量的输入、不出芯片的输出 |
| `sources` | 用到的每个包：版本、仓库、提交号、有没有未提交的改动；黑盒另列上游子模块的地址与提交号 |
| `toolchain` | bsc、yosys、sv2v、ecc、息壤的版本与 PDK 的位置 |
| `ecc` | 各步骤的状态与耗时、单元数、面积、最后一个出了时序的步骤的建立与保持裕量、功耗、签核清单 |
| `gate` | 下表几道检查的结果 |

### 检查号

| 号 | 默认 | 什么时候报 |
|:--:|:--:|:--|
| `XR-ASIC-001` | error | 要流片的包没有 `asic` 段 |
| `XR-ASIC-002` | error | payload 位不够，或两处分到同一位 |
| `XR-ASIC-003` | error | 有端口哪儿都没落 |
| `XR-ASIC-004` | error | 端口不存在、方向不对、落了两处、三态组位宽不一或没写 `oe`、常量放不进位宽 |
| `XR-ASIC-005` | error | 展平、自检或 ecc 没跑通 |
| `XR-ASIC-006` | error | 建立时间最差裕量为负 |
| `XR-ASIC-007` | error | 保持时间最差裕量为负 |

前四道在写任何文件之前报。时序那两道按诊断闸门放宽：`diagnostics: { XR-ASIC-006: warn }`，照跑并留在报告里。

## 七、`deps` —— 依赖

照搬 cargo 的来源模型：

```yaml
deps:
  hwcore: "^1"
  amba:   { git: "https://github.com/Tape-Out/amba", rev: "a1b2c3d" }
  imsic:  { path: "../imsic" }

patch:
  hwcore: { path: "../hwcore-fork" }
```

解析结果固化进 `xirang.lock`，而**锁跟着装配走，不跟着工作区走**：装配是交付物、锁是它的一部分，所以它住在那个仓里。

**叶子 IP 不带锁**。它是被别人依赖的库，钉死反而会跟使用它的装配冲突——照搬 cargo 对二进制与库的区分。`xirang lint` 据此分别对待：装配没锁是错，叶子没锁只是一句说明。

**一条硬规则**：同一依赖**不得同时以 git submodule 与 `git:` 出现**。是 submodule 就写成 `path:`——`.gitmodules` 管「目录从哪来」，`xirang.lock` 管「解析到哪个目录」，两者管的是不同的事，同时用才会变成两个真相源。

---

## 八、`instances` —— 装配（有它即为 SoC）

```yaml
name: soc-mcu
bus: apb4                     # 全局默认后端，实例可覆盖

instances:
  - name: gpio0
    of: gpio
    with: { numPins: 32, irq: true, drive: two-level }
    addr: 0x1000_0000         # 省略则用该 IP 的 regmap.yaml 里的 base
    irq_to: plic0.src[3]

  - name: uart0
    of: uart
    with: { profile: ns16550 }
    addr: 0x1000_1000
    bus: axi4-lite            # 覆盖全局默认

  - name: plic0
    of: plic
    with: { mode: plic, sources: 32 }
```

**装配是递归的**：`soc-smp` 含两个 `hart`，每个 `hart` 含一个 `imsic`，三层用同一套语义。因此不存在单独的「SoC schema」——`ip.yaml` 有 `instances:` 就是装配，没有就是叶子 IP。

地址重叠在展开期报错，不留到仿真。

**装配包不得含自有 RTL。** 长出自有 RTL 就说明下层缺件，该补的是下层，不是把胶水塞进装配包。

**`bus: none`**：片外没有总线口，片上总线只由片内的发起方驱动。无核的设备（交换机、转换器）靠 `spis` 这类 SPI 从口桥管理。片上一个发起方都没有时报错——地址图上的设备谁也访问不到。

---

## 八之二、`test` —— 测试矩阵

一个 IP 在默认配置下过了，不等于它在别的配置下也过。**门控写漏了的表现恰恰是「关掉了硬件还在」**，那只有把那一个开关单独关掉才看得见。所以门禁不跑一个点，跑一张矩阵。

```yaml
test:
  matrix: auto            # auto（默认）| full | none
  extra:                  # 额外要测的交叉点
    - {fifoDepth: 3, parity: true}
  skip:                   # 派生出来但不该测的
    - {fifoDepth: 64}
```

整段可省，省略等同 `matrix: auto`。

### `auto` 派生出哪些点

**线性覆盖，不做全组合**：

| 点 | 取值 |
|:--:|:--|
| 默认 | 全部旋钮取默认值 |
| 下界 | 布尔全关、档位取第一个、数值取 `range` 下界 |
| 上界 | 布尔全开、档位取最后一个、数值取 `range` 上界 |
| 每个布尔单独开 | 该开关连同它 `depends` 的一起打开，其余默认 |
| 每个布尔单独关 | 只关这一个，其余默认 |
| 每个档位的每个取值 | 各一个点 |
| 每个数值的两端 | `range` 上下界各一个点 |

点数随旋钮个数线性增长，不随组合数指数增长。**全组合是价目表的活**——价目表必须给出上界，非穷尽不可；测试要问的只是「每个门控真的生效吗」，那是线性的。要补交叉点就写进 `extra`。

`full` 展开成全组合，只给旋钮很少的包用；超过 64 点工具报错，让人改回 `auto` 并把真正在意的交叉点写进 `extra`。

### 库包也走矩阵

库包没有地址图、不是装配，但**旋钮反倒比 IP 更多**——总线的实现从头到尾按 `aw`/`dw` 写。所以库包一样有 `params`，一样按矩阵逐点跑 `htest/*Tb.bsv`；没有旋钮的库包只有一个点，与从前逐字节相同。

测试台要读得到这一点的取值：`htest/mk*.py` 每个点被调用一次，拿到 `{label, knobs}`，把位宽发成 BSV 的数值类型给测试台 `import`。

### 装配也走矩阵

装配自己没有旋钮，它的配置是各实例旋钮的乘积，全组合一定爆。所以装配写明要扫哪几条轴，每条轴照 `auto` 的规则取值：

```yaml
test:
  axes: [cpu.mul, cpu.smode, mem.words]   # 点号键指到实例的旋钮；装配套装配就接着点下去
  extra:
    - {mem.words: 4096, cpu.mul: false}
  skip:
    - {mem.words: 64}
```

- **不写 `axes` 只跑默认那一点**，与从前相同。`matrix: full` 在这几条轴上做全组合，同样以 64 点为限。
- 每一点的取值作为从根往下的覆盖进层叠，赢过实例的 `with:`，来源记作这一点的名字。
- 守卫照常生效：派生的点被排除就跳过并说明，手写的点被排除就报错。
- 每一点跑调度门禁与装配自己的 `htest/mk*.py`。生成脚本收到的 `knobs` 是**全部实例**的点号键（`{"cpu.mul": true, "mem.words": 256, …}`），认矩阵的照它改期望。**测试台逐字节相同也照跑**：被测的是这一点生成的 SoC，每一点都不一样。
- 点号键落不到实例上一律报错，`with:` 里的也一样，不静默丢掉。

**反例**：`axes` 或 `extra` 提到不存在的实例或旋钮，error；叶子写 `axes`，error，叶子的轴就是它全部的旋钮；把一条轴删掉，只属于它的点必须从报告里消失。

### 不合法的点

`auto` 派生的点若被约束判为不合法（例如上界把两个互斥开关同时打开），**跳过并在报告里说明**；`extra` 里手写的点不合法则**报错**——那是人写错了，不是派生的副产品。

守卫比约束先一步：派生的点踩到守卫时，先把联动字段修正进允许的取值域，修完能过就照跑，报告里记下改了什么。只有被试的那个字段本身被排除时才不提供。为一个联动字段把整条腿丢掉，等于因为「32 位下没有 D 扩展」就不测 32 位。

### 解析下来相同的点只跑一次

两个不同的覆盖可能解析出同一份取值（`depends` 把某个开关压回去时常常如此）。工具按解析后的取值去重，报告里注明与哪一点相同。

### 每个点跑什么

| 检查 | 看什么 |
|:--:|:--|
| 调度门禁 | 会不会卡：`G0004` `G0021` `G0006` `G0035` |
| 寄存器一致性 | 读写对不对：从 `regmap.yaml` 生成 |
| 各仓自己的行为测试 | 行为对不对：`htest/mk*.py` |
| 黑盒的展开与回执 | 这一组参数展不展得开，声明与展开后的设计对不对得上 |
| 上游自己的测试 | `test.upstream`，见六的「上游自己的测试」 |

`htest/mk*.py` 收两个参数：输出目录，以及这一点的 `{"label": ..., "knobs": {...}}`。**认矩阵的生成脚本照它改包名与期望，不认的照旧生成同一份**——生成物逐字节相同的点不重复跑，于是「这个测试台其实不看配置」在报告里是明写的，而不是悄悄浪费。

### 声明的范围是承诺

`range` 写到哪，就得能建到哪。上线当天扫全库逮到两条从没人建过的上界：一个 MAC 的帧缓冲区地址写死了位宽，实际只撑得住清单说的一半；一个中断控制器的源数上限其实由寄存器图定死，比清单说的少四倍。两个都过了前面所有门禁。**承诺兑现不了就改小承诺，不许留着。**

---

## 八之三、`diagnostics` —— 诊断闸门

每一道检查有一个**稳定检查号**与一个**默认严重级**。文案可以改，号不改；写脚本、写豁免、在群里说「我又撞上那个」，指的都是号。

```
noshow < trace < debug < info < warn < error
```

**只有 `error` 挡住动作。** `warn` 及以下照跑，并留在报告里。

```yaml
diagnostics:
  XR-AREA-003: info      # 旋钮没计价：不挡仿真
  XR-WIRE-002: error     # 引脚没处置：挡
```

**四处可以改，就近生效**：工作区 → 装配 → 包 → 单条链接。`ran config --why` 要答得出这一级是从哪一层来的——与旋钮取值同一套追溯，不另起炉灶。

### 号段与默认级

| 号 | 拦下什么 | 默认 |
|:--:|:--|:--:|
| `XR-WIRE-001` | 引脚驱进来的值，模块里一次也没读 | `error` |
| `XR-WIRE-002` | 引脚四选一没处置：连接、导出、接成常量、声明不用 | `error` |
| `XR-REG-001` | 寄存器接口暴露的方法，实现里没有用到 | `error` |
| `XR-ADDR-001` | 某个合法配置下两个寄存器占同一个地址 | `error` |
| `XR-SCHED-001` | 生成 Verilog 时 `bsc` 报出的规则冲突 | `error` |
| `XR-LOCK-001` | 清单、源码与锁对不上 | `error` |
| `XR-GEN-001` | 生成物与源描述漂移 | `error` |
| `XR-CONV-001` | 位宽转换 | `error` |
| `XR-CONV-002` | 突发或原子性被摊平 | `error` |
| `XR-CDC-001` | 两端时钟域不同而没写 `cdc` | `error` |
| `XR-SRC-001` | glob 或正则一个文件都没匹配到 | `error` |
| `XR-SRC-002` | Flist 里的 `${变量}` 没有值 | `error` |
| `XR-SRC-003` | Flist 的宏或目录被清单覆盖 | `info` |
| `XR-VIEW-001` | 视图名不是 `rtl`、`sim`、`syn` | `error` |
| `XR-MACRO-001` | `{{…}}` 指向不存在的旋钮，或不是档位旋钮 | `error` |
| `XR-MACRO-002` | 逐档映射漏了合法档位 | `error` |
| `XR-RCPT-001` | 顶层非打包数组端口长度为 0 | `error` |
| `XR-RCPT-002` | 探针的期望不满足 | `error` |
| `XR-RCPT-003` | 探针找不到符号 | `error` |
| `XR-DIAG-001` | 放宽了一条前端放不宽的诊断 | `error` |
| `XR-CFG-001` | 导入的值不在取值域里 | `error` |
| `XR-CFG-002` | 导入的键不认识 | `warn` |
| `XR-CFG-003` | 有新旋钮要回答，但不在终端里 | `error` |
| `XR-SPEC-001` | 用了规范里有、本版工具还没实现的写法 | `error` |
| `slang:<诊断名>` | 外来 RTL 的语言问题，名字照 slang | `error` |
| `XR-AREA-001` | 价目表量的是另一份生成产物 | `info` |
| `XR-AREA-002` | 参数改了，量出来的面积不变 | `info` |
| `XR-AREA-003` | 改这个旋钮，面积预测不动 | `info` |
| `XR-AREA-004` | 预测面积低于实测 | `info` |
| `XR-AREA-005` | 标了价，却没有一行实测打开过这个特性 | `info` |
| `XR-AREA-006` | 没有价目表 | `info` |

**面积那一族默认是 `info`，不是 `error`。** 没有综合流程的人要能跑行为仿真；两块积木拼出来的 USB 转串口没有价目表，照样该是一等公民。

### 转换要用户知情

位宽与突发转换**默认报错**，链接上写 `allow` 才放行，且留在报告里；`warn` 居中。照 Rust 的 `allow` / `warn` / `deny`：**不许用户不知情就悄悄转换，也不拦知情的自动转换。**

```yaml
links:
  - from: seg_axi
    to: seg_apb
    allow: [XR-CONV-001]      # 64 位下挂 32 位，知道自己在做什么
```

### 外来 RTL 的语言兼容

外来 RTL 不一定完全合 IEEE 1800。前端是 slang，它报的每一条错误按 `slang:<诊断名>` 进闸门，默认 `error`。包级照 Rust 的 `#[allow]`、`#[deny]` 放宽或收紧：

```yaml
diagnostics:
  allow: [slang:UsedBeforeDeclared, slang:SysFuncHierarchicalNotAllowed]
```

`allow` 等于 `info`（照跑，留在报告里），`deny` 等于 `error`，`warn` 居中；也可以像上面那样逐条写级别。

- **放宽由工具翻译成前端的兼容开关**，开了哪些写进回执：`UsedBeforeDeclared` 对应 slang 的 `--allow-use-before-declare`，`SysFuncHierarchicalNotAllowed` 与 `ConstEvalHierarchicalName` 对应 `--allow-hierarchical-const`，`UnknownModule` 对应 `--ignore-unknown-modules`（厂商原语如 Xilinx 的 `xpm_*` 没有源码，当黑盒只核顶层）。对应表是数据，新加一条不改结构。
- **开关作用于这个包的整次展开**，放不到其中几个文件上。回执本来就一个包一个包地展开，所以放宽不会漏到别的包。
- **后果由声明的人承担。本组织自己的包默认最严**：除特殊情况并写明理由外，不放宽。
- **前端没有开关的诊断放不宽**：写了报 `XR-DIAG-001`。「声明放宽了、实际仍然报错」比不声明更让人糊涂。

**反例**：
- 没打补丁的 Vortex 不写 `allow`：报 `slang:UsedBeforeDeclared` 并指到源文件的行。写了 `allow`：照跑，回执里列出 `--allow-use-before-declare`。
- `allow: [slang:UndeclaredIdentifier]`：slang 没有放过未声明标识符的开关，报 `XR-DIAG-001`。

### 不许把「没量」说成「过了」

没量出来的报 `unknown`，与「量了且在预算内」是两种结果。面积、IPC、频率、时钟树、单条指令最长耗时这几项**先有测法，后有阈值**：测法没落地之前它们只出 `unknown`，不配 `error`。给还测不准的量配上 `error`，门禁看着完整，实际上是在骗人。

**反例**：把某个包的价目表整个删掉，`ran test` 必须照跑并在报告里标 `XR-AREA-006 info`；把 `XR-AREA-006` 在工作区改成 `error`，同一条命令必须当场挡住。两次行为一样，说明这套覆盖没生效。

---

## 九、层叠与来源追溯

一个旋钮的最终取值由六层叠出来，低到高：

1. BSV 类型默认（**结构由 BSV 类型权威，只有取值层叠**）
2. `pdk` 包提供的工艺事实
3. 本 `ip.yaml` 的默认
4. workspace 默认
5. 装配处 `instances[].with`
6. 命令行

**约束求解可以改写任何一层的取值**，但那是「被约束强制」，不是「被上层覆盖」——两者必须能区分，否则用户以为自己写错了。

工具必须能对任意一个旋钮回答五问：**最终值 · 谁定的（层/文件/行）· 被压掉的候选及其来源 · 是否被约束强制 · 这个取值花了多少面积**。

**生成钩子不得改写旋钮取值**，只能往下游添产物与声明。理由不是洁癖：钩子一旦能回改配置，上面那五问就答不了了。

---

## 九之二、配置导入与补全

配置要能像 Linux 的 defconfig 一样：拿来一份，改其中一部分，余下的按默认补全，再存成一份小文件分享出去。四个命令照 kconfiglib 取名：

| 命令 | 做什么 |
|:--:|:--|
| `ran config import <文件> [--format …]` | 读入一份配置，作为一层取值 |
| `ran config olddefconfig` | 没给值的旋钮按默认补全；不认识的键丢掉，并逐条报告 |
| `ran config oldconfig` | 新出现的旋钮逐个问。不在终端里时报 `XR-CFG-003` 并列出要答的全部，不挂着等输入 |
| `ran config savedefconfig -o <文件>` | 只存与默认值不同的旋钮，次序固定 |

- **导入的配置落在第 4 层**（workspace 默认）。导入多份时后导入的为准。`config --why` 要答得出某个值来自哪份文件的哪一行。
- **格式按读入器扩展**：我们自己的（`savedefconfig` 出的 YAML）、kconfig 的 `.config`、上游格式。上游格式的**第一个是 Vortex 的 `VX_config.toml`**：取字面值的键；`expr:` 开头的是由别的项算出来的，跳过并记 info。新增一种格式只加一个读入器。
- **往返一致**：对任何合法配置 `x`，`import(savedefconfig(x))` 解出来与 `x` 相同。

**反例**：
- 导入文件里某个值不在取值域：`XR-CFG-001`，error，指到文件与行。
- 导入文件里有不认识的键：`XR-CFG-002`，warn；`olddefconfig` 丢掉它并列出。
- 把 `savedefconfig` 出的文件里一个值改回默认值，再存一次，那一行必须消失。

## 九之三、`dt` —— 在设备树里怎么露面

`ran export -f dts` 从解出的装配出设备树源，`-f svd` 出 CMSIS-SVD，`-f h` 出整颗芯片的 C 头。三者读同一张表：每个实例的地址、大小、它的包在设备树里叫什么。包在清单里写 `dt:` 说后一件：

```yaml
dt:
  node: cpu                        # 节点名，按设备树规范的通用名：cpu、memory、serial、gpio……
  compatible: [riscv]              # 追加在 "tape-out,<包名>" 之后
  size: { knob: words, scale: 4 }  # 存储报实际装了多少；不写就用装配给的地址窗口
  props:
  - { name: riscv,isa-base, value: rv32i }
  - { name: mmu-type, value: riscv,sv32, when: { mmu: true } }
  - { name: mmu-type, value: riscv,none, when: { mmu: false } }
```

- 不写 `dt:` 的包，节点名就是包名，`compatible` 只有 `tape-out,<包名>`。`node: cpu` 的实例进 `cpus`，`node: memory` 的进根节点，其余进 `soc`（`simple-bus`）。
- `props` 按写的次序出，`when` 不满足的跳过；`value` 缺省是空属性（如 `interrupt-controller`）。
- **中断本版不导**：今天的中断连线写在 `connect:` 里是 BSV 表达式，从那里猜的中断号不可信，等连线结构化之后再补。
- SVD 与 C 头只收有寄存器图、且有总线地址的实例；`shape: none` 的包（`rvcore` 的 CSR 空间）不进。

**反例**：
- `dt` 里写了不认识的键：清单校验报错，列出认得的四个。
- 存储节点照译码窗口出大小：`soc-mcu` 的 256 字存储会报成 64 KiB，Linux 会去用不存在的内存——`size` 必须给出实际大小。
- 属性写在子节点之后：dtc 拒收；生成器保证属性在前。

---

## 十、本版明确不做

注册表（registry）· 并行 DAG 与增量构建 · 跨包的芯片级旋钮传播（cmake 那种 target 属性传递）· 版本求解的回溯（先只支持精确锁）· `instances` 的条件展开（`for` / `if`）。

这些每一条都有真实用途，但 `gpio` + `soc-mcu` 这条竖切一条都用不上。**等真有 IP 需要，再按各自来源的原名加进来。**
