# SafeSpeech: Robust and Universal Voice Protection Against Malicious Speech Synthesis

> **作者**：Zhisheng Zhang, Derui Wang, Qianyi Yang, ... , Jie Hao, and Yixian Yang (BUPT, CSIRO Data61, NUS, HKU)  
> **论文链接**：[arXiv:2504.09839](https://arxiv.org/abs/2504.09839)  
> **代码**：[github.com/wxzyd123/SafeSpeech](https://github.com/wxzyd123/SafeSpeech)  
> **阅读时间**：2026-05-24  

---

## 概要

SafeSpeech 是一种**训练阶段**的语音保护方法：在用户音频上传到互联网前，给它加入难以察觉的扰动，使得攻击者即便爬到这段音频、微调 TTS 模型，也无法克隆出该说话人的音色（SA1），同时合成质量低到不可用（SA2）。核心创新是 **SPEC（Speech PErturbative Concealment）**——选择跨模型通用的 mel 音频损失作为"关键目标"，并用 KL 散度引导模型把"该说话人的音频"误学成"高斯噪声"。

**区别于先前方法**：AntiFake / AttackVC 走对抗样本路线（max loss）只能挡零样本克隆；SafeSpeech 走不可学习样本路线（min loss），同时挡微调克隆和零样本克隆。

---

## 解决问题

### 威胁模型

攻击者爬取公开网络上的目标受害者语音 → 微调 TTS 模型 → 克隆出受害者声音 → 实施电信诈骗、绕过声纹锁等。

真实案例：2019 年英国 CEO 被深伪语音骗走 24.3 万美元。

### 现有方法的缺陷

1. **场景局限**：AntiFake 等只防御零样本克隆，但攻击者可以改用微调式克隆绕过；
2. **质量未防**：先前方法只让合成音色不像目标，但合成质量仍高 → 攻击者仍可用伪造音频进行其他不法行为，如寻找相似音色的新受害者以及掩盖真实身份等；
3. **鲁棒性弱**：对抗扰动易被数据增强、对抗训练等方法去除。

---

## 方法核心

### 整体框架

```
原始音频 x → 加扰动 δ → 受保护音频 x' = x + δ → 上传
                  ↑
        通过代理模型 M 反向优化生成
                  ↑
        目标函数 L = L_SPEC + α·L_perception
```

### 关键设计 1：Pivotal Objective（关键目标选择）

**问题**：TTS 模型损失项多达 8 个（如 BERT-VITS2），逐个优化既低效又难以跨模型迁移。

**作者的洞察**：选一个满足三条原则的"关键目标"：

- (a) 能通过扰动 δ 优化（损失对 δ 可导，即可求梯度，且非零）
- (b) 跨 TTS 模型通用（不依赖特定架构）
- (c) 易优化（收敛快或信息熵高）

选定关键目标函数：

$$\mathcal{L}_{mel} = \Vert \hat{x}_{mel} - x'_{mel} \Vert_1$$

因为所有 TTS 都输出语音，都能算 mel 频谱 ℓ1 距离，且收敛速度最快（实验中从 98.8 降到 16.7）。

### 关键设计 2：SPEC（Speech PErturbative Concealment）

仅靠关键目标还会有残留的语音可懂度。作者引入"噪声引导"机制：

$$\mathcal{L}_{noise} = D_{KL}(\hat{x}_{mel}, z_{mel}) + \Vert \hat{x}_{mel} - z_{mel} \Vert_1$$

其中 z 是高斯噪声。这一项让 TTS 模型学到"输出趋近噪声"。

完整 SPEC：

$$\mathcal{L}_{SPEC} = \mathcal{L}_{mel} + \beta \mathcal{L}_{noise}$$

**反直觉点**：mel 损失让合成接近"受保护音频"，noise 损失让合成接近"噪声"——看似冲突，实则**协同**：两者方向一致（都是"让模型学不到说话人特征"），只是机制不同。这就是为什么 β 在 5 个数量级范围内都有效——两个协同项的权重不敏感。

### 关键设计 3：感知优化

仅靠 ℓp 范数约束扰动幅度不足以保证人耳不可察觉。作者引入：

$$\mathcal{L}_{perception} = \mathcal{L}_{stoi} + \mathcal{L}_{stft}$$

- STOI：时域可懂度
- STFT：频域结构相似度，特征获取

其中 STFT 损失：

$$\mathcal{L}_{stft} = \Vert STFT(x+\delta) - STFT(x) \Vert_2$$

### 最终目标

$$\mathcal{L} = (\mathcal{L}_{mel} + \beta \mathcal{L}_{noise}) + \alpha (\mathcal{L}_{stoi} + \mathcal{L}_{stft})$$

最小化此目标 + ℓp 范数约束 → 生成扰动 δ。

---

## 为什么是"最小化损失"？

| 路线 | 数学 | 直觉 | 防御场景 |
|------|------|------|---------|
| **对抗样本**（AntiFake） | max L | 让模型在推理时"考砸" | 零样本克隆 |
| **不可学习样本**（SafeSpeech） | min L | 让模型以为自己"学会了"，提前停止学习 | 训练时克隆 + 零样本克隆 |

**为什么不可学习样本能挡训练？** 因为训练损失天然被防御者压低 → 攻击者训练时梯度信号微弱 → 模型参数几乎不更新 → 学不到说话人特征。

---

## 实验亮点

### 主结果（表 1）

在 LibriTTS + BERT-VITS2 上：

- WER：clean 24% → SPEC 99.6%（合成几乎不可懂）
- SIM：clean 0.604 → SPEC 0.204（远低于克隆阈值 0.25）
- 5 个 TTS 模型上一致有效（强迁移性）

### 零样本场景（图 4）

即便扰动只在 BERT-VITS2 上生成，迁移到 F5-TTS、XTTS、FishSpeech 等零样本模型上也有效（F5-TTS SIM 从 0.885 跌到 0.094）。

### 用户研究

- 受保护后合成语音 MOS = 1.07（接近"完全听不清"）
- 98.3% 的参与者认为受保护音频和原音频是**同一个人**说的 → 扰动不可察觉

### 鲁棒性测试

- ✓ 扛得住 SG 去噪、DEMUCS 深度去噪
- ✓ 扛得住 9 种数据增强（含 AudioPure 扩散净化）
- ✓ 扛得住对抗训练（即使 ρa > ρu）
- ✓ 扛得住干净数据微调（反而被覆盖）
- ✓ 真实物理世界测试通过（50 dBA 仍有效）

---

## 我的批判性思考

### 1. "real-time" 定义不严谨

论文反复强调 real-time，但实际有 **14 秒前置时间**（前 14 秒说话不受保护）。

### 2. 硬件门槛被淡化

14 秒前置是在 NVIDIA A800 上测的。普通用户笔记本/手机能不能跑？论文没提。如果需要云端 GPU，那"语音保护"和"上传声音到云"本身就是矛盾的。

### 3. 评估指标的人本性不足

- 用户研究只有 80 人，且只测了**英语**；
- 客观 SIM < 0.25 阈值的设定来自 ECAPA-TDNN，但**人耳判别阈值**和它一致吗？没验证；
- 跨语言、跨文化的泛化未测。

### 4. 历史音频无法追溯保护

SafeSpeech 只能保护**上传前**的音频，SafeSpeech 救不了过去。这是个根本性局限，论文未讨论。

### 5. 一些写作上的小瑕疵（细读才发现）

- 6.4 节正文出现 "SIM = 0.21" 但图 5(a) 没画 SIM → 数据藏在文字里
- 7.2.1 节 WER = 100% 看似"防御无效"，实际是模型架构权重加载问题
- "ℓp 范数约束在 4.1 节"——其实在 4 节开头的问题形式化里

---

## 可能的延伸方向

如果做后续工作，几个切入点：

### 方向 A：跨语言泛化

论文只测了英语 → 中文、日语等声调/音节结构不同的语言效果如何？

### 方向 B：历史音频追溯保护

能否设计某种"被动声纹保护"机制，使已上传到网上的音频在被克隆时**也能触发噪声生成**？

---

## 引用与延伸阅读

### 重点参考文献（论文引用）

- AntiFake (CCS 23) - 对抗样本路线代表作
- Unlearnable Examples (ICLR 21) - 不可学习样本开山之作
- Mitigating Unauthorized Speech Synthesis (LAMPS 24) - 同作者前序工作

### 拓展的相关论文

- [ ] *Adversarial Examples Make Strong Poisons* (NeurIPS 21) - 理解 PAP 路线
- [ ] *VSMask* (WISEC 23) - 同领域实时方法
- [ ] *Image Shortcut Squeezing* (ICML 23) - PAP 攻击 vs 防御

---

## 阅读笔记信息

- **第一次精读**：2026-05-24
