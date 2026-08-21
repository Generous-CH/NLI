大型语言模型在命题逻辑和一阶逻辑方面表现出色，但目前尚无基准测试来检验它们能否对必然性、可能性、义务或许可进行推理——而这些正是模态逻辑和义务逻辑的范畴。我们推出 ModalBench，这是首个基于kripke语义学的基准测试，用于评估大型语言模型（LLMs）的模态推理能力。 ModalBench 包含 1,500 道题目，涵盖五个模态系统（K、T、S4、S5 和义务逻辑 D）、三个难度等级以及两种呈现形式（形式符号和自然语言），每道题目均配有经过计算验证的真实答案。 我们采用三种提示策略对三种大语言模型进行了评估（共计27,000次推理），并发现：(1) 在“思维链”提示策略下，模型准确率达到73%–91%，其中K系统（无限制可及性）难度最大，准确率为65%–75%； (2) 大型语言模型在自然语言下对模态的推理能力优于形式化记法——三款模型中有两款在叙事呈现方式下的得分显著更高，其中Qwen3-235B在自然语言下的表现优势达18.4个百分点（p < 10−99）； (3) “世界枚举”提示起到了支架作用：它能使较弱的模型提升9–11个百分点，但对于最强的模型则并非必要——该模型在零样本提示下准确率可达95.9%； (4) 所有模型均表现出隐性的 S5 偏差，在欧几里得公理不成立的系统 K 中，该偏差导致该公理被过度应用，偏差幅度最高达 +56 pp。


table1
k 无任何约束，T自反，S4自反+传递，S5自反+传递+对称
D序列性
**系统 K（无约束）对所有模型最难（65–75% 准确率）**，因为模型无法依赖任何“熟悉的结构性规律”（如自反或传递）来辅助推理。**所有模型在系统 K 中都会过度应用“公理 5（欧几里得性质）”**，即默认认为世界之间存在对称/等价关系，哪怕系统 K 根本没有这个约束。这正是 Table 1 中“S5 约束最严格”这个事实所对应的模型行为偏差。系统 D（道义）用“序列性”替代了“自反性”，因为义务（OB）不要求当前世界一定满足该义务（比如“你应该不偷窃”在当前世界可能为假），但要求至少存在一个“道德上可及”的世界使其成立。

5 个 modal systems，3 个 difficulty tiers，23 类 axioms，5 类 paradox patterns


### 阶段 1：生成 Kripke 框架（Frame Generation）
- **目标**：为每个模态系统（K, T, S4, S5, D）生成一个符合其约束条件的可达关系 RR。
- **方法**：
    - 使用**受约束的随机生成**（constrained randomness），即随机生成有向图，但强制满足系统特定的性质（例如，T 系统必须自反，S4 必须自反+传递，S5 必须自反+传递+对称）。
    - 生成后立即进行**形式化验证（post-hoc validation）**，用算法检查该图是否真的满足所需性质。如果不满足，则丢弃重新生成。 
- **框架规模**：每个框架包含 **2 到 7 个世界**（平均 4.5 个），覆盖从小型到中型的推理空间。
### 阶段 2：命题赋值采样（Valuation Sampling）

- **目标**：为每个世界分配原子命题（p,q,rp,q,r 等）的真假值。
- **命题数量**：每个框架使用 **2 到 5 个原子命题**，确保有足够的组合复杂度来构造不同难度的公式。
- **采样方式**：随机采样，但不做额外平衡（因为最终的标签平衡由公式选择阶段来保证）。
### 阶段 3：公式选择（Formula Selection）
- **目标**：为每个框架挑选合适的模态公式，覆盖三个难度层级。
- **公式池**：
    - 论文维护了一个预定义的**公式模板库**，包含 **23 种不同的公式类型**（如 □p□p, ⋄(p∧q)⋄(p∧q), □⋄p□⋄p, □p→p□p→p 等）。
    - 每个公式模板都预先**标记了它对应的公理名称**（如 T, 4, 5, B, K, D, Ross, Chisholm 等），方便后续做逐公理的诊断分析（第 5.5 节）。
- **选择策略**：根据 Tier 层级从对应池中抽取公式，确保每个系统 × 每个 Tier 的组合都有 200 道题。
### 阶段 4：标准答案计算（Ground Truth Computation）
- **目标**：递归地应用公式 (1) 和 (2)（即 Kripke 语义），计算出该公式在当前 Kripke 模型中的真实真假值。
- **实现方式**：直接编写程序（非 LLM），从最内层的子公式开始，逐层向外求值，直到得到最终结果。
- **标签平衡**：
    - 生成过程会**强制保持 50% True / 50% False 的全局平衡**（按每个 cell，即“系统 × Tier”为单位）。  
    - 论文特别提到，如果不做这种强制平衡，简单启发式策略（如“看到 □□ 就猜 True”）会达到 47.1% 的准确率，说明这种启发式虽然略高于随机，但远不能成功。通过强制平衡，**任何不基于真正推理的简单策略都会回归到 50% 基线**，从而让模型表现更有区分度。
|**Tier 1**|简单（Easy）|**单个模态算子**，嵌套深度为 1，只含原子命题和简单布尔连接词。|□p□p, ⋄(p∧q)⋄(p∧q), □(p∨¬q)□(p∨¬q)|
|**Tier 2**|中等（Medium）|**嵌套模态**，深度为 2，需要递归求值。|□⋄p□⋄p, ⋄(□p∧⋄q)⋄(□p∧⋄q), □(p→⋄q)□(p→⋄q)|
|**Tier 3**|困难（Hard）|**公理交互公式**或**道义悖论**，需要同时理解多个模态性质的组合，或处理反直觉的义务冲突。|□p→□□p□p→□□p（公理 4），⋄p→□⋄p⋄p→□⋄p（公理 5），以及 Chisholm、Gentle Murderer 等悖论场景。|


## 双轨制呈现（第 3.3 节）
这是一个非常巧妙的设计：**每个问题都生成两个版本，它们共享完全相同的 Kripke 语义和标准答案，但表达方式完全不同。就是formal track 和 natural language track**
一个使用标准的集合论和模态逻辑符号。明确列出世界集合 WW、可达关系 RR、赋值 VV，公式也用 □,⋄□,⋄ 表示。
一个将同一 Kripke 模型嵌入一个叙事场景中（例如“爱丽丝在探索房间，房间之间有单向观察窗，可达关系 = 可见性”）。模态算子转换为日常用语：“必然”=“在所有可观察到的房间里都成立”，“可能”=“在至少一个可观察到的房间里成立”。
**两个模型（Llama, Qwen3）在 NL 轨道上显著更好**（Qwen3 甚至高出 18.4 个百分点），这说明“形式符号”反而成为了阻碍，而**自然语言中的空间/物理隐喻**激活了模型更擅长的推理路径。

## 第 4 节：实验设置（Experimental Setup）
三种模型，三种提示策略:Zero shot ,Chain of thought(在问题后加上“Think step by step.”（但不指定具体步骤),World-Enumeration CoT提供一个**结构化的分步模板**（论文附录 B），明确指示模型：  列出所有可达世界,对每个子公式在每个可达世界求值,用 ∀（全称）或 ∃（存在）聚合,递归处理嵌套模态,最后输出“True”或“False”
- chain-of-thought 是“自由发挥”的逐步推理，模型自己决定怎么组织思路。
- World-Enumeration 是“强制模板”，模型必须按 Kripke 语义的机械步骤来走，没有自由发挥空间。
**题目总数**：3,000 道（1,500 个独特 Kripke 实例 × 2 个轨道：形式化 + 自然语言）。**模型数**：3 个。**提示策略数**：3 种。**总推理次数**：3 个模型×3 种策略×3,000 道=27,000 次推理

打lable:Formal Semantics / Automated Verification
我现在是：Finite Model
    ↓
集合
    ↓
Generalised Quantifier
    ↓
True / False

ModalBench：

Kripke Model

    ↓

Accessible Worlds

    ↓

Modal Semantics

    ↓

True / False

你未来：

Kripke Model

    ↓

Modal Evaluation

    ↓

得到一个集合

    ↓

Generalised Quantifier

    ↓

True / False

Formal track：LLM 到底是不会 Modal Logic，还是不会读 Modal Logic 符号？

deontic logic：自然语言里大量存在应该，必须，也许，这些天然带有modality

模型可能会：错误地把较弱 Modal Logic 中不成立的东西，当成在更强 S5 条件下成立的东西。因为语言模型可能从训练语料中学到：Modal logic = □ / ◇但它没有真正理解：R的 frame properties。也就是说它可能知道：□P大概是什么意思，但不知道：到底有哪些 world 可以访问？于是它偷偷假设：所有世界都差不多,或者：关系非常强,这就产生了 S5-like reasoning。这和你的研究非常有关系假设你的：Most students must be scientists.变成：Mostx(Student(x),□Scientist(x))那么 LLM 可能会犯：must = all worlds这种简单错误。但真正需要的是：∀v(wRv)也就是: **所有 accessible worlds**,不是：所有可能世界。这就是 ModalBench 所暴露出的核心问题。

一些模型在 Natural Language track 上表现比 Formal track 更好。符号/自然语言推理，不是一种能力

核心 pipeline：

定义 Modal Systems

        ↓

K / D / T / S4 / S5

        ↓

生成公式

        ↓

生成 Kripke semantics

        ↓

自动计算 ground truth

        ↓

Formal / Natural Language

        ↓

构造 benchmark

        ↓

LLM evaluation

        ↓

Accuracy / Error Analysis

S5bias默认的accessibility relation比实际条件更强

我能从这篇论文学到什么：
（1）kripke Model的程序表示
  (2)Recursive Modal Evaluator：递归模态求值，然后支持：然后支持：Atom，Not，And，Or，Implication，Box，Diamond
（3）自动生成数据：我已经有了quantifier generator，可以扩展Modal operator generator
  (4)自动label系统最终应该：
Natural Language

      ↓

Logical Representation

      ↓

Kripke + GQ Evaluator

      ↓

True / False
(5)你可以定义：
Level 1:
Most students are scientists.
Level 2:
Most students must be scientists.
Level 3:
Most students must possibly be scientists.
Level 4:Most students must be scientists
and few scientists can be engineers.
你的真正难点在：
Qx□P(x) vs □QxP(x)​以及：Qx◊P(x) vs ◊QxP(x)
比如：Most students must be scientists.
你定义：
Mostx​(Student(x),□Scientist(x))
然后再定义：In every possible world, most students are scientists.为：
□Mostx​(Student(x),Scientist(x))
现在问：Mostx​(Student(x),□Scientist(x))≡?□Mostx​(Student(x),Scientist(x))
一般并不等价。
这就是一个非常漂亮的研究问题：
Generalised Quantifier 和 Modal Operator 的 scope interaction 是否被 LLM 正确理解？这已经明显超出了 ModalBench 本身。