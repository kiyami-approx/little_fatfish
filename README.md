# 小肥鱼 · LLM-LW

> 一个能跑在 **STM32F407** 上的中文对话模型
> 266 KB 权重 · 1 ms 推理 · 完全离线 · 纯 numpy 手写训练

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![MCU](https://img.shields.io/badge/MCU-STM32F407ZGT6-blue)](https://www.st.com/en/microcontrollers-microprocessors/stm32f407zg.html)
[![Params](https://img.shields.io/badge/Params-68K-green)]()
[![Accuracy](https://img.shields.io/badge/Accuracy-80.4%25%20%C2%B1%201.3-brightgreen)]()

---

## 这是什么

把一句话通过串口发给单片机，单片机判断你想干什么，然后在 OLED 上回你一句。

```
你     > 你好
小肥鱼 > 你好，我是小肥鱼

你     > 我心情不好
小肥鱼 > 别难过，我陪你

你     > 十二减四
小肥鱼 > 12-4=8

你     > 苹果好吃吗
小肥鱼 > 不太懂，换个说法
```

### 这不是什么

**这不是生成式大模型。** 它不会"写"文字，也没有 Transformer。

它本质是一个 **37 类的文本分类器**：
- 输入：一句话
- 输出：37 个意图中的一个（打招呼 / 表达难过 / 问时间 / 算术题 / 不懂……）
- 然后从表里挑一句对应的回复

**但这也正是它能塞进单片机的理由**。整套系统（含 3500 字中文字库）只占 **406 KB Flash / 7.9 KB RAM**。

---

## 亮点

**⚡ 真的能跑起来，不是玩具 demo**
完整实现了"串口输入 → 特征提取 → 神经网络推理 → OLED 中文显示 → 串口回传"的闭环，在真实硬件上验证过。

**📦 没有任何深度学习框架依赖**
训练代码只用 **numpy**。前向传播、反向传播、softmax、交叉熵、动量优化、早停，全部手写。适合想搞懂"神经网络到底怎么跑"的人。

**🔬 全链路可复现**
从 `dataset.py` 里的原始语句，到 `model_weights.h` 里的一个 float，每一步都有代码、有数据、有对拍验证。

**📝 诚实的实验记录**
哪些方案有效、哪些完全没用（随机森林？EBM？dropout？），都记在[详细文档](小肥鱼项目总结.md)里。**包括失败的尝试**。

**🀄 支持中文，还是混合写法**
`3+5`、`十二减四`、帮我算100-37 都能算。中文显示走 3500 字点阵字库。

---

## 效果

| 指标 | 数值 |
|---|---|
| 意图类别 | 37 |
| 训练样本 | 1201 条原始 → 4266 条增强后 |
| 模型参数 | 68,005 个（266 KB） |
| 准确率（5 折交叉验证） | **80.4% ± 1.3** |
| 准确率（单次划分） | 81.1% |
| 杂句兜底率 | 100% |
| Flash 占用 | 406 KB / 1 MB |
| RAM 占用 | 7.9 KB / 192 KB |
| 单次推理 | 约 1 ms |

**支持的意图**（37 类）：

| 分类 | 意图 |
|---|---|
| 社交 | 打招呼、告别、感谢、道歉、夸我、求夸 |
| 身份 | 名字、身份、主人、年龄、位置 |
| 情绪 | 开心、难过、生气、累、无聊、孤独、想念、兴奋 |
| 生活 | 天气、时间、吃饭、饿、睡觉、学习、玩耍 |
| 娱乐 | 讲笑话、唱歌、讲故事 |
| 能力 | 会什么、求助、再说一遍、为什么、确认 |
| 运算 | 四则运算 |
| 兜底 | 不懂 |

---

## 快速开始

### 硬件要求

| 部件 | 型号 |
|---|---|
| 开发板 | STM32F407ZGT6（LQFP144，1 MB Flash / 192 KB RAM） |
| 显示屏 | SSD1306 兼容 OLED，0.96" 128×64 |
| 调试器 | ST-Link V2 |
| 晶振 | **外部 8 MHz** |

### 接线

**OLED**（软件 I2C，GPIO 模拟供电）：

| OLED | STM32 |
|---|---|
| GND | PB6 |
| VCC | PB7 |
| SCL | PB10 |
| SDA | PB11 |

> ⚠️ **OLED 的电源由 PB7 提供**。如果你的模块有独立电源脚，建议接 3.3V，并修改 `OLED.c` 里的 `OLED_POWER_BY_GPIO` 宏。

**串口**：PA9 (TX) / PA10 (RX)，接板载 USB 转串口，或外接 USB-TTL。

### 烧录

**方式一：STM32CubeIDE**

1. 用 STM32CubeIDE 打开本工程
2. 编译（Ctrl+B）
3. 烧录（Run / Download）

**方式二：命令行**

```bash
arm-none-eabi-gcc ... -T STM32F407ZGTX_FLASH.ld ...

STM32_Programmer_CLI -c port=SWD -w build/app.elf -v -rst
```

### 对话

用任意串口助手（或 PuTTY、minicom）：

| 设置 | 值 |
|---|---|
| 波特率 | **115200** |
| 数据位 | 8 |
| 校验 | 无 |
| 停止位 | 1 |
| 编码 | **UTF-8**（选 GBK 会乱码） |
| 发送 | **必须带换行**（`\n` 或 `\r\n`） |

发一句 `你好`，OLED 第 2 行显示你说的，第 3 行显示小肥鱼的回复，串口也会收到回复。

### 试这几句

```
你好
我心情不好
现在几点
十二减四
7除2
苹果好吃吗
```

---

## 技术架构

### 一句话

**字符哈希特征 + 两层 MLP + 置信度兜底 + 规则算术**

### 数据流

```
串口 "你好\n"
   │
   ▼ USART1 中断 → 环形缓冲
归一化      UTF-8 解码 → 纠错（泥嚎→你好）→ 去标点（保留 +-*/）
   │
   ▼
特征提取    单字 + 前邻词组 + 后邻词组 + 三字组合 → 1024 维 → L2 归一化
   │        再叠加上一句的特征（权重 0.35）当上下文
   ▼
前向推理    1024 → 64 (ReLU) → 37
   │        softmax(z / 温度)  ← 温度 0.6
   ▼
判决        最高概率 < 0.35 ？ 是 → 改判"不懂"
   │        是算术题？ 是 → 交给 parse_calc 真去算
   ▼
输出        OLED 显示 + 串口回传
```

### 四个关键设计决策

**① 为什么用字符哈希而不是分词**

MCU 装不下词表。哈希直接把 Unicode 码点取模：

```python
feat[ord(ch) % 1024] += 1
```

不需要任何外部字典文件，任意汉字都能映射到 1024 个桶之一。

**② 为什么加三字组合（trigram）**

单字 + 双字词组抓不住「谢谢你」「别难过」这种**固定搭配**。加 trigram 后准确率从 72.8% 跳到 **80.7%**。

**③ 为什么算术要"模型判断 + 代码计算"**

分类模型对同一个类只能返回同一句回复，没法把 `3+5` 和 `7+2` 算出两个不同答案。

所以模型只负责判断"这是算术题"，真正的计算交给 `parse_calc()`。

> 通用套路：**模型干它擅长的（分类），代码干它擅长的（精确计算）**。

**④ 为什么有两道保险防"乱答"**

1. **训练时**：数据里塞大量杂句进 `unknown` 类
2. **推理时**：最高置信度低于 0.35 就强制改判"不懂"

单靠第一道不够——模型仍可能把没见过的怪句子硬塞给某个已知类。

---

## 项目结构

> ⚠️ **注意**：本项目由两部分代码组成，当前分别放在两个目录。开源时建议合并成一个仓库（见下方建议结构）。

```
llm-lw/
├── README.md                    ← 本文件
├── LICENSE
├── 小肥鱼项目总结.md              ← 详细技术文档（1157 行）
│
├── firmware/                    ← STM32 固件（来自 LLM_LW_ZGT6/）
│   ├── Core/
│   │   ├── Driver/              ← OLED 驱动
│   │   │   ├── Inc/  OLED.h  OLED_Font.h（3500 字点阵，约 119 KB）
│   │   │   └── Src/  OLED.c
│   │   ├── Inc/
│   │   │   ├── model.h          ← 推理接口
│   │   │   ├── model_weights.h  ← 权重数据（823 KB 文本，68005 个 float）
│   │   │   ├── correspond.h     ← 对话逻辑
│   │   │   ├── bt_fifo.h        ← 串口环形缓冲
│   │   │   ├── usart.h  gpio.h  main.h
│   │   │   └── stm32f4xx_hal_conf.h   ← 注意 HSE_VALUE = 8000000
│   │   └── Src/
│   │       ├── model.c          ← 核心：UTF-8 解码 + 特征 + 前向 + 算术
│   │       ├── correspond.c     ← 收句 → 推理 → 显示/回传
│   │       ├── main.c  usart.c  bt_fifo.c  gpio.c  delay_us.c
│   │       └── stm32f4xx_it.c  stm32f4xx_hal_msp.c  ...
│   ├── Drivers/                 ← ST HAL 库（见下方许可说明）
│   ├── STM32F407ZGTX_FLASH.ld
│   └── LLM_LW_ZGT6.ioc          ← CubeMX 工程文件
│
└── model/                       ← PC 训练代码（来自 PyLearn/LLM-LW/）
    ├── dataset.py               ← 37 类意图数据 + 特征提取 + 算术解析
    ├── model.py                 ← 纯 numpy 的两层 MLP
    ├── train.py                 ← 训练流程 + 早停 + 温度 + 阈值 + 聊天模式
    ├── export_weights.py        ← 导出 C 头文件 + 对拍验证
    └── model.npz                ← 训练好的权重（275 KB）
```

---

## 如何扩展

### 加一个新意图

编辑 `model/dataset.py`，在 `INTENTS` 里加一条：

```python
{
    "tag": "hungry",                                 # 类别名（英文，会进 C 端）
    "patterns": ["我饿了", "肚子饿", "有吃的吗"],      # 建议 10 条以上
    "response": "快去吃点什么吧",
},
```

可选：往 `EXTRA_RESPONSES` 加两条备选，让回复会轮换：

```python
EXTRA_RESPONSES = {
    ...
    "hungry": ["别饿着", "去吃饭吧"],
}
```

然后重新训练和导出：

```bash
python train.py
python export_weights.py
cp model_weights.h ../firmware/Core/Inc/
```

重新编译固件即可。

> 💡 如果新回复用到了字库中没有的汉字，需要重新生成字模。

### 加一句说法（最快）

只想让模型多认识一种说法？直接加进 `EXTRA_PATTERNS`：

```python
EXTRA_PATTERNS = {
    "praise": [..., "你太强了"],
}
```

这是提升泛化能力最省事的方式。

### 改回复

改 `INTENTS` 里的 `response` 或 `EXTRA_RESPONSES`。

### 换引脚

`firmware/Core/Driver/Src/OLED.c` 顶部：

```c
#define OLED_SCL_PIN    GPIO_PIN_10
#define OLED_SDA_PIN    GPIO_PIN_11
#define OLED_VCC_PIN    GPIO_PIN_7
#define OLED_GND_PIN    GPIO_PIN_6
```

---

## 已知局限

**这几点请务必知道**：

**① 准确率约 80%，不是 95%+**

37 个类别、1200 条样本，这个数据量下 80% 已经是"字符哈希 + 小 MLP"架构的上限。**补数据是唯一有效的继续提升手段**。

**② 否定词处理不好**

`我开心` 和 `我不开心` 的特征几乎一样（模型只看见"开心"）。虽然专门补了一批负面说法缓解，但这是个结构性缺陷。

**③ 没有真正的对话记忆**

虽然有"上一句特征加权"的轻量上下文机制，但它只是辅助信号，不是真正的对话状态跟踪。

**④ 同一意图的回复是固定的**

每个意图只有 3 条备选回复轮换使用。同一类别下，回复内容不随输入变化。

**⑤ 算术只支持两个数、一个运算符**

`1+2+3` 这种连算不支持。

**⑥ 哈希碰撞**

不同汉字可能落到同一个特征桶里，造成偶发的误判。

---

## 构建环境

### 训练侧（PC）

```
Python  3.9+
numpy   2.x
```

可选（只用于实验对比，**推理不需要**）：

```
scikit-learn    # 对比随机森林等方案
```

```bash
pip install numpy
```

### 固件侧

| 工具 | 版本 |
|---|---|
| STM32CubeIDE | 1.19.0 |
| ARM GCC | 13.3（CubeIDE 自带） |
| STM32Cube FW_F4 | 1.28.3 |

**关键编译选项**：

```
-mcpu=cortex-m4 -mfloat-abi=hard -mfpu=fpv4-sp-d16
-O0 -ffunction-sections -fdata-sections -Wl,--gc-sections -lm
```

> ⚠️ **`HSE_VALUE` 必须是 8000000**（8 MHz 晶振）。如果板子上是 25 MHz 晶振，改成 25000000 并调整 PLL 参数。

---

## 许可

- **本项目的原创代码**（`model/` 全部、`firmware/Core/` 下的自写部分）：**MIT License**
- **`firmware/Drivers/` 下的 STM32 HAL 库**：属于 STMicroelectronics，遵循其原有许可（BSD-3-Clause），版权归 ST 所有

使用本项目时请保留上述两部分的版权声明。

---

## 详细文档

想深入了解每个环节的原理、完整的实验结果、踩过的坑：

📖 **[小肥鱼项目总结.md](小肥鱼项目总结.md)**（1157 行）

内容包括：
- 特征提取的完整原理（为什么是哈希、为什么加 trigram）
- 反向传播的手工推导
- 温度缩放和置信度阈值的数学
- MCU 端内存布局（为什么必须 `static` / `const`）
- 全部调优实验记录（含**被证伪**的方案）
- 8 个踩坑的完整排查过程

---

## 致谢

- [STM32CubeMX / HAL](https://www.st.com/en/development-tools/stm32cubemx.html) — ST 的官方开发工具链
- 字模来自 **Windows 宋体 simsun.ttc**

---

<p align="center">
  <i>从「把 OLED 驱动移植到 HAL」开始，到「板子上真的能聊天」结束。</i><br>
  <i>80% 不是一个漂亮数字，但每一步提升都知道为什么。</i>
</p>
