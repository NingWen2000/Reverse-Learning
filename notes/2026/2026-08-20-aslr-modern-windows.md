# 现代 Windows 是否仍在使用 ASLR

- 创建日期：2026-08-20
- 更新时间：2026-08-20
- 来源：自主学习
- 架构：x86 / x64 / ARM64
- 环境：Windows 10、Windows 11、Windows Server
- 状态：分析中（官方文档已核对，待完成本机动态实验）

## 原始问题

ASLR 安全机制现在还在使用吗？

## 分析思路

不能只看 Windows 安全中心里“强制随机化映像”的开关。需要区分三个层次：

1. PE 文件是否通过 `DYNAMIC_BASE` 标志主动支持重定位。
2. Windows 是否随机化映像、栈、堆和其他虚拟内存分配。
3. Exploit Protection 是否强制没有主动加入 ASLR 的旧映像重定位，即 Mandatory ASLR。

只有第三项默认关闭，并不代表整个 ASLR 机制已经停用。

## 当前判断

现代 Windows 仍在使用 ASLR。MSVC 默认启用 `/DYNAMICBASE`；64 位映像还默认启用 `/HIGHENTROPYVA`。Windows 10/11 的 Exploit Protection 默认开启 Bottom-up ASLR 和高熵 ASLR，但系统级 Mandatory ASLR 默认关闭，以兼顾旧程序兼容性。

因此，“ASLR 仍在使用”和“系统强制所有旧程序参与 ASLR”是两个不同判断。

## Windows 与 PE 底层原理

### PE 标志

链接器的 `/DYNAMICBASE` 会在 PE Optional Header 的 `DllCharacteristics` 中设置 `IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE`。它告诉 Windows Loader：这个 EXE 或 DLL 可以不在首选 `ImageBase` 处加载。

64 位程序还可以设置 `IMAGE_DLLCHARACTERISTICS_HIGH_ENTROPY_VA`，让加载器利用更大的虚拟地址空间增加随机性。Microsoft 文档说明 `/HIGHENTROPYVA` 依赖 `/DYNAMICBASE`，且二者在相应的现代 MSVC 目标上默认启用。

### 重定位过程

假设链接时的首选基址是：

```text
PreferredImageBase = 0x140000000
ActualImageBase    = 0x7FF600000000
Delta              = ActualImageBase - PreferredImageBase
```

Windows Loader 映射映像后，根据 `.reloc` 中的 Base Relocation Block 找到需要修正的绝对地址，并加上 `Delta`。常见重定位类型包括：

- x86：`IMAGE_REL_BASED_HIGHLOW`
- x64：`IMAGE_REL_BASED_DIR64`

如果旧程序删除了重定位表，加载器就无法安全修正其中的绝对地址。Mandatory ASLR 的“阻止已去除重定位信息的映像”选项可以选择直接拒绝加载这类文件。

### 与汇编的关系

x64 大量使用 RIP-relative 寻址，例如：

```asm
lea rcx, [rip + displacement]
```

目标地址由当前 RIP 与相对位移计算，同一模块整体移动后，相对距离通常不变，因此不一定需要重定位。

而把完整虚拟地址直接编码进指令或数据时，例如：

```asm
mov rax, 0000000140012340h
```

这个绝对值会受到映像基址变化影响，通常需要 `.reloc` 中的对应修正项。

## 默认设置中的关键区别

根据 Microsoft 当前的 Exploit Protection 文档，Windows 10、Windows 11 与相应 Windows Server 版本的系统默认值包括：

- Mandatory ASLR：默认关闭
- Bottom-up ASLR：默认开启
- High-entropy ASLR：默认开启
- DEP、CFG 等其他缓解措施也作为分层防护继续使用

Mandatory ASLR 关闭主要是兼容性选择：某些旧程序依赖固定基址，或者已经删除 `.reloc`。正常使用 `/DYNAMICBASE` 构建的现代程序仍然可以参与 ASLR。

在 ARM、ARM64 和 ARM64EC 目标上，Microsoft 文档说明不能通过 `/DYNAMICBASE:NO` 禁用 ASLR。

## Ghidra 静态检查

1. 导入 PE 后记录首选 `ImageBase`。
2. 在 PE Optional Header 中检查 `DllCharacteristics` 的 `DYNAMIC_BASE` 与 `HIGH_ENTROPY_VA` 标志。
3. 在 Memory Map 或程序树中确认 `.reloc` 是否存在且非空。
4. 选择一个关键函数，记录其 RVA：

```text
RVA = Ghidra 中的静态地址 - PreferredImageBase
```

注意：Ghidra 显示的地址通常以分析时采用的映像基址为基准，不等于某次运行中的真实 VA。

## x64dbg 动态验证计划

1. 使用 x64dbg 启动一个启用了 `/DYNAMICBASE` 的自编译测试程序。
2. 在 Modules 或 Memory Map 中记录主模块的实际加载基址。
3. 用 `实际基址 + RVA` 找到 Ghidra 中选择的关键函数并设置断点。
4. 彻底结束进程后重新启动多次，再记录加载基址。
5. 对照函数 RVA：基址可能变化，但同一构建中函数 RVA 应保持不变。
6. 再构建一个 `/DYNAMICBASE:NO` 的对照样本，比较实际加载情况和 PE 标志。

一次启动中观察到固定地址不能证明 ASLR 未启用；某些模块可能复用地址，随机结果也可能偶然重复，因此应进行多次新进程实验并检查 PE 标志。

## 安全意义与局限

ASLR 不是内存漏洞的修复，而是提高利用成本。它使攻击者难以预先知道代码、栈、堆或库的地址，但以下情况仍可能削弱或绕过它：

- 信息泄露暴露了模块或堆地址
- 进程中存在未启用 ASLR 的模块
- 32 位地址空间熵较低
- 攻击者能够反复尝试或利用部分地址覆盖
- 其他内存破坏原语足以构造更完整的利用链

因此现代 Windows 将 ASLR 与 DEP、CFG、SEHOP、CET/硬件强制栈保护等机制组合使用。

## 阶段性结论

ASLR 没有被淘汰，仍是现代 Windows 的基础漏洞缓解机制。需要牢记：`Mandatory ASLR 默认关闭` 只表示系统不会默认强迫所有旧映像重定位，不等价于 `ASLR 默认关闭`。

## 官方资料

- [Microsoft：/DYNAMICBASE](https://learn.microsoft.com/en-us/cpp/build/reference/dynamicbase-use-address-space-layout-randomization)
- [Microsoft：/HIGHENTROPYVA](https://learn.microsoft.com/en-us/cpp/build/reference/highentropyva-support-64-bit-aslr)
- [Microsoft：Exploit protection 默认配置](https://learn.microsoft.com/en-us/defender-endpoint/evaluate-exploit-protection)
- [Microsoft：Exploit protection 中的 Mandatory 与 Bottom-up ASLR](https://learn.microsoft.com/en-us/defender-endpoint/exploit-protection-reference)

## 后续练习

- [ ] 分别编译 `/DYNAMICBASE` 与 `/DYNAMICBASE:NO` 的 x64 测试程序
- [ ] 在 Ghidra 中确认两个样本的 PE 标志和 `.reloc`
- [ ] 在 x64dbg 中记录多次启动的模块基址
- [ ] 任选一个函数，用 RVA 在变化后的基址上重新定位

## 发布检查

- [x] 不含账号、Token、Cookie 或私人路径
- [x] 不含比赛 Flag 或受限题目附件
- [x] 官方资料链接已记录
- [ ] x64dbg 动态实验尚未完成
