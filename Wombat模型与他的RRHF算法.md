![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-14-14-33-11-image.png)

# RRHF算法

主要的贡献是改进了RLHF中PPO迭代训练环节的高时间复杂度

### 计算每个policy的输出（概率似然）

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-14-14-38-22-image.png)

### 计算rank loss

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-14-14-38-49-image.png)

根据rank排名，低rank的log score要向高rank看齐。

### 监督微调环节

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-14-14-40-57-image.png)

以rank最高的y作为groundtruth，对P_pi进行微调。

### 最终Loss

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-14-14-42-11-image.png)

总结来说就是，让p_pi的输出尽可能接近高rank response，切不断的用高rank的[request, response]对去训练微调P_pi.

# wombat训练配置

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-14-14-35-37-image.png)

policy：ChatGPT, text-davinci-003, LLaMA, Alpaca

dataset： 来自Alpaca数据集的queries

RM：使用ChatGPT作为reward model
