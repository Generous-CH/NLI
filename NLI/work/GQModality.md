传统的Kripke模型Kripke Model中，M=(W,R,V)
而我改进后的Kripke Model M=(W,D,R,V),其中 D 是所有可能世界共用的单一恒定论域，而赋值函数 V 为每个采样的一元谓词在每个世界中指定 D 的一个随机子集。

related work 中，GQ 没有考虑到自然语言的多样性，modality没有考虑到GQ当中的问题，在一般语言环境下，GQ是很常见的
## 四种定义

| 类型            | 项目生成的公式                  | 实际含义                                                  |
| ------------- | ------------------------ | ----------------------------------------------------- |
| `Q_Box_P`     | \(Q(n,A,\Box B(x))\)     | 在当前世界先选出 \(A\)，统计其中有多少人在所有可达世界都是 \(B\)                |
| `Box_QP`      | \(\Box Q(n,A,B(x))\)     | 每个可达世界分别计算当地同时是 \(A\) 和 \(B\) 的人数，并要求每个世界都达到数量条件      |
| `Q_Diamond_P` | \(Q(n,A,\Diamond B(x))\) | 在当前世界先选出 \(A\)，统计其中有多少人在至少一个可达世界是 \(B\)；不同的人可以由不同世界见证 |
| `Diamond_QP`  | \(\Diamond Q(n,A,B(x))\) | 必须存在某一个可达世界，在该世界当地同时是 \(A\) 和 \(B\) 的人数达到数量条件         |

换成集合表达式：

$$
Q\_Box\_P: Q_n\left(\left\{x\in A(w)\mid \forall v\in R(w),\,x\in B(v)\right\}\right)
$$
$$
Box\_QP: \forall v\in R(w),\quad Q_n\bigl(A(v)\cap B(v)\bigr)
$$
$$
Q\_Diamond\_P: Q_n\left(\left\{x\in A(w)\mid \exists v\in R(w),\,x\in B(v)\right\}\right)
$$
$$
Diamond\_QP: \exists v\in R(w),\quad Q_n\bigl(A(v)\cap B(v)\bigr)
$$
```
├─ QM_stratified.csv                    单句真假判断
└─ QM_scope_contrast.gold.csv           1,024 个核心作用域对照 pair
   ├─ QM_scope_contrast.public.csv
   ├─ QM_scope_contrast_paraphrased.*   每个 pair × 3 种语言表达
   ├─ scope_contrast_splits/                   IID 语言切分
   │  └─ scope_iid_test.prompts.jsonl
   ├─ scope_contrast_template_ood/             模板 OOD 切分
   │  └─ scope_template_ood_test.prompts.jsonl
   └─ scope_paraphrase_annotation.csv   人工语义验证样本
```

QM_stratified:严格笛卡尔分层:1,024 个单句判断样本。4种作用域× 2种数量词× 4个阈值× 2种真假标签× 2种出度= 128个平衡单元
1,024 条意味着每个单元正好 8 条。


QM_scope_contrast ,
1024 个 pair 每个 pair 有两个句子 一共 2,048 个句子真假判断
每个 pair 共用完全相同的：Kripke 模型,当前世界,主语,谓词,宾语,谓词,数量词,阈值
两个句子只改变作用域：两个句子恰好一个为真，一个为假。为什么保证一真一假：随机生成时，很多模型中两种作用域碰巧会得到相同真值。例如两边都真，这并不能证明模型理解了差别。核心数据主动构造谓词外延，让两个读法落在阈值两侧。因此每个样本都确实体现作用域差异。


