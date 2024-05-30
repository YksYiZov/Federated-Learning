# 联邦学习的实验
An experiment about Federated-Learning from this [repository](https://github.com/TsingZ0/PFLlib).

## 初步学习
初步学习发现，框架代码可以分为四个部分。

程序流程、模型类、服务类、客户类。
### 训练框架理解
#### 流程
- 使用argparse库完成训练时参数的配置。
- 根据输入的参数，选择对应项模型和算法，配置相应数量的客户信息。
- 开始训练，并输出训练结果。
- 完成训练，保存模型。

#### 模型类
其中主要保存了训练需要的模型，进行基础的模型设置。（神经网络的模型架构）

#### 服务类
负责与客户端与服务端交互的中间类，包括模拟客户端训练模型，上传客户训练参数，服务端模型参数更新，模型的下发等过程。

#### 客户类
训练本地模型，上传模型参数。

### 详细训练流程
- 下发模型
  - 先将模型下发给客户。
- 模拟客户训练
  - 随机挑选某些客户，模拟客户完成了模型更新。
- 模拟客户上传
  - 训练出新模型的客户将模型上传到服务端，根据权重更新模型。
- 重复此流程

### 一些问题
#### 大量客户的并行问题
框架中展示了20个用户同时使用的情况，但是如果训练方法得到推广，如何处理更多用户的需求？可能涉及到并行算法的设计。
#### 权重算法
观察到权重算法存在疑问，权重信息是由数据所占比例决定的，但是这个数据并非总数据，而是每轮新上传模型的客户的总数据，这可能会导致一些问题，因为假如有一个客户每轮都上传，其他客户上传次数少，但是数据量大，那么每轮上传总有几轮会导致他的梯度权重过大，模型向他的方向倾斜。

比如A用户每次都上传，上传1单位数据，B用户每隔3次上传一次，上传10单位数据。按照理想情况，模型权重收到双方数据的影响的比例应该维持在`3:10`

但是实际上模型权重收到的影响应该是`1:0`,`2:0`,`(3:0)/11+(2:10)*10/11=23:100`。

所以如果出现这样的更新频率不同，最终模型权重会向更偏向更新频率高的客户。

## 基于GAN的FL
### 概述
基于GAN的FL与传统的FL的最大区别在于利用训练好的生成器生成合成数据，这些合成数据可以帮助训练，通常这些生成器部署在客户端。
### 一般流程
- 客户端训练GAN，上传到服务端，服务端下发给所有客户端
- 客户端同时使用其他客户端训练得到的数据分布高度不同的生成器同步生成假数据来进行训练
- 正常的FL训练过程
  

### Federated Learning with GAN-based Data Synthesis for Non-IID Clients
在这篇论文中，最大的创新点在于
- 服务端保存了合成数据集
- 客户端训练使用混合数据，不依赖其他的生成器

#### 基本流程
（有预训练的生成器）
- 下载新的模型和合成数据集
- 更新合成数据集标签
- 本地训练，上传模型
- 服务器用上传的模型给合成数据标注，高置信度数据加入合成数据集
- 更新模型

#### 合成数据集
- 数据集由服务端持有
- 数据集数据均为合成数据
- 数据集数据的标签均为高置信度标签
- 数据集数据标签均由产生他的本地模型产生

#### 客户端训练
- 下载服务端的合成数据集
- 将合成数据集中的数据和真实数据按一定比例混合
- 用混合的数据进行训练

### FLGAN: GAN-Based Unbiased Federated Learning Under Non-IID Settings
在这篇论文中，最大的创新点在于
- 使用一种新的选择策略在保护隐私的同时选取合适的生成器
- 使用额外的生成器来解决非独立同分布问题

#### 基本流程

- 训练GAN，并上传数据分布，生成器权重
- 根据数据分布，为每个客户端选择分布最不同的生成器
- 本地生成器和选择的生成器与真实数据一起训练本地模型
- 上传模型参数
- 更新模型

#### 选择生成器

- 不同模型上传本地的数据分布
- 根据数据分布选取最不同的几个生成器（使用cos来判断分布相似度）
- 将选择的生成器给到客户端（加密的生成器）

#### 额外生成器
- 额外生成器可以模拟异构的数据的生成
- 同时使用本地数据和生成的数据，使得不同客户端的数据倾向于同分布

### 我的理解
#### GAN的优势 
提出GAN用于FL的原因可能是希望通过一个优秀的生成器来制造更多的数据，因为受制于隐私，不同客户端数据不可能互通，但是这样数据之间的异构性比较严重，所以通过这种人造数据的形式来模拟一个数据互通的环境，建立在一个相当优秀的生成器的条件下，这个方法确实可以取得很好的效果。

#### GAN用于FL开发的方向
充分利用GAN可以训练出优秀的生成器的方法，利用这个生成器来帮助完成数据的同分布化操作，但是目前看来，只用到了GAN的生成器而没有用到判别器，未来是否考虑将判别器加入考量？比如根据样本真假的置信度增加一个权重？

### 一些困惑
#### 关于混合数据
数据混合有什么作用？为什么要将真数据和生成数据按一定比重混合？这样的得到的数据是否有意义。

生成的数据本身可能更利于机器学习，混合数据相当于让真实数据更贴近机器学习中的优质数据，可能更便于模型的收敛，但是这样会对模型的真实效率有什么影响吗？

同时有可能输入的合成数据量远远大于真实数据，按论文中写到至少合成数据和真实数据的比例是`1：1`，甚至更大，那么这个模型就只能建立在生成器模拟真实数据足够好的情况上了。

#### 生成器的训练
无论使用的合成数据来源于什么，都会面临一个新数据与原有数据异构的问题，那么在新的更加适合新数据模式的生成器中，他是否会在训练中忘记从老数据的学到的经验？而且面临新数据模型的训练明显会滞后一些。

## 实际实验
### FebAvg 实验
#### FebAvg 实验结果

| 有无GAN | 轮次 | 时间每轮 | 最好准确率 |
|------|------|------|------|
| 有 | 100 | 8s | 0.883 |
| 无 | 100 | 1s | 0.698 |

#### FebAvg 修改代码内容
##### 生成器
放在`FBLlib/system/decoder/generators`下，使用`decoder/train_generator.py`可自动化训练生成器，保存在`decoder/generators`中，为了使用模型需要利用`decoder/models`中的模块，需要移动请注意添加模块搜索路径。

没有使用GAN的训练方法训练条件变分自编码器(CVAE)，只是普通的CVAE训练方式。
##### 服务
新增了`serverganavg.py`在`FBLlib/system/flcore/server`中，可以手动切换要测试的类是带有生成器的客户端或者不带生成器的客户端。
更换`serverganavg.py`中`set_client(ClientObj)`中的`ClientObj`即可，`ClientGanAvg`为带生成器的客户端，`ClientNoGanAvg`是不带生成器的客户端。
##### 客户
新增了`clientganavg.py`和`clientnoganavg.py`在`FBLlib/system/flcore/client`中。
两者的区别在于是否使用生成器作为训练集的补充。

#### 思考
没办法比上正常训练MNIST的正确率，原因可能在于生成器生成的图像仍然不够真实，下一步会尝试用GAN方法来训练生成器。

数据集不够复杂，异构不够明显，或许在更复杂的数据集中可能效果更好。

有GAN的情况下前期收敛更快。

PS：Batch-Size对结果可能有较大影响。
### FebProx 实验
#### FebProx 实验结果
| 有无GAN | 轮次 | 时间每轮 | 最好准确率 |
|------|------|------|------|
| 有 | 100 | 10s | 0.868 |
| 无 | 100 | 9s | 0.908 |

#### FebProx 修改代码内容
##### 生成器
放在`FBLlib/system/decoder/generators`下，使用`decoder/train_generator.py`可自动化训练生成器，保存在`decoder/generators`中，为了使用模型需要利用`decoder/models`中的模块，需要移动请注意添加模块搜索路径。

没有使用GAN的训练方法训练条件变分自编码器(CVAE)，只是普通的CVAE训练方式。
##### 服务
新增了`serverganprox.py`在`FBLlib/system/flcore/server`中，可以手动切换要测试的类是带有生成器的客户端或者不带生成器的客户端。
更换`serverganprox.py`中`set_client(ClientObj)`中的`ClientObj`即可，`ClientGanProx`为带生成器的客户端，`ClientNoGanProx`是不带生成器的客户端。
##### 客户
新增了`clientganprox.py`和`clientnoganprox.py`在`FBLlib/system/flcore/client`中。
两者的区别在于是否使用生成器作为训练集的补充。

#### 思考
使用GAN对于Prox方法提高不明显，或者说几乎不提高。可能的原因是，尽管数据异构，但是MNIST间可能还是可以通过简单的线性关系拟合出更好的模型，可能对于更复杂的模型或者数据集会体现出优势。

有可能因为Prox方法加权反而使得本地效果变差一些。