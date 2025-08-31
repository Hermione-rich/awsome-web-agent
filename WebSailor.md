# WebSailor

##### 介绍

- Motivation：训练出具备超强Deep Search能力的开源Web Agent

  - 不足：目前训练集的难度一般，让高中生刷小学题显然不能提升能力

  - 解决：首先创建了SailorFog-QA数据集，包含[模棱两可的query-有难度的answer]。

    ​	    再采用两阶段SFT-RLVR高效训练。

- 方法描述

  - 数据集创建

    - 构建基于fuzzy entity的KG：先从Wikipedia找一批fuzzy entity，然后去互联网检索信息，抽取相关联的实体和关系，得到一个小KG，然后随机采样实体节点，继续联网检索相关的实体和关系来扩充KG。以上步骤迭代多次，就会得到一个大规模以模糊实体为基础的KG，

    - 从KG中采样子图并模糊描述：基于KG进行子图采样，然后从子图中构建query和answer，为了让query更难，故意将一些人名地名时间等等模糊化。

    - 创建reasoning trajectory：因为要做sft，所以需要reasoning trajectory。如果直接让已有的推理模型生成，轨迹中会包含大量啰嗦的，具有不必要风格的`<think></think>`内容。

      为此，先让模型生成轨迹，再把thinking部分去掉，只保留tool调用和返回结果，用另一个llm专门根据tool调用和返回结果生成thinking，这样能得到精简的reasoning trajectory。

  - 两阶段训练

    - 拒绝采样FT：先对训练数据过滤，再做sft，让llm具备TIR的能力。
    - DUPO：GRPO变体，训练前先把那些rollout全对的query去掉，太简单了对训练没帮助也浪费资源；训练过程中，把batch中那些rollout全对or全错的query也删掉。为了补充到batch size，采样复制batch内其他query的方式，这样做比DAPO训练效率提升2-3倍。

- 我的关注：how to 构建 Trajectories

  - 提示一个专家开源LRM生成完整的解题轨迹，包括其原始思路。从整个轨迹中，选择性地丢弃LRM的原始冗长思路，只保留成功的act-obs序列 $(a_0,o_0,a_1,o_1,...)$，这个轨迹代表了解决路径的“what”和“how”，没有“why”。

  - 接下来，重建缺失的"why"。对于每个动作轨迹中的步骤 t， 拥有来自上一步的历史

    $\mathcal{H}_{t-1} = ( \hat{a}_0, a_0, o_0, \dots, \hat{t}_{t-1}, a_{t-1}, o_{t-1} )$

    以及专家选择的动作 $a_t$ 和后续观察 $o_t$。然后，我们提示另一个强大的指令跟随模型 $\pi^*$，生成一个新的思路 $\hat{t}_t$，作为执行动作 $a_t$ 的简洁逻辑理由：

    $$\hat{t}_t \sim \pi^*(\tau | \mathcal{H}_{t-1}, a_t, o_t).$$

    通过对每一步迭代应用这一方法，我们合成了一个完整的、高质量的推理轨迹 $\mathcal{H}_T = (\hat{a}_0, a_0, o_0, \dots, \hat{t}_T, a_T, o_T)$，其中推理清晰且目标导向。为了这次重建，我们使用了另一个大语言模型，并强制采用“短推理链(CoT)”风格。这是一个关键的设计选择，确保最终的推理链对于长期任务来说足够紧凑。该方法使我们能够大规模生成监督数据，培养复杂的推理模式，而不会有直接模仿的负面副作用。

- 实验细节