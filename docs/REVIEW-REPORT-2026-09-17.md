# 独立审核报告 — cardreader-ota-gt32（2026-09-17）

> 审核对象：`https://github.com/BOBO-Lite/cardreader-ota-gt32`（main @ `a094fcd`）
> 行为权威对照：`docs/YMODEM-GT32-D1-R02.md`（含 R02.1）+ `docs/ai-collab/*`
> 本文档为外部独立审核（用户委托）产物；结论只读部分不改变任何冻结行为。

---

## 0. 审核范围与方法

| 项 | 内容 |
| --- | --- |
| 范围 | 全部生产源码（`include/` 13 个头、`src/` 13 个源、`tools/factory_pack_main.c`，约 4.2 k 行）、8 个主机测试、3 个检查脚本、`Makefile`、R02 规范与 ai-collab 文档逐条对照 |
| 方法 | 全文精读；R02（§2.2 选路表、§3.3 协议、§3.4 Meta、§3.5 安装、§3.6 失败出口、§3.8 GT32）逐条比对；主机测试复现；双静态分析器；手工边界/并发/失败注入分析 |
| 工具 | MinGW-w64 GCC 14.2.0、clang（两种分析器均已用"故意缺陷样本"做阳性对照，确认有效）；`make test` 全量 |
| 说明 | 本机 MinGW 无 ASan/UBSan 运行库，动态 sanitizer 不可用，已用 `gcc -fanalyzer` + `clang --analyze` 双静态分析替代（结论见 §5） |
| 基线 | 修改前 `make test` 全绿（8 个主机测试二进制 + 4 个检查目标全部 PASS） |

## 1. 结论摘要

- **总体质量高**：分层/职责与 R02 §6、AGENTS 红线一致（boot.c 唯一阶段机、drv_ymodem 无副作用判帧、gt32_raw 独占 SPI、传输路径不写 Meta、禁 CE 均有静态检查兜底）；边界检查（长度/对齐/越界/半写/重复包/回绕）密集；错误码统一负值；测试对每个关键行为都有对应断言。
- **未发现内存安全、越界、未定义行为类缺陷**（人工精读 + 双静态分析器零告警）。
- **发现 1 项与 R02 的实质偏差（A-1）**：EOT1 的"完整镜像检查"目前是长度桩，向量 + Secondary 独立 CRC 复核被推迟到安装阶段。行为上仍不破坏 App（擦 App 前必验），但失败时序/恢复路径与规范不同。因属冻结行为面且修复会改变现有测试预期，**未擅自修改，待决策**。
- **发现 5 项低风险问题（B-1 ~ B-5），已按"零行为变更"原则修复并通过全量回归**（详见 §3、§5）。
- **其余为文档一致性、测试覆盖补强、实机 E3/E4 待办类建议（C 节）**。

## 2. 问题清单（清单总表）

### A 类 — 与 R02 规范的偏差（需设计决策，未修改）

- [ ] **A-1（中）** EOT1 完整镜像检查为长度桩：`src/boot.c` `Boot_OtaImageCheckStub()` 只查 `bytes_written == declared_size`；R02 §2.3 步骤 5 / §3.3.3 要求的"向量合法 + Secondary 独立 CRC32 = 过程累计 CRC32"实际推迟到安装阶段（`Boot_InstallOnce → OtaImage_VerifySecondary`，仍在擦 App 之前）。详见 §4.1。

### B 类 — 已修复（零行为变更 / 纯健壮性，全部验证通过）

- [x] **B-1（低）** `src/boot.c` 重复实现 CRC32（`Boot_Crc32Update` / `Boot_Crc32Finalize` 与 `OtaImage_Crc32Update` / `OtaImage_Crc32Finalize` 逐字节等价）→ 删除本地副本，复用 `ota_image` 版本；`BOOT_CRC32_INIT` 改用 `OTA_IMAGE_CRC32_INITIAL`。收益：去重 32 行，直接减小 Boot 体积（16 KiB 预算）。
- [x] **B-2（低）** `src/ota_image.c` `OtaImage_CopySecondaryToApp()` 中死变量 `body_len`（赋值后仅 `(void)`）→ 删除。
- [x] **B-3（低）** 检查脚本 UTF-8 编码：`scripts/check_transfer_no_meta_commit.py`（2 处）与 `Makefile` `check_no_chip_erase` 内联检查（1 处）的 `read_text()` 未指定编码，在 GBK 等非 UTF-8 默认编码环境（中文 Windows）直接崩溃（实测复现）；与另外两个脚本不一致 → 统一补 `encoding`。
- [x] **B-4（低）** `Makefile` `CC ?= gcc` 永不生效（GNU make 内建 `CC=cc` 使 `?=` 无从赋值；无 `cc` 的系统上构建直接失败）→ 改为 `ifeq ($(origin CC),default)` 守卫赋值，保留环境变量/命令行覆盖语义。
- [x] **B-5（低）** `tools/factory_pack_main.c` `ReadAll()` 空文件路径：`malloc(0)` 可能返回 NULL 且 `fread(NULL,1,0,…)` 属未定义行为边界 → 显式拒绝空文件（提示"空文件"）并简化 NULL 判断。

### C 类 — 仅报告/建议（低优先级，本次未修改）

- [ ] **C-1（低）** 版本字段校验强度不一致：`ota_meta_codec.c:OtaMeta_ValidateFilename()` 对 `cardreader-v` 之后仅查字符集 `[0-9.]`；`ota_meta.c:OtaMeta_ExtractVersion()` 另查 `v<d>.<d>.<d>` 结构（YMODEM/Boot 路径受此保护）。工厂工具 `factory_pack.c:FactoryPack_ExtractVersion()` 两者都不查结构，可打包出 Boot YMODEM 路径不会接受的畸形版本名（如 `cardreader-v...bin`）。影响极小（版本字段仅记录、不参与选路），建议统一校验或文档记录。
- [ ] **C-2（低）** `FactoryPack_Verify()` 未核对"Meta 版本字段 == 文件名提取版本"（R02 §3.4.2 要求一致；Build 自动满足，Verify 作为反向核对可加查，属可选加固）。
- [ ] **C-3（低）** 文档一致性：R02 §6 文件表将"1 ms 时基"归 `drv_time.c`，实现实际使用 `BoardGt32_GetTickMs()`（无 `drv_time` 模块）；README 未说明。建议补一行说明或后续切片对齐。
- [ ] **C-4（低）** 测试覆盖可补强（现有覆盖已很完整，以下路径无直接用例）：
  - `Boot_OtaConsumeRound` 收到 `DRV_YM_ITEM_INVALID` → NAK 的分支；
  - 非 `DONE_GRACE` 相位的 5 s 催发（`WAIT_HEADER/RECEIVE_DATA/WAIT_EOT1/WAIT_END_BLOCK0` 的 C/NAK 重发）；
  - 两个"单 CAN 轮"累计成取消（`can_count` 路径；现仅测了单轮内双 CAN）；
  - `WAIT_END_BLOCK0` 收到 EOT 再回 ACK+C 的分支。
- [ ] **C-5（提示）** 实机面（README/DESIGN_FREEZE 已声明"未验证/未接线"，非缺陷）：目标 FMC、BKP、GPIO/SPI、USART/DMA、NVIC/VTOR/MSP 交接、UART TC 超时复位、传输失败 15 s 等待与重试计数等均为桩；E3/E4 待办。
- [ ] **C-6（可选）** 仓库无 CI；建议 GitHub Actions 跑 `make test`（Linux runner 原生支持，无需改动现有目标）。
- [ ] **C-7（很低）** `Fmc_ReadMetaWord` 校验 4 字节对齐而 `Fmc_ProgramMetaWord`（Mock）不查，语义不对称；现有调用点均对齐，无现状影响。

## 3. 已实施修复的逐项说明（B 类）

所有修改遵循"零行为变更或纯健壮性"原则，未触碰 R02 行为面（分区、旗标语义、时序、失败出口、BKP 契约、禁 CE）。

### B-1 删除 boot.c 的重复 CRC32 实现

- **问题**：`Boot_Crc32Update/Finalize` 与 `OtaImage_Crc32Update/Finalize` 是逐字节等价的同一算法（poly `0xEDB88320`、初值/终值异或 `0xFFFFFFFF`、`NULL && len>0` 提前返回语义一致），属复制粘贴漂移风险 + Boot 体积浪费（16 KiB 预算敏感）。
- **修法**：删除 `BOOT_CRC32_INIT` 与两个静态函数；三个调用点改用 `OTA_IMAGE_CRC32_INITIAL` / `OtaImage_Crc32Update` / `OtaImage_Crc32Finalize`（boot.c 本已 include `ota_image.h`，无新增依赖）。
- **验证**：全量测试（含过程 CRC 与 Meta 提交的 CRC 校验用例）通过；双静态分析器零告警。

### B-2 删除 ota_image.c 死变量 body_len

- 赋值后从未参与任何计算/比较，仅 `(void)` 抑制告警。删除后逻辑不变；`make test`（含安装搬运用例）通过。

### B-3 检查脚本/内联检查补 UTF-8 编码

- **问题复现**：在 GBK 默认编码环境（中文 Windows）运行 `make test`，`check_no_chip_erase` 与 `check_transfer_no_meta_commit.py` 均因读取 UTF-8 中文注释触发 `UnicodeDecodeError` 直接失败；另两个脚本因已写 `encoding="utf-8"` 而正常。
- **修法**：`scripts/check_transfer_no_meta_commit.py` 第 18、22 行与 `Makefile` 内联检查补 `encoding='utf-8'`（Linux UTF-8 环境下行为完全不变）。
- **验证**：修复前可复现崩溃、修复后四个检查目标全部 PASS（见 §5）。

### B-4 Makefile CC 默认值修复

- **问题**：`CC ?= gcc` 因 GNU make 内建 `CC=cc` 永不生效（实测 `CC=[cc]`）；在无 `cc` 的系统（如本机 MinGW）构建即失败，与"默认 gcc"意图不符。
- **修法**：`ifeq ($(origin CC),default)` → `CC = gcc`；命令行 `make CC=clang` 与环境变量 `CC=clang make` 均保持可覆盖。
- **验证**：本机 `mingw32-make clean test` 不再需要手工传 `CC=gcc`，全量通过。

### B-5 出厂工具空文件处理

- **问题**：`ReadAll()` 对 0 字节文件走 `malloc(0)`（可返回 NULL）→ `fread(NULL, 1, 0, …)`，虽主流 libc 实践安全，但属标准未定义行为边界。
- **修法**：显式拒绝空文件并提示 `空文件: <path>`；`buf == NULL` 判断去掉多余的 `sz != 0` 条件。
- **验证**：CLI 冒烟——`pack`/`verify` 正常（退出码 0），空文件输入退出码 1 且输出"空文件"提示。

## 4. 未修改项的详细分析（A 类）

### 4.1 A-1 EOT1 完整镜像检查为桩（中）

**现状**（`src/boot.c`）：

```c
static int Boot_OtaImageCheckStub(const BootOtaSession_t *sess)
{
    if (sess->bytes_written != sess->declared_size) {
        return 0;
    }
    /* 向量/独立 Secondary CRC 复核：Slice4 主机桩视为通过 */
    (void)sess;
    return 1;
}
```

EOT1 只校验"收满声明长度"，即进入第二次 EOT 握手；向量合法性与"Secondary 独立读取 CRC == 过程累计 CRC"的实际校验发生在安装阶段 `Boot_InstallOnce()` → `OtaImage_VerifySecondary()`（在擦 App 之前，全片可靠）。

**R02 要求**（§2.3 步骤 5、§3.3.3 表）：

> 收完声明字节，长度、向量、整镜像 CRC32 通过……第一次 EOT：完整镜像检查通过后 NAK；
> 完整镜像检查：接收长度=声明长度、向量合法、Secondary 独立 CRC32=过程累计 CRC32。失败不得关成成功会话，不得提交 `IMAGE_READY`。

且 §3.6 明确"镜像校验"失败属于**传输尝试**出口（15 s 等待 → 重试 ×3 → 复位；不置 FAIL、不提交 READY）。

**行为差异对比**（以"Secondary 落盘损坏/写失败"为例）：

| | 现状实现 | R02 规范路径 |
| --- | --- | --- |
| EOT1 | 仅长度检查 → NAK → 继续握手 | 完整检查失败 → CAN+CAN，会话失败 |
| 会话结束 | 提交 `IMAGE_READY`（App 被标记待安装） | 不提交 READY，走传输失败出口 |
| 后续 | 安装阶段复核失败 ×3 → 写 `MATE_OTA_FAIL` → 复位 → 上电 OTA | 15 s 等待 → 重传（用户可重试，Meta 不变） |
| App 安全 | 从不擦 App（等价安全） | 从不擦 App |

**影响**：两路径都不破坏 App；但现状会多一次无谓的 READY 提交 + FAIL 写入 + 自动复位，且"升级失败"的恢复行为比规范更重（用户重新上电后处于 FAIL=OTA 状态而非可立即重试的干净状态）。

**修复建议**（需设计确认后实施，注意这是冻结行为面）：

```c
static int Boot_OtaImageCheck(const BootOtaSession_t *sess)
{
    if (sess->bytes_written != sess->declared_size) {
        return 0;
    }
    return (OtaImage_VerifySecondary(sess->declared_size, sess->image_crc32) != 0) ? 1 : 0;
}
```

**连带影响**（修复时须同步处理）：
- `tests/test_boot_ota.c:Test_Boot_CommitThreeFailStayOta` 的回放负载（全 `0xB0`）向量非法，会在 EOT1 被拦截，需改用合法向量负载（其关注的"commit 失败留 OTA"场景可通过注入 Meta 擦除失败保留）；
- EOT1 与安装阶段会各做一次全镜像复核（两次整片读取，约 2 × 113 KiB @ SPI），这是规范原意（§3.3.3 + §3.5 步骤 3），非冗余。

**本次未修改原因**：涉及冻结的失败出口/时序（A3 红线），且会改变现有测试语义；按项目流程应走"暂停 → 设计变更 → 确认"。

## 5. 验证记录

全部验证均在改动后复跑。

```text
$ mingw32-make clean test          # （等价于 Linux 的 make clean test）
... 8 个构建目标全部编译通过（-Wall -Wextra -Werror 零告警）
PASS: no Chip Erase opcodes/symbols in gt32/boot/fmc/meta/image production sources
PASS: transfer path does not call OtaMeta_Commit (Meta only at SESSION_DONE)
PASS: AppOta sources have no Modbus FC/register implementation symbols
PASS: factory_pack sources have no YMODEM/drv_ymodem implementation
build/test_ota_meta_codec   → RESULT: PASS
build/test_gt32_raw         → All gt32_raw tests passed
build/test_ota_meta_commit  → 全用例 PASS（含 E1~E5、boot-ok/boot-fail）
build/test_boot_ota         → 全用例 PASS
build/test_ota_install      → RESULT: ALL PASS
build/test_boot_select      → RESULT: ALL PASS
build/test_app_ota          → RESULT: all AppOta Slice 8 host tests PASS
build/test_factory_pack     → RESULT: all FactoryPack Slice 9 host tests PASS
```

```text
$ factory_pack pack  -b boot.bin -a cardreader-v0.1.0.bin -o factory-test.bin
OK: wrote 131072 bytes -> factory-test.bin
$ factory_pack verify factory-test.bin
OK: verify PASS            （退出码 0）
$ factory_pack pack -b boot.bin -a empty.bin -o nope.bin
空文件: empty.bin           （退出码 1，修复后的新错误路径）
```

```text
静态分析（改动前后均执行）：
$ gcc -std=c99 -Wall -Wextra -fanalyzer -c <每个源文件>      → 零告警
$ clang --analyze -Xanalyzer -analyzer-output=text <每个源文件> → 零告警
阳性对照（故意缺陷样本）确认两套分析器均能检出 null-deref/leak/越界，排除假阴性。
```

**未执行**：ASan/UBSan 动态插桩（本机 MinGW 工具链无对应运行库）、Keil 目标构建、实板 E3/E4——均超出本次主机审核范围。

## 6. 本轮改动文件清单

| 文件 | 改动 |
| --- | --- |
| `src/boot.c` | 删除重复 CRC32 实现（-32 行），改调 `OtaImage_Crc32*`（B-1） |
| `src/ota_image.c` | 删除死变量 `body_len`（B-2） |
| `scripts/check_transfer_no_meta_commit.py` | 2 处 `read_text()` 补 `encoding="utf-8"`（B-3） |
| `Makefile` | `CC ?= gcc` → `origin` 守卫；内联检查补 `encoding='utf-8'`（B-3/B-4） |
| `tools/factory_pack_main.c` | 空文件显式拒绝 + NULL 判断简化（B-5） |

> 建议：以上改动均为非行为性修复；按项目流程可在下一交付卡中一并登记（不构成对 R02 的任何偏离）。

---

*报告完。A-1 请决策；C 节各项可按需排期。*
