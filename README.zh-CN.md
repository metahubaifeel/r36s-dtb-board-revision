# R36S 刷完黑屏：怎么找到属于你这块主板的 dtb

> **一句话结论：** R36S 的屏幕参数写在一个 `.dtb`（设备树）文件里，它躺在 TF 卡的第一个 FAT32 分区（`BOOT` 分区）上。引导程序只会加载 `boot.ini`（或 `extlinux/extlinux.conf`）里点名的那一个文件。如果这个文件描述的屏幕型号和你机器上实际装的那块屏对不上，屏幕就永远锁不上信号，结果是黑屏——没有任何报错。修法通常是**用电脑把正确的 `.dtb` 放回 `BOOT` 分区**，而**不需要重刷整个镜像**。

这篇文章里出现的每一个文件名、每一个时序数字、每一条菜单文字，都是**从真实文件里读出来的**：一份社区 R36S 刷机包、包里那个 Windows 版 `dtb_selector` 工具内部的 `.dtb` 二进制、以及社区固件项目的公开源码。可以复现的命令都附了真实输出。我们没能验证的地方，一律明确标注 **⚠️ 未实测**。

---

## 一、现象：大家原话是怎么说的

以下引文来自中文掌机交流 QQ 群，**只标群名**。原始数据集里发言人无法可靠归属到每条消息，所以不写昵称。

> `[r36s（开源掌机）交流1群]` 大佬们 刚买的新机，要自己刷机dtb试了很多次都黑屏。看群文件也没找到v30的相关信息。

> `[r36s（开源掌机）交流1群]` 现在进来就是想找v30 dtb 试过几个固件都没专用的就来这里找找看了

> `[开源掌机中文社区⑤群]` 主要是我找不到能用的镜像文件 ／ 没有对应的dtb ／ 找不到dtb ／ dtb没选对吧 ／ 有没有这个板子能用的镜像

> `[开源掌机中文社区⑤群]` 你这个没有带 dtb ？ ／ `rk3326-r36s-sauce04-linux.dtb` ／ 如何替换dtb.jpg

> `[r36s（开源掌机）交流1群]` v30要刷最新的arkos4clone，已经解决wifi问题了

> `[r36s（开源掌机）交流1群]` 这个ARKOS刷机文件3.6G的里面带游戏吗
> （答案：不带，见第七节。）

> `[R36S掌机爱好者]` 群友在群里贴出的 dtb 检测工具输出，原文照抄：
> ```
> ⚠️ ⚠️ 部分匹配（屏幕参数一致，MD5不同）
> 🔋 你的dtb中电池设计容量为: 3380
> 🔋 你的dtb中电池最大容量为: 3718
>
> 检测到屏幕参数一致但MD5不同，可以尝试以下设备选项：
> 运行 dtb_selector 后依次尝试以下选项，选择兼容性最好的：
> • Game Console → Game Console R36s Panel 4
> • Game Console → Game Console R36XX
> • Game Console → Game Console R36H
> • Game Console → Game Console O30S
> ```

这段输出里有两点值得抠出来。第一，存在一个工具会告诉你**「屏幕参数一致，MD5 不同」**——说明用来辨认屏幕的参数就存在 dtb 自己里面。第二，兜底列表里出现了 **R36XX、R36H、O30S**，这些是**别的机器**，不是 R36S 的屏幕版本。这就是这个问题本来的形状：你要拿一个小文件去匹配一大家子长得很像的板子。

---

## 二、`.dtb` 到底是什么，为什么差一点就不行

R36S 用的是瑞芯微 **RK3326**。ARM Linux 的内核不像 PC 那样自己去探测硬件，它是**被一个设备树文件（device tree blob）告知**硬件长什么样的。设备树由文本（`.dts`）经 `dtc` 编译成扁平二进制，引导程序把它交给内核。某个节点缺失或写错，内核照样能跑起来，只是那块硬件不工作——不报错、不崩溃、不响。

这正是为什么 dtb 不对的表现是**黑屏，而不是开机失败**。我们检查的那份社区刷机包里，`dtb_selector` 工具内部一共 **85 个文件，其中 40 个是 `.dtb`**，分布在 `R36s、G80C、G80XF、R36Max、R36Pro、Mymini、U8、Xf35H、Xf40H` 九个板型目录下。选错了，内核照样启动，只是拿着错误的屏幕描述在跑。

### 差异就出在屏幕描述上

包里针对**原版 R36S** 有四个文件，标着 Panel 1 到 Panel 4。我们把这些文件解析开了：

| 版本 | dtb 里写着的 `model` 字符串 | 行 前肩/同步/后肩 | 场 前肩/同步/后肩 | 行总数 | 场总数 | 刷新率 |
|---|---|---|---|---|---|---|
| Panel 1 | `Game Console R35S/R36S fix by AeolusUX` | 80 / 20 / 80 | 20 / 4 / 12 | 820 | 516 | 68.5 Hz |
| Panel 2 | `Game Console R35S/R36S fix by AeolusUX` | 46 / 2 / 44 | 16 / 2 / 14 | 732 | 512 | 77.4 Hz |
| Panel 3 | `Game Console R35S/R36S fix by AeolusUX` | 46 / 2 / 44 | 16 / 2 / 14 | 732 | 512 | 77.4 Hz |
| Panel 4 | `Game Console R35S/R36S fix by lxc` | 46 / 2 / 44 | 165 / 2 / 14 | 732 | 661 | 59.9 Hz |

这张表要再读一遍，因为陷阱全在里面：

- **四个文件的分辨率完全一样**——640 × 480（`hactive` = 640，`vactive` = 480）。
- **至少 Panel 1 和 Panel 4 的屏控制器字符串也完全一样**——两者都声明 `elida,kd35t133` 和 `simple-panel-dsi`。
- **像素时钟完全一样**——四个都是 29.000 MHz。
- **只有消隐（blanking）区间不同**，于是行/场总数变了，刷新率跟着变成 68.5 / 77.4 / 77.4 / 59.9 Hz。

MIPI-DSI 屏幕只接受它能锁定的行场时序。给一块要 Panel 4 时序的屏喂 Panel 1 的时序，整条链路上不会有任何错误信息，屏幕就是不亮。背光可能还亮着——所以大家描述成「屏幕亮着但是黑的」。

<div align="center">
<img src="docs/02-measured-panel-timings.png" alt="从四个 R36S 屏幕版本 DTB 文件里解析出来的时序对照表：分辨率和像素时钟相同，只有消隐区间和刷新率不同" width="100%">
</div>

**第二个陷阱：光看标号不够。** 同一份包里，**AmberELEC 那棵树**的「Panel 3」目录又是另一套时序——行前肩/同步/后肩是 119 / 2 / 119，场是 13 / 2 / 5，算出 65.9 Hz。也就是说「Panel 3」不是一套通用时序，它是**某个固件家族下的** Panel 3。

**第三个陷阱：** Panel 2 和 Panel 3 的时序数值完全相同，但两个 `.dtb` 文件并非逐字节相同——树里还有别的地方不一样。**不要因为看得见的数字一样就认为两个文件可以互换。**

<div align="center">
<img src="docs/01-boot-chain-and-panel-timing.png" alt="R36S 启动链路流程图：BOOT 分区存放 dtb，boot.ini 点名，U-Boot 加载，内核从中读取屏幕消隐参数，屏幕要么锁上要么黑屏" width="100%">
</div>

---

## 三、为什么 V12 / V21 / V30 的 dtb 不通用

社区其实是把 R36S 这一家子当成**两个互相独立的坐标轴**来看的，这是刷机前最该先搞清楚的一件事：

- **主板版本（V12 / V21 / V30 …）**——你是**哪台机器**。它决定整棵设备树：内存、WiFi 焊盘、功放、摇杆走线、电池参数。
- **屏幕版本（Panel 1 / 2 / 3 / 4 …）**——你这台机器上**碰巧装的是哪块屏**。同一个主板版本的两台机器，屏幕版本可能不一样，因为工厂会换屏厂。

你需要的 dtb 是这张二维网格上的一个点，而**第二个坐标从机器外面完全看不出来**。这就是「试了很多个 dtb 全都黑屏」这句话这么常见的原因：四个屏幕版本乘上好几个板子家族，瞎猜的命中率很低。

### 关于 V12 / V21 / V30，我们实际验证到的部分

有真凭实据的：

- 社区刷机包里的 `dtb_selector.exe` 依次问你：**机型** → **系统** → **屏幕类型 P1–P4**（只有原版 R36S 才问第三步）。它**不问**「你是 V12 还是 V21 还是 V30」。
- 原版 R36S 跑 ArkOS 时，引导程序点名的文件是 `rk3326-r35s-linux.dtb`，而且**每个屏幕版本各不相同**（四个不同的 MD5、四套不同的时序、两个不同的 `model` 字符串）。
- 随刷机材料一起流传的一份 V12 说明里写得很直白：**「原版机v12的dtb文件名为：`rk3326-gameconsole-r36s-v4.dtb`」**。这是关于 V12 的**一个**已核实的数据点。
- V30 和 V21 的真机主板照片上，版本号直接丝印在板子上：`R36S-V30 / 2025-11-18` 和 `R36S-V21 / 2024-12-18`。两块板子都是瑞芯微 RK3326 加两颗内存颗粒，V21 那块左边缘还标着 `WIFI` 焊盘。也就是说**主板版本是能读出来的**——但你得拆机。

⚠️ **未实测：** 我们**没有**找到任何权威公开来源把「V12 / V21 / V30」一一对应到具体的 dtb 文件名或具体的屏幕版本上。群里流传的说法（比如「v30 要刷最新的 arkos4clone」）和 V30 出厂更晚、需要更新的驱动这个事实是自洽的，但我们没有确认存在这样一张对应表，**你也别把它当成存在**。如果有人给你一张「V30 = X」的表，请当成未确认信息。

⚠️ **未实测：** 我们检查的这份 dtb 包是**社区拼的合集**，我们不知道它的确切来源，它也不等于官方固件发布。

<div align="center">
<img src="docs/06-v30-board-silkscreen.jpg" alt="拆开外壳的 R36S V30 主板照片，丝印为 R36S-V30 / 2025-11-18，下方肩键处标有 RIGHT 和 LEFT" width="72%">
<br><sub>V30 主板。肩键安装位之间丝印为 <code>R36S-V30</code> / <code>2025-11-18</code>。</sub>
</div>

<div align="center">
<img src="docs/07-v21-board-silkscreen.jpg" alt="透过蓝色透明外壳看到的 R36S V21 主板照片，丝印为 R36S-V21 / 2024-12-18，左侧标有 WIFI 焊盘" width="58%">
<br><sub>V21 主板。<code>R36S-V21</code> / <code>2024-12-18</code>，注意左边丝印的 <code>WIFI</code> 焊盘。</sub>
</div>

---

## 四、怎么确认自己是哪块板、哪块屏

**主板版本——拆机看。** 版本号和日期直接印在 PCB 上、肩键旁边，形如 `R36S-V21` / `R36S-V30`。这是我们见过证据支持的**唯一**可靠办法；外壳颜色说明不了板子型号。

**屏幕版本——外面看不出来，丝印上也没有。** 实际可行的路子有三条：

1. **用固件项目自带的识别工具。** (d)ArkOS4Clone 项目在 README 里明写：*「Original DTB files are preserved. You can identify your device type via https://r36s.dpdns.org/dtbTools.html」*。它支持的设备清单里，原版 R36S 被拆成 `R36s Panel 0`、`Panel 1`、`Panel 2`、`Panel 3`、`Panel 4 Type 1`、`Panel 4 Type 2` 六项；另外还有独立的 **Clone R36s**（Type 1/2/3/5，再分带功放/不带功放）和 **SoySauce R36s**（Panel 1–5）家族。群里那位贴出的文件名 `rk3326-r36s-sauce04-linux.dtb` 正好对得上 SoySauce 这个家族。
2. **有依据地试。** 你的屏幕不在列表里的话，就按你的板子家族把候选挨个试，停在第一个画面稳定、不歪不裁的上面。群里那段输出就是这个套路：*「运行 dtb_selector 后依次尝试以下选项，选择兼容性最好的」*。
3. **能进系统就读系统**，见第六节。

---

## 五、怎么修

这里有**三套互不相同的规矩**，把它们搞混正是把机器弄坏的常见原因。先找到你那一行。

| 固件家族 | 由什么决定加载哪个 dtb | 要不要改 dtb 文件名？ |
|---|---|---|
| **(d)ArkOS4Clone**（目前还在维护的那条线） | 由 `console-detect` 机制根据分区上存在的 dtb 文件来识别机型 | **不要改。** 项目原话：*"Device detection handled by console-detect, do not rename DTB files."* |
| **ArkOS-R3XS 及较早的社区包** | `BOOT` 分区上的 `boot.ini` 点名一个 dtb 文件 | `boot.ini` 里的那一行必须和分区上的文件对得上。有些包要求固定文件名，所以才有了「换个文件、名字不变」这种做法。 |
| **Rocknix 类固件** | `BOOT` 分区上的 `boot.ini` 点名一个 dtb 文件 | 改 `${dtb_loadaddr}` 那一行上的文件名。下面是包内文件和流传的 V12 说明里都验证过的写法。 |

### 5a. (d)ArkOS4Clone 路线（建议起点）

官方安装指南给了 Windows 和手动两条路。

**用工具（Windows）：**
1. 下载 ArkOS4Clone 的发布镜像。
2. 刷进 TF 卡。
3. 运行 `dtb_selector.exe`，选择你的设备。
4. 按提示选择电池驱动版本（**Original** 或 **arkos4clone_fix**）。
5. 重启机器。

**手动（Linux、macOS——官方给的非 Windows 流程）：**
1. 挂载 TF 卡的 `BOOT` 分区。
2. 把 `consoles/<你的设备>/` 里的 `boot.ini` **和两个 `.dtb` 文件**复制到该分区根目录。
3. 复制对应的内核 `Image`——`consoles/kernel/original/Image`，或者 `consoles/kernel/arkos4clone_fix/Image`，要和你选的电池驱动对上。
4. 把 `consoles/logo/<分辨率>/logo.bmp` 复制到该分区根目录。
5. 卸载 TF 卡、插回机器、重启。

关于工具的来源：这个工具就是项目自己编译的。它的构建脚本会生成 `dtb_selector_win32.exe`、`dtb_selector_win64.exe`，以及 macOS Intel 和 macOS Apple silicon 版本，输出到镜像的 `dArkOS` 启动目录里。我们检查的社区包里那个中文名 `机型dtb选择，放到boot分区运行.exe`，是别人重新打包的同一类工具。这一点值得知道，因为它意味着**这个工具和它提供的设备清单来自一个公开源码的项目**——你可以拿它去对照维护中的设备清单，而不是盲信某个网盘包。

项目里还有两条直接解释「为什么还是不开机」的说明，原文照抄：

- *"Please connect the charger before powering on. For original devices, make sure the battery voltage is above 3.3V before booting."*
- 原厂系统或其它发行版的 dtb 和 ArkOS4Clone 的工具链**不兼容**。不要把两棵树混着用。

### 5b. 手动改 `boot.ini` 路线（Rocknix 与较早的 ArkOS 包）

dtb 就是第一个小的 **FAT32** 分区上的一个普通文件，任何电脑都能挂载。**不需要重刷任何东西。**

1. 用 Rufus、Win32DiskImager 或 Etcher 把镜像刷进 TF 卡。（我们检查的那份刷机包里就带了 `rufus-3.10.exe`。）
2. **如果 Windows 不显示 `BOOT` 分区**，手动给它一个盘符：按 <kbd>Win</kbd>+<kbd>X</kbd> → **磁盘管理**，按容量认准 TF 卡，右键那个标着 `BOOT` 的小 FAT32 分区 → **更改驱动器号和路径** → **添加** → 随便选一个空闲盘符。
3. 打开 `BOOT` 分区。你会看到类似 `KERNEL`、`SYSTEM`、`SYSTEM.md5`、`KERNEL.md5`、`boot.ini`、`uInitrd`、`logo.bmp`，以及一个或多个 `.dtb` 文件。
4. 用纯文本编辑器打开 `boot.ini`。需要改的只有加载 dtb 的那一行。下面是包里真·原文，原版 R36S 跑 ArkOS 用的：

   ```ini
   setenv dtb_loadaddr "0x01f00000"
   load mmc 1:1 ${dtb_loadaddr} rk3326-r35s-linux.dtb
   booti ${loadaddr} ${initrd_loadaddr} ${dtb_loadaddr}
   ```

   同一份包里，**克隆机**板型用的是加载 `KERNEL` 而不是 `Image`/`uInitrd` 的写法：

   ```ini
   setenv loadaddr "0x02000000"
   setenv dtb_loadaddr "0x01f00000"
   setenv dtbo_loadaddr "0x01e00000"

   load mmc 1:1 ${loadaddr} KERNEL

   load mmc 1:1 ${dtb_loadaddr} rk3326-r36s-g80c.dtb

   booti ${loadaddr} - ${dtb_loadaddr}
   ```

5. **只改** `load mmc ... ${dtb_loadaddr} ...` 这一行末尾的**文件名**，让它等于你已经复制到同一分区的那个 `.dtb` 的名字。路径、地址、`bootargs`、`booti` 那行都别动。
6. 存成不带 BOM 的 ASCII/UTF-8（CRLF 或 LF 都行），弹出卡，开机。

随刷机材料流传的那份 V12 说明讲的是同一件事，并给出了一个关于原版 V12 的、已核实的具体例子：

> ```ini
>  ${dtb_loadaddr} rk3326-rg351mp-linux.dtb
>
> booti ${loadaddr} - ${dtb_loadaddr}
> ```
> → 把 `${dtb_loadaddr} rk3326-rg351mp-linux.dtb` 改成 `${dtb_loadaddr} rk3326-gameconsole-r36s-v4.dtb`

注意**原厂**那一行点名的是什么：`rk3326-rg351mp-linux.dtb`——一块**安伯尼克 RG351MP** 的设备树。这就是整个麻烦的源头：这些镜像默认继承了 RG351MP 那棵树，然后指望每个 R36S 用户自己去把那一行改到自己屏幕上。

### 5c. 如果某个包要求固定文件名

有些包不让你挑名字，引导程序只找一个写死的文件。下面这张截图就是这样一份包的**真实** `BOOT` 分区：里面的 dtb 叫 `rf3536k3ka.dtb`，旁边还有一个 `rf3536k3ka.dtb.HH`。图上的批注说，要把原文件**换成同名文件**、复制进同一个分区。

<div align="center">
<img src="docs/05-boot-partition-on-a-pc.jpg" alt="在电脑上打开的掌机 BOOT 分区：extlinux 文件夹、boot.ini、KERNEL、KERNEL.md5、logo 位图、SYSTEM、SYSTEM.md5，以及名为 rf3536k3ka.dtb 的 dtb 文件，图上有中文批注指出该替换的文件" width="72%">
<br><sub>某个刷机包的 <code>BOOT</code> 分区。dtb 就是那个叫 <code>rf3536k3ka.dtb</code> 的文件；包自己的说明是要用同名文件覆盖它。</sub>
</div>

这种情况下的做法是：把正确的 `.dtb` **改名成这个包要求的名字**，复制进去覆盖旧的。**这跟 (d)ArkOS4Clone 的要求正好相反**——那边不许改名。动手之前先确认自己在哪种固件上。

### 5d. 如果它用 `extlinux` 而不是 `boot.ini`

有些固件走 `extlinux/extlinux.conf`。文件里 dtb 写在 `FDT` 那一行。下面是同一份包里两个真实例子：

```ini
LABEL ArkOS
  LINUX /Image
  FDT /rf3536k4ka.dtb
  INITRD /uInitrd
```

```ini
LABEL ArkOS
  LINUX /Image
  FDT /rf35mg3ka.dtb
  INITRD /uInitrd
```

道理一样：一行、一个文件名，而且这个文件名必须真的存在于 `BOOT` 分区上。

### 5e. 在下结论说「是 dtb 的问题」之前

有两件事看起来和 dtb 不对一模一样，但不是：

- **刷完第一次开机本来就慢。** 卡要重新分区、根文件系统要扩容。这套流程的社区教程说要给它**大约三分钟**再判断。
- **电池电量太低或没电。** 见 5a 里那条充电器说明——有些板子电量不足时压根不开机，dtb 再对也没用。

---

## 六、怎么验证修好了

按可信度从高到低：

1. **画面锁上并且一直稳定。** dtb 对了，画面不滚、不撕裂、不横向偏移、不会被裁掉一条黑边。dtb 完全不对通常是全黑；但**错一半**的 dtb 可能给你一个畸变的画面——那仍然算错。
2. **重启之后还在。** 如果只亮一次、第二次不亮，先怀疑文件写坏了或者卡要挂了，不是 dtb 选错了。
3. **让系统自己报出身份。** 内核会把设备树暴露出来。标准 Linux 上：

   ```sh
   cat /proc/device-tree/model
   # 很多内核上等价的路径：
   cat /sys/firmware/devicetree/base/model
   ```

   对我们解析过的那几个屏幕版本文件，这个字符串是：Panel 1–3 为 `Game Console R35S/R36S fix by AeolusUX`，Panel 4 为 `Game Console R35S/R36S fix by lxc`。文件内容就是属性的原始值，结尾可能带一个终端不显示的 NUL 字节——看着奇怪就用 `xxd` 对一下。

   ⚠️ **未实测：** 这两个 `model` 字符串是我们**直接解析 `.dtb` 二进制**确认的。我们**没有**在真机 R36S 上跑过这两条命令，所以 `cat` 的确切输出（包括结尾字节）属于「预期如此」，不是「实测如此」。
4. **系统自己的重写工具认同你。** 包里那份 `boot.ini` 自带这段注释：

   > *"Changes made to this are overwritten every time there's a new upgrade / To make your changes permanent change it on `boot.ini.default` / After changing it on `boot.ini.default` run the `bootini` command to rewrite this file with your personal permanent settings."*

   也就是说那套固件上有一个 `bootini` 命令，会用 `boot.ini.default` 重新生成 `boot.ini`。如果你改了 `boot.ini`、一升级又变回去了，原因就在这。⚠️ 我们**没有**在真机上跑过 `bootini`，这里只是**照抄文件自己的注释**。

---

## 七、怎么别再踩

- **把能用的那一对备份好。** 屏幕一亮，立刻把用上的那个 `.dtb` **和** `boot.ini`（或 `extlinux.conf`）拷到电脑上，放进一个按机器命名的文件夹，比如 `r36s-v21-panel3/`。十秒钟的事，能省一个晚上。
- **永远不要格式化或「修复」`BOOT` 分区。** 它是 FAT32，Windows 会很热心地问你要不要修。修完引导程序就什么都找不到了。
- **默认镜像里那个 dtb 名字多半是错的。** `boot.ini` 里写着 `rk3326-rg351mp-linux.dtb` 是**正常初始状态**，不是 bug。你要做的就是改掉那一个文件名。
- **别在不同固件项目之间混用 dtb。** ArkOS4Clone 的工具链明说原厂和第三方的 dtb 与它不兼容。
- **别以为两台机器一样。** 主板版本和屏幕版本是两个独立坐标轴。再买一台 R36S，不代表可以跳过第四节。
- **搞清楚哪条线还在维护。** ArkOS 本身已经归档，还在持续更新模拟器的是 ArkOS4Clone 这个分支社区版——这就是为什么新版本机器的建议是「刷最新的 arkos4clone」。
- **镜像不是游戏包。** 「这个 ARKOS 刷机文件 3.6G 的里面带游戏吗」——不带。固件镜像 = 系统 + 模拟器。游戏在 `ROMS` 分区里、按平台分文件夹，要你自己往里拷。
- **黑屏不等于砖。** dtb 是 FAT32 分区上的普通文件，所以最常见的失败可以用任意一台电脑放回正确文件修好。⚠️ **未实测：** 我们**没有**穷举所有失败模式。第一次开机扩容时断电、刷错了**内核**、或者卡本身在坏，都是另一类问题；dtb 选错不能当作「任何黑屏都无害」的许可证。

---

## 八、附：我们检查的这份包里到底有什么

<div align="center">
<img src="docs/03-dtb-selector-menu-and-tree.png" alt="从刷机包中提取的 dtb_selector.exe 菜单：8 个机型、5 个系统、4 个屏幕类型，以及对应的 dtb 目录树，共 9 个板型目录" width="100%">
</div>

<div align="center">
<img src="docs/04-inspecting-the-pack.png" alt="对着刷机包跑真实命令的终端渲染：strings 计数得到 85 个文件，dtb 目录里的工具与压缩包名，R36s Panel 4 ArkOS 目录内容，boot.ini 里的 dtb_loadaddr 行，以及四个屏幕版本 dtb 文件的 md5sum" width="76%">
</div>

从工具内部解出来的目录树，原文照抄：

```
dtb/
├── R36s/
│   ├── P1/  P2/  P3/  P4/
│   │   ├── arkos/
│   │   │   ├── boot.ini
│   │   │   ├── rk3326-r35s-linux.dtb          ← boot.ini 加载的就是这个
│   │   │   ├── rg351mp-kernel.dtb
│   │   │   ├── rk3326-rg351mp-linux.dtb
│   │   │   └── rk3326-rg351mp-linux.dtb.orig / .tony
│   │   ├── amberelec/rk3326-rg351mp-linux.dtb
│   │   └── rocknix/overlays/mipi-panel.dtbo
├── G80C/    arkos/rf3536k3ka.dtb, extlinux/extlinux.conf
├── G80XF/   arkos/rf3536k3ka.dtb, extlinux/extlinux.conf
├── R36Max/  arkos/rf3536k4ka.dtb, Emuelec/rf3536k4ka.dtb
├── R36Pro/  arkos/rf3536k4ka.dtb, Emuelec/rf3536k3ka.dtb
├── Mymini/  arkos/rf35mg3ka.dtb
├── U8/      arkos/rf3536k4ka.dtb + rk3326-U8-linux.dtb
├── Xf35H/   Emuelec/rf3640g3ka.dtb
└── Xf40H/   Emuelec/rf3640g3ka.dtb
```

这棵树里有两点，比任何泛泛的建议都值钱：

- **原版 R36S 走 `boot.ini`**，**克隆机板型走 `extlinux/extlinux.conf`**。两套机制，两个改法位置。
- Panel 4 的 ArkOS 目录里有一个 `Readme.md`，全文就是：

  > **60hz DTB for Panel 4**
  > *Note: The 60Hz variant is experimental. We'd be happy to hear from any issues you encounter compared to the stock 77Hz mode.*

  也就是说，同一个屏幕版本本身就可能有多于一个可用 dtb——一个 60 Hz 的，一个原厂的——而且 60 Hz 那个被作者自己标成实验性。Panel 4 的 60 Hz 文件要是表现异常，那是**已知状况**，不是玄学。

---

## 九、常见问题

**dtb 选错会把 R36S 刷成砖吗？**
通常不会。dtb 是 FAT32 分区上的文件，任何电脑都能挂载，标准救援办法就是把正确文件放回去。把「砖」当成最后的假设，不是第一个。注意事项见第七节。

**同一主板版本的两台 R36S，用的 dtb 一样吗？**
不一定。主板版本和屏幕版本是两个独立坐标轴。这个项目的支持设备清单里，原版 R36S 就单列了六项（`Panel 0` 到 `Panel 4 Type 2`），另外还有独立的克隆机和 SoySauce 家族。

**屏幕是亮的但全黑（背光发亮），这是 dtb 问题吗？**
和 dtb 问题吻合。背光和 MIPI-DSI 数据通路是分开驱动的，所以一块始终收不到可锁定时序的屏，背光照样能亮。

**可以直接抄别人的 dtb 吗？**
只有在**主板版本和屏幕版本都一致**的时候。同一主板家族里 Panel 1 和 Panel 4 会给出不同的刷新率（68.5 Hz 对 59.9 Hz），而这两个文件声明的分辨率和屏控制器是同一个。

**3.6G 的 ArkOS 刷机镜像里带游戏吗？**
不带。固件镜像 = 系统 + 模拟器。游戏放在 `ROMS` 分区。

---

## 十、相关链接

- 我们的在售主板版本与配色：<https://r36s.space/products/r36s>
- 上手与首次开机说明：<https://r36s.space/guide>
- 常见问题：<https://r36s.space/faq>

### 本文用到的来源

第一手——我们直接检查过的文件：

- 一份社区 R36S 刷机包，内含 `机型dtb选择，放到boot分区运行.exe`（即 `dtb_selector`，Windows PyInstaller 打包）、`rockbox.zip`、`rufus-3.10.exe`、一个装有 `rf3536k3ka.dtb` 的 RAR、一个装有 `arkos4clone-uboot.dtb` 与 `rk3326-r50s-linux.dtb` 的 ZIP、一个裸的 `rk3326-..ux.dtb`，以及一张展示 `BOOT` 分区结构的 JPEG。
- 从选择器工具里解出来的 85 个文件，含 9 个板型目录下的 40 个 `.dtb`、各自的 `boot.ini` 与 `extlinux/extlinux.conf`，以及 `dtb/R36s/P4/arkos/Readme.md`。
- 随刷机材料流传的一份 V12 说明（`原版V12刷Rocknix说明.txt`）和一份通用 R36S 刷机教程。
- R36S V30 与 V21 拆机主板照片。

公开来源——用于流程与设备清单：

- [(d)ArkOS4Clone](https://github.com/lcdyk0517/arkos4clone) 及其 [wiki 安装指南](https://github.com/lcdyk0517/arkos4clone/wiki/Installation)、[支持设备清单](https://github.com/lcdyk0517/arkos4clone/wiki)——用于「用工具」和「手动」两套 dtb 安装步骤、「不要改 dtb 文件名」这条规则、充电器与电池电压那条说明、设备清单，以及 DTB 分析工具的链接。
- [ArkOS](https://github.com/christianhaitian/arkos)——上述社区镜像所基于、并从中分叉出去的上游项目。

**发现有错或想补充，欢迎提 issue 或 PR。** 如果你的屏幕版本不在列表里，最有价值的贡献就是：你那个能用文件里的 `.dtb` `model` 字符串，加上它的消隐参数。
