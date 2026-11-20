# 模板使用说明（72M + 串口 + printf + LED）

> 从这份模板复制出新工程时，**先花 30 秒过一遍下面的清单**，能省掉 90% 的低级问题。

## 一、这份模板包含什么

| 项 | 内容 |
|---|---|
| 时钟 | HSE 8MHz → PLL ×9 → **72MHz**；APB1=36MHz，APB2=72MHz |
| 调试口 | SYS → Debug = **Serial Wire**（PA13/PA14 保留 SWD） |
| 串口 | **USART1**：115200，8N1，PA9/PA10，中断已在 NVIC 使能 |
| printf | `#include <stdio.h>` + `int fputc(...)` + 工程勾选 **Use MicroLIB** |
| LED | **PC13**：推挽输出，初始电平 High（低电平点亮） |
| 主循环 | 空的（干净的起点） |

## 二、开工前检查清单 ⚠️

复制完模板、打开 Keil 后，逐条确认：

- [ ] **工程名/路径**：不含中文和空格
- [ ] **魔术棒 → Target**：☑ **Use MicroLIB**（printf 必需）
- [ ] **魔术棒 → Debug → Settings → Flash Download**：☑ **Reset and Run**（下完自动跑，否则板子跑的还是旧固件）
- [ ] **魔术棒 → C/C++**：建议勾 **C99 Mode**（可以随用随声明变量）
- [ ] **CubeMX 里再生成代码后**：确认 `USER CODE` 区的内容还在

## 三、printf 三件套（最容易丢）

新工程如果 printf 没输出，**按这个顺序查**：

1. `main.c` 里有 `#include <stdio.h>` 吗？
2. `huart1` 所在的文件（`main.c` 或 `usart.c`）里有 `int fputc(...)` 吗？
   - 如果 `huart1` 在 `usart.c` 定义，**fputc 就要写在 `usart.c`**，否则要 `extern UART_HandleTypeDef huart1;`
   - 用了 `FILE` 类型的文件，**必须** `#include <stdio.h>`
3. 工程设置里 **Use MicroLIB** 勾了吗？

> 三条缺一，printf 就是哑巴，而且**编译不报错**。

## 四、常见坑速查

| 现象 | 原因 |
|---|---|
| 串口一个字都没有 | ① MicroLIB 没勾 ② 没加 fputc ③ 板子在跑旧固件（没复位） |
| 烧录完行为没变 | Keil 没勾 Reset and Run，或没按复位键 |
| 编译报 `FILE undefined` | 当前文件没 `#include <stdio.h>` |
| 烧录报 `Cannot access target` | 板子被别的东西占用 / 线松 / 需要 Connect under Reset |
| 程序一点反应都没有 | 检查是不是卡在 `Error_Handler()`（时钟没配起来） |

## 五、复制后要改什么

1. CubeMX 打开 `.ioc` → 加外设 → **GENERATE CODE**
2. 代码只写在 `USER CODE BEGIN/END` 之间
3. 新建的工程目录改名（如 `12_xxx`），**别覆盖模板本身**

## 六、重建（关教程）时的建议

**从模板复制**，不要用 CubeMX「新建工程」——新建会丢掉上面的所有基础设施（printf、LED、时钟配置），每次都要重补一遍。

> 这条是 2026-11-20 用血换来的：两个重建工程都因为这个原因 printf 失效。