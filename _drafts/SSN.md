## Temporal Action Detection with Structured Segment Networks(SSN)

[TOC]

### 前言

* 论文地址：[Temporal Action Detection with Structured Segment Networks](http://arxiv.org/abs/1704.06228)

* Github: [yjxiong/action-detection](https://github.com/yjxiong/action-detection)

最近开始看一些有关视频检测方向的文章，其中比较重要的一篇文章就是Temporal Action Detection with Structured Segment Networks，在这里做一些记录。

首先简单介绍一下什么是视频检测任务：视频检测的主要是给定一段视频，在视频中检测出给定动作的开始时间和结束时间[start, end]，相当于是时间维度上的检测，不要求检测动作发生在每一帧的具体位置，只需要给出发生时间。与视频检测任务很相关的一个任务是视频分类，即给定一段视频，判断视频中的人在做什么。这个任务相对简单，输入视频也相对比较短，更重要的一点是，视频分类的输入视频都经过了trim——保证每段短视频中仅有一个任务，视频检测与视频分类的关系很类似图像检测和图像分类之间的关系

视频分类任务已经发展了挺久了，也有一些不错的文章做出了很好的效果，但是视频检测的指标目前还不是很高。本篇文章（SSN）作为视频检测的Baseline，被许多后续的文章作为比较的对象，因此比较值得一读。

### 主体思路

在本文出现之前，视频的检测很依赖于视频片段的特征提取——也就是将视频分为等长的片段（snippet），每个片段进行训练分类，然后输出片段的特征。然而这类方法的主要弊病是，由于视频片段过短，常常无法包括一个完整的动作，有时候仅仅是该动作发生的一个小步骤，这给训练带来了很大的困难，然而如果增大滑动窗口的稠密程度，又大大降低了计算效率，如何能够像Faster-RCNN的RPN那样，提出一些高质量提名框（Proposal）就称为一个主要问题，可以想象，在视频分类精度已经不错的现状下，如果Proposal的质量提高，必然能够大大提高整个检测任务的精度。

本文就“动作完整性”这个点出发，设计了一个对视频进行“结构分析”的框架，主要思路是，将每一个动作拆分为`开始(start)`，`进行中(cause)`，`完成(ending)`三个阶段(stage)，用这三阶段共同的context，来预测这个proposal是否是完整的，如果某proposal中的不是完整的，就不会把他预测为动作。另外本文也提出了一种比较科学的Proposal预测机制，称为Temporal Actionness Grouping(TAG)，用于生成分类网络训练使用的提名框。

总结一下本文的贡献点：

- 提出了基于三阶段（starting, course, ending）结构分析的池化结构Structured Temporal Pyramid Pooling（STPP）用于提取视频的特征，并进行视频的完整性预测
- 提出了一种新的视频提名框预测方法Temporal Actionness Grouping(TAG)，增强了提名框预测的效果
- 在视频检测比较著名的数据集THUMOS14和ActivityNet1.2上评估，获得了不错的效果

### 方法

网络主体结构如下图：

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181209200418.png)

先解释一下图中几个比较重要的元素：

- 绿框：TAG提出来的proposal，直接包括动作的头尾，长度为s
- 黄框：在绿框的基础上，向前向后各多取s/2长度的框，形成整个动作的context，黄框和绿框把整个区域分为三个stage：[starting, cause, ending]
- 中间CNN: 特征提取部分，使用一个2D网络（ResNet/Inception等）逐帧提取特征
- 平行四边形：对不同的stage使用不同层次的pooling，即Structured Temporal Pyramid Pooling，最后合成一整条特征
- 分为三个loss优化：
  - action类别分类，使用Cross Entropy
  - action完整性分类，使用OHMEHingeloss 
  - action框的精修，相当于faster-rcnn的bbox regression

### Temporal Actionness Grouping

SSN的第一步就是提取绿色的proposal，是如何做的呢，具体有以下几个大步骤：

1. 首先使用常用的滑动窗口方法，在原视频上滑动出一些原始proposal
2. 这些proposal进行二分类训练，判别该proposal与GT的IoU是否超过一定阈值
3. 训练完成以后，等长采样每个视频，输出每个小段的动作概率，形成一个时间维度上每个片段的可能性（actionness）
4. 使用TAG方法在3的基础上组合出有效的proposal

前面的几步都比较好理解，重点说一下是如何从actionness组合出有效的proposal的过程，这是文章的主要创新点之一：

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181209201940.png)

显现我们得到了每个小片段中含有动作的概率了，就是上图这样的，一个曲线图。作者把这个曲线图倒过来（中间红色曲线），高的地方作为“山脊”，山脊中间的地方作为“盆地”。向盆地中灌不同深度的水（图中不同程度的蓝色）其实就是取不同大小的阈值，这个阈值我们称为$\gamma$。然后对同一个阈值的盆地，进行合并，最后得到不同的proposal。那么如何合并呢？我们从一个种子盆地出发，不断的溶解他后面的盆地，把他加入到proposal中，直到生成的proposal中的盆地总长度除以proposal的总长度小于某个阈值$$\tau$,则停止溶解（当然最后一个盆地也不会被加进去），这样就形成了上图的橙色虚线框。

要注意的是几点：

- 阈值$\gamma$有多个，这样做时为了有不同长度的proposal，显然，阈值越低，连续的propsal越长
- 阈值$\tau$也有多个，这样做是为了给不同的盆地组合出更丰富的proposal
- $(\gamma, \tau)$都取遍[0, 1]的区间，溶解出来的框作为我们最上面的绿色框使用，向前向后各多采样半个propsal长度就是我们的黄色提名框了

### Structured Temporal Pyramid Pooling

为了能够评估proposal是否完整，SSN提出了三段动作结构分析，使用这个结构，我们proposal提出三个阶段的特征，并做完整性二分类。这个Structured Temporal Pyramid Pooling就是用来在不同的层次上提取不同阶段的特征而设计的。

具体举一个例子，在主要框架图中，我们知道黄框占8帧，绿色占中间的4帧，那么前2帧即为starting阶段，中间4帧是cause阶段，后2帧为ending阶段，在把每一帧都提取好特征以后，我们先使用一个“一层”池化，将starting，cause，ending部分的特征图都做average pooling，由于中间的cause部分是动作发生的主要部分，更加复杂，信息更丰富，因此使用“两层“池化来，两层池化将动作分为两段，每段分别做pooling，这样我们就得到了5段pooling以后的特征，把他们全都concat起来，作为后面分类的特征使用。

接下来就是训练三个header了，他们分别是动作分类器，完整性分类器和bounding box回归器，这三个header使用不同的数据样本来训练：

- 正例：和GT的IoU超过0.7的数据，这种样本完整性标签为1（action_label=1, 2, 3…, completeness_label=1）
- 负例：与任何GT的重叠度都很低（action_label=0, completeness_label不计算）
- 不完整：该proposal中80%的区域都落在GT里面，但是只有不到0.3的IoU（只是GT的一个片段）（action_label=1,2,3…, completeness_label=0）

动作分类器使用正例和负例训练，完整性分类器使用正例和不完整样本训练，bouding box回归使用正例训练。

## 实验

文中给出了两个权威数据集THUMOS14和ActivityNet v1.3的结果，THUMOS14数据集稍小，效果也更好一些，不过目前其他的文章(比如TAL-Net)在THUMOS14上mAP已经40+了，但是SSN总体来说来两个数据集上的效果比较均衡，不会出现严重偏科现象，比如TAL-Net在ActivityNet上效果较差。

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181209204609.png)

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181209204625.png)



## 总结

SSN作为一个很不错的视频检测框架，基本已经具备了视频检测的各大要素如Proposal提取，滑动窗口，多尺度pooling，动作分类，bounding box微调等，作为入门很适合一读，更重要的是SSN的代码开放，并且有不少人在讨论SSN，这也让SSN称为视频检测入门一个比较不错的框架，接下来可能会总结更多视频检测相关的文章，感兴趣的同学可以关注。