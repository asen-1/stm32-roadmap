# 【草稿】STM32 输入捕获：让 PWM 测自己，一下午挖出四个坑

> 状态：骨架 · 作者填写
> 目标平台：CSDN · 目标 1200~1500 字
> 素材来源：2026-11-19 ~ 11-20

---

## 一、先说结论

（3 句话，让人只看这段也有收获）

- 想验证「输入捕获」，最快的办法是**用自己的 PWM 输出接到自己的捕获引脚**（自环测试）；
- 捕获测不出数，先查**通道号**（CH1 写成 CH4 是最容易犯的错）；
- 工程 printf 静默，先查 **stdio + fputc + MicroLIB 三件套**。

---

## 二、实验设计：为什么要「自己测自己」

（写清楚思路）

```
PA6（TIM3_CH1，PWM 输出）──杜邦线──▶ PB6（TIM4_CH1，输入捕获）
```

**好处**：信号源是我自己产生的，频率/周期**完全已知**，一测就知道对不对。

- 我让 PA6 输出 **1kHz** 的 PWM（PSC=71, ARR=999）
- 捕获端 PSC 也设 71（1 格 = 1µs）
- 所以理论上应该读到：**period = 1000 µs，freq = 1000 Hz**

（贴一张串口截图）

---

## 三、坑一：通道号写错（CH4 vs CH1）

**现象**：串口一个字都没有，`period_ready` 永远是 0。

**我的代码**：

```c
HAL_TIM_IC_Start_IT(&htim4, TIM_CHANNEL_4);   // 错
```

**问题**：我配的是 **PB6 = TIM4_CH1**，但启动的是 **CH4**。CH4 根本没配置，捕获永远不会触发。

**正确**：

```c
HAL_TIM_IC_Start_IT(&htim4, TIM_CHANNEL_1);
```

**教训**：**「CubeMX 里配的」和「代码里用的」必须是同一个东西。**

（这类错误我已经犯过好几次了：舵机信号线插到 5V、ADC 用了被按键占用的 PA0 —— 都是「选错了编号」）

---

## 四、坑二：printf 三件套在「新建工程」里全没了

**现象**：改对通道后，串口还是没输出。

**排查过程**：

1. 先量了 PA6 电压 → 有变化，PWM 正常
2. 看串口助手 → 端口能打开，但没数据
3. 回头检查 `main.c` →

```c
// 没有 #include <stdio.h>
// 没有 int fputc(...)
// 工程设置里 Use MicroLIB 是关的
```

**根因**：这个工程是我用 **CubeMX「新建工程」** 做的，不是**复制模板**做的。

> 我的模板里有 `stdio.h`、`fputc`、MicroLIB，但新建工程从零开始，这些**都不会带过来**。

**修复三步**：

1. `main.c` 的 `USER CODE BEGIN Includes` 里加 `#include <stdio.h>`
2. `huart1` 定义所在的文件（我这里是 `usart.c`）里加 `fputc`
3. Keil → Options → Target → 勾 **Use MicroLIB**

（还踩了个小坑：`fputc` 放 `main.c` 时报 `huart1` 未定义，因为变量在 `usart.c` 里 —— 要么放对文件，要么加 `extern`）

**教训**：**重建（关教程练习）时，也要从模板复制，而不是 CubeMX 新建。**

---

## 五、坑三：花括号把 while 提前关了（第三次）

**现象**：串口彻底静默，连 `tick` 都没有。

**我的代码**：

```c
while (1)
{
    if (period_ready) { ... }
}                          ← 这里多了一个 }

    static uint32_t last_t = 0;    // 全跑到循环外面了
    if (HAL_GetTick() - last_t >= 500) { printf(...); }
}                          ← main 结束
```

**根因**：加新代码时没检查大括号配对，`while` 被提前关闭，后面的代码只执行一次就 `main` 返回。

**教训**：这是我在**同一个坑上第三次跌倒**（串口回显、呼吸灯、这次）。

**检查方法**：把光标放在 `while (1)` 的 `{` 上，看 Keil 高亮配对到哪个 `}`；或者看缩进——`/* USER CODE END 3 */` 前面的 `}` 必须和 `while` 对齐。

---

## 六、坑四：EC11 选不了 Encoder Mode

**现象**：TIM4 的 `Combined Channels` 是灰的，选不了 Encoder Mode。

**原因**：Channel1 已经被占用（之前配的 Input Capture）。

**解决**：先点 **Reset Configuration** 把通道释放，`Combined Channels` 才能选 Encoder Mode。

---

## 七、最终结果

### ① 输入捕获

（贴串口截图）

```
period = 1000 us, freq = 1000 Hz
```

**改 PWM 的 ARR 到 1999 后 → 变成 2000 µs / 500 Hz**，证明捕获真的在工作，不是巧合。

### ② EC11 编码器

（贴串口截图）

```
cnt = 0   speed = 0
cnt = 4   speed = 4     ← 正转一格 = +4（四倍频）
cnt = 8   speed = 4
cnt = 43  speed = 27    ← 快速转动
cnt = 44  speed = -4    ← 反转是负数
cnt = 24  speed = -20
```

**一格 = 4** 是因为四倍频（A↑ A↓ B↑ B↓ 各计一次）。

---

## 八、可复用的三条原则

1. **自环测试**：输出接输入，用已知信号验证未知链路
2. **配置和代码要对得上**：CubeMX 里选 CH1，代码里就得用 `TIM_CHANNEL_1`
3. **新建工程 ≠ 复制模板**：重建练习也要从模板走，否则 printf 基础设施全丢

---

## 九、附

完整工程：https://github.com/asen-1/stm32-roadmap

---

## 写作自查（发布前删掉）

- [ ] 两张串口截图（捕获 + 编码器）
- [ ] 接线示意图（PA6→PB6）
- [ ] 至少一处「我以为是 XX，结果是 YY」
- [ ] 标题带「STM32 / 输入捕获 / 编码器」
