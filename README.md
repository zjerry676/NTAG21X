# NTAG21X
# NTAG21x 与 MIFARE Classic 1K/EV1 对比笔记

这是一份面向嵌入式软硬件开发的工程速查笔记。对比基准固定为：

- NTAG213/215/216：NXP `NTAG213_215_216` 数据手册 Rev. 3.2。
- MIFARE Classic 1K/EV1：NXP `MF1S50YYX/V1` 数据手册。
- MIFARE Classic 的 NDEF 映射：NXP AN1305 的 MAD/TLV 方案。

老款 MIFARE Classic、国产兼容卡和克隆卡可能在 UID、射频参数、随机 ID、写入寿命和安全实现上不同。涉及量产卡时，应以实际卡片和读卡器的测试结果为准。

## 结论先行

- 手机直接读写 URL、短文本、配对信息和普通标签类 NDEF：优先 NTAG21x。
- 既有门禁、票务或读卡器基础设施要求 MIFARE Classic：选择 M1，但必须实现扇区认证、访问位和 value block 规则。
- 不能只替换标签芯片而复用另一套驱动。两者的地址单位、认证流程、写入粒度和命令集不同。
- 对新建的高安全场景，不要把 MIFARE Classic 或 NTAG21x 的基础保护当作完整的应用层安全方案。

## 对比总表

| 维度 | NTAG213/215/216 | MIFARE Classic 1K/EV1 | 工程影响 |
| --- | --- | --- | --- |
| 标准定位 | 原生 NFC Forum Type 2 Tag，按 4 字节页组织 EEPROM | MIFARE 专有存储器和命令模型；NDEF 通过 MAD/TLV 映射 | NFC 手机兼容性和驱动模型不同 |
| 空口 | ISO/IEC 14443-A，13.56 MHz，106 kbit/s | ISO/IEC 14443-A，13.56 MHz，106 kbit/s | 激活流程有共同基础，但应用命令不兼容 |
| 典型工作距离 | 约 100 mm，取决于天线、匹配和读写器 | 约 100 mm，取决于天线、匹配和读写器 | 不能把距离当作芯片的固定保证值 |
| 抗碰撞、校验 | 支持 ISO/IEC 14443-A 抗碰撞、CRC 和奇偶校验 | 支持 ISO/IEC 14443-A 抗碰撞、CRC 和奇偶校验 | 可复用射频激活层，不能复用存储器层 |
| 射频输入电容 | 典型 50 pF | 约 14.9 pF（最小）、16.9 pF（典型）、19.0 pF（最大） | 天线和匹配网络不能直接视为完全兼容替代品 |
| 存储组织 | 4 字节页；用户区 144/504/888 字节 | 1 KB EEPROM；16 个扇区，每扇区 4 个 16 字节块 | 地址和读写粒度完全不同 |
| 应用数据空间 | 数据手册给出的用户区为 144/504/888 字节 | 约 752 字节，为扣除制造商块和 16 个 sector trailer 后的布局推导值 | MIFARE 的可用空间不是简单的 1024 字节减一段保留区 |
| 安全模型 | 32 位 PWD、16 位 PACK、可选读写保护、静态/动态锁定位、ECC 原创签名；无类似 MIFARE Classic 的加密数据通道 | 每扇区 Key A 6 字节、Key B 6 字节、访问位、三次认证和 CRYPTO1 加密交换 | 两套认证状态机和密钥管理不能互换 |
| UID | 固定为厂家写入的 7 字节 UID | EV1 变体可为 7 字节 UID 或 4 字节 NUID | 应用层不要把 UID 长度写死为 4 字节 |
| NDEF | 原生 CC 字节描述 NDEF 区域；可做 UID 和 NFC 计数器 ASCII 镜像 | 需要把 NDEF 放入 NFC sector，由 MAD、TLV 和 sector access bits 共同管理 | 手机端体验和卡片初始化流程不同 |
| 写入寿命 | EEPROM 写入耐久度典型目标为 100,000 次量级 | 数据手册给出 100,000 次量级；兼容卡可能不同 | 高频计数应评估磨损和备用策略 |

## NTAG21x 工程速查

### 容量和内存地图

| 型号 | EEPROM 总量 | 页数 | 用户区 | 用户页 | 动态锁页 | 配置页 |
| --- | ---: | ---: | ---: | --- | --- | --- |
| NTAG213 | 180 字节 | 45 页 | 144 字节 | `04h-27h` | `28h` | `29h-2Ch` |
| NTAG215 | 540 字节 | 135 页 | 504 字节 | `04h-81h` | `82h` | `83h-86h` |
| NTAG216 | 924 字节 | 231 页 | 888 字节 | `04h-E1h` | `E2h` | `E3h-E6h` |

每页为 4 字节。通用地图如下：

| 页 | 作用 |
| --- | --- |
| `00h` | 固定值，保留 |
| `01h` | UID 和内部字节的一部分 |
| `02h` | UID 余下部分、内部字节和静态锁定位 |
| `03h` | Capability Container（CC） |
| `04h` 起 | NDEF 和用户数据 |
| 用户区末页之后 | 动态锁定位、配置页和制造商保留区，具体页号随型号变化 |

CC 的默认值和 NDEF 区域声明：

| 型号 | Page `03h` 默认值 | CC 中的 NDEF 大小字段 | 备注 |
| --- | --- | --- | --- |
| NTAG213 | `E1 10 12 00` | `12h`，144 字节 | Page `04h` 默认 `01 03 A0 0C`，Page `05h` 默认 `34 03 00 FE` |
| NTAG215 | `E1 10 3E 00` | `3Eh`，496 字节 | Page `04h` 默认 `03 00 FE 00`，Page `05h` 全零 |
| NTAG216 | `E1 10 6D 00` | `6Dh`，872 字节 | Page `04h` 默认 `03 00 FE 00`，Page `05h` 全零 |

CC 的 NDEF 大小字段是格式化区域的声明值，不应与数据手册列出的整段用户 EEPROM 字节数混为一谈。写入 NDEF 时还要正确生成 TLV 的 `03h`（NDEF Message）和 `FEh`（终止符）。

### 激活和状态机

NTAG21x 使用 ISO/IEC 14443-A 的基础激活流程：

1. `REQA (26h)` 或 `WUPA (52h)`：寻卡并唤醒。
2. `ANTICOLLISION CL1` / `SELECT CL1`：处理 UID 第一级。
3. 对 7 字节 UID 继续 `ANTICOLLISION CL2` / `SELECT CL2`。
4. 收到 `SAK` 后进入 ACTIVE 状态，可发送 NTAG 应用命令。
5. `HLTA (50h 00h + CRC_A)`：进入 HALT；再次使用 `WUPA` 唤醒。

NTAG 没有 MIFARE Classic 的扇区认证状态。密码保护在 `PWD_AUTH` 成功后改变受保护页的读写权限，不能当作一条 CRYPTO1 式加密数据通道。

### 常用命令

| 命令 | 码 | 数据阶段 | 说明 |
| --- | --- | --- | --- |
| `READ` | `30h` | 1 字节页号 | 从指定页开始返回连续 4 页，共 16 字节；跨末页时按数据手册规则处理 |
| `WRITE` | `A2h` | 页号 + 4 字节 | 一次写一页，随后等待 ACK/NAK |
| `FAST_READ` | `3Ah` | 起始页 + 结束页 | 连续读取页范围，不应越过有效地址 |
| `GET_VERSION` | `60h` | 无 | 返回 8 字节产品版本信息；`60h` 在 MIFARE Classic 中含义不同 |
| `READ_CNT` | `39h` | 计数器编号 | 读取 NFC counter |
| `PWD_AUTH` | `1Bh` | 4 字节 PWD | 返回 2 字节 PACK；失败时按配置限制重试 |
| `READ_SIG` | `3Ch` | 无 | 读取 ECC 原创性签名 |

常见 ACK 为 4 位 `Ah`；NAK `0h`、`1h`、`4h`、`5h` 表示不同错误类别。实际驱动必须按 bit frame、CRC_A 和超时规则处理，而不是把返回值当作普通 8 位字节。

### 安全和配置

- `PWD` 为 32 位，`PACK` 为 16 位。
- `AUTH0` 指定从哪一页开始启用密码保护，`PROT` 决定保护读操作还是只保护写操作。
- `AUTHLIM` 可限制连续密码失败次数；超过限制后标签可能永久拒绝密码认证，必须谨慎配置。
- Page `02h` 的静态锁定位和动态锁页控制用户区是否可再写；锁定位通常是不可逆的。
- 配置页还包含 `CFGLCK`、`NFC_CNT_EN`、`NFC_CNT_PWD_PROT`、`MIRROR`、`MIRROR_PAGE` 和 `MIRROR_BYTE` 等字段。锁配置前应保留原始页内容并在样卡上验证。
- NTAG 的 ECC 签名适合做原创性校验；它不是应用层密钥协商或端到端保密机制。

### UID 和镜像

NTAG21x 的 UID 为固定 7 字节。通过配置页可以把 UID 的 ASCII 表示或 NFC counter 的 ASCII 表示镜像到用户数据区。UID 镜像占 14 字节，counter 镜像占 6 字节，联合镜像还包含分隔字符，总长度为 21 字节。镜像位置由 `MIRROR_PAGE` 和 `MIRROR_BYTE` 指定，应用 TLV 长度必须同步调整。

## MIFARE Classic 1K/EV1 工程速查

### 扇区和块

MIFARE Classic 1K 有 1024 字节 EEPROM，分为 16 个扇区，每扇区 4 个 16 字节块，共 64 个块：

| 块类型 | 数量 | 内容 |
| --- | ---: | --- |
| Sector 0 Block 0 | 1 | 制造商块，包含 UID、校验字节和厂家数据，正常应用不能覆盖 |
| 普通数据块 | 47 | 用户数据或 value block，受本扇区访问位控制 |
| Sector trailer | 16 | Key A 6 字节、Access Bits 3 字节、GPB 1 字节、Key B 6 字节 |

因此，按标准 1K 布局推导，普通应用数据空间约为 `2 x 16 + 15 x (3 x 16) = 752` 字节。这个数字是布局推导值，不是把所有 1024 字节都当作可连续寻址的保证；value block、MAD 和 TLV 还会进一步占用空间。

### 认证和访问位

- 每个扇区分别拥有 Key A 和 Key B，各 6 字节。
- 读写数据块前，读写器先用 `60h`（Key A）或 `61h`（Key B）对目标扇区执行三次认证。
- 认证成功后，后续命令在该扇区的 CRYPTO1 加密会话中执行；切换扇区通常需要重新认证。
- Sector trailer 的 Access Bits 和 GPB 决定 Key 是否可读、数据块是否只读、是否为 value block，以及 Key A/Key B 哪些操作被允许。
- 出厂传输配置常见默认 Key 为 `FF FF FF FF FF FF`，但量产卡和系统初始化后不应依赖默认值。

CRYPTO1 是传统兼容安全机制。对于新建的安全相关应用，应评估 MIFARE Plus 或 MIFARE DESFire，并在应用层设计密钥轮换、重放防护和后端校验。

### 常用命令

| 命令 | 码 | 数据阶段 | 说明 |
| --- | --- | --- | --- |
| `AUTHENTICATE A` | `60h` | 块号 + nonce/密钥协商 | 对目标扇区使用 Key A；认证后建立 CRYPTO1 会话 |
| `AUTHENTICATE B` | `61h` | 块号 + nonce/密钥协商 | 对目标扇区使用 Key B |
| `READ` | `30h` | 1 字节块号 | 读取一个 16 字节块；不是 NTAG 的 4 页窗口 |
| `WRITE` | `A0h` | 块号 + 16 字节 | 两阶段传输：命令后发送 16 字节数据并等待 ACK |
| `INCREMENT` | `C1h` | 块号 + 4 字节值 | 对 value block 做加法 |
| `DECREMENT` | `C0h` | 块号 + 4 字节值 | 对 value block 做减法 |
| `RESTORE` | `C2h` | 块号 | 把 value block 值载入内部传输寄存器 |
| `TRANSFER` | `B0h` | 块号 | 把内部传输值写入目标 value block |

MIFARE Classic 也使用 `REQA`、`WUPA`、级联防碰撞、`SELECT` 和 `HLTA` 等 ISO/IEC 14443-A 激活命令，但 `60h`、`A0h` 以及认证后的加密状态都属于 MIFARE Classic 命令模型。

### NDEF 映射

MIFARE Classic 不是原生 NFC Forum Type 2 Tag。按 AN1305，NDEF 通常放在专用 NFC sector 中，并由以下信息共同管理：

- MAD（MIFARE Application Directory）标识 NFC 应用占用的扇区。
- 扇区访问位定义读写权限以及 Key A/Key B 的使用方式。
- NFC sector 内使用 TLV，典型类型包括 `03h`（NDEF Message）和 `FEh`（终止符）。

因此，MIFARE Classic 的“能否被手机识别为 NDEF”取决于卡片初始化、MAD、TLV 和访问位是否整体正确，不能只写入一段 NDEF 字节。

## 命令兼容性陷阱

两种标签共享 ISO/IEC 14443-A 激活层，但应用层命令不能按命令码直接替换：

| 操作 | NTAG21x | MIFARE Classic 1K/EV1 | 常见后果 |
| --- | --- | --- | --- |
| `READ (30h)` | 从页号开始读 4 页，返回 16 字节 | 读一个块，返回 16 字节 | 返回长度相同，但地址单位不同 |
| 写入 | `WRITE (A2h)`，4 字节页 | `WRITE (A0h)`，16 字节块，两阶段传输 | 缓冲区、ACK 时序和掉电恢复策略不同 |
| `60h` | `GET_VERSION` | Key A 认证 | 把一套驱动的探测命令发给另一种卡会进入错误状态 |
| 认证 | `PWD_AUTH (1Bh)`，PWD/PACK | `60h`/`61h`，每扇区 Key A/Key B，三次认证 | 认证状态、密钥长度和保护范围不同 |
| 连续读取 | `FAST_READ (3Ah)` | 无等价 NTAG 命令 | 需要按块循环读取并跨扇区重新认证 |
| 计数器/签名 | `READ_CNT (39h)`、`READ_SIG (3Ch)` | 无直接等价命令 | 不能把 NTAG 的镜像和原创签名能力投射到 MIFARE |
| value block | 无 MIFARE value block 命令 | `INCREMENT`/`DECREMENT`/`RESTORE`/`TRANSFER` | 余额类数据需使用不同的数据结构和事务流程 |

驱动分层时，可以复用 RF 唤醒、抗碰撞、CRC_A 和超时基础设施；寻卡后的卡型识别、地址抽象、认证、读写和错误处理应分别实现。

## 选型建议

| 场景 | 建议 | 原因 |
| --- | --- | --- |
| 手机读写 URL、短文本、配对信息、普通标签 NDEF | NTAG213/215/216 | 原生 Type 2、CC/TLV 直接、页模型简单，手机兼容性好 |
| 已有门禁、票务或专用读卡器，系统依赖 MIFARE Classic | MIFARE Classic 1K/EV1 | 兼容既有扇区、密钥和 value block 工作流 |
| 需要比传统卡更强的新安全设计 | 评估 MIFARE Plus 或 MIFARE DESFire | NTAG 密码保护和 Classic CRYPTO1 都不是完整应用层安全方案 |
| 只想换芯片而保留另一套驱动 | 不建议 | 射频输入电容、内存寻址、认证、写入粒度和命令码均存在实质差异 |

## 来源与范围

本文只覆盖协议、内存组织、基础安全和工程选型，不包含 MCU 驱动、NDEF 解析器或外部依赖。事实来源：

1. [NXP NTAG213/215/216 data sheet Rev. 3.2](https://www.nxp.com/docs/en/data-sheet/NTAG213_215_216.pdf?pspll=1)
2. [NXP MIFARE Classic EV1 1K data sheet](https://www.nxp.com/docs/en/data-sheet/MF1S50YYX_V1.pdf)
3. [NXP AN1305: MIFARE Classic as NFC Type MIFARE Tag](https://www.nxp.com/docs/en/application-note/AN1305.pdf)
4. [NXP MIFARE Classic product family](https://www.nxp.com/products/rfid-nfc/mifare-hf/mifare-classic:MC_41863)

使用非 NXP 原厂 M1 卡时，必须另外验证 UID 类型、射频输入参数、随机 ID、写入寿命、默认密钥行为和安全实现。
