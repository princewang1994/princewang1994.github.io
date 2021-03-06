## Soft(er)-NMS：非极大值抑制算法的两个改进算法



[TOC]

### 前言

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201161114.png)

论文连接：

* [Soft-NMS -- Improving Object Detection With One Line of Code](https://arxiv.org/abs/1704.04503)
* [Softer-NMS: Rethinking Bounding Box Regression for Accurate Object Detection](https://arxiv.org/abs/1809.08545)



之前用了一篇[博客](http://blog.prince2015.club/2018/07/23/NMS/)详细说明了NMS的原理，Cython版加速实现和CUDA加速版实现，然而NMS还是存在一些问题，本篇博客介绍两种NMS的改进——Soft-NMS和Softer-NMS，提高NMS的性能。

### 传统NMS存在的问题

NMS是为了去除重复的预测框而设计的算法，首先我们回忆一下普通NMS的过程（具体的请参考[这篇博客](http://blog.prince2015.club/2018/07/23/NMS/)）：

假设所有输入预测框的集合为S，算法返回结果集合D初始化为空集$\phi$，具体算法如下：

1. 每次选择S中最高分框M，并把M从S中删除，加入D
2. 遍历S中剩余的框，每个框用bi表示，如果和B的IoU(M, bi)大于阈值t，将bi其从S中删除
3. 从S未处理的框中继续选一个得分最高的加入到结果D中，重复上述过程1, 2。直至S集合空，此时D即为所求结果。

那么NMS存在什么问题呢，其中最主要的问题有这么几个：

- 物体重叠：如下面第一张图，会有一个最高分数的框，如果使用nms的话就会把其他置信度稍低，但是表示另一个物体的预测框删掉（由于和最高置信度的框overlap过大）
- 存在一些，所有的bbox都预测不准，对于下面第二张图我们看到，不是所有的框都那么精准，有时甚至会出现某个物体周围的所有框都标出来了，但是都不准的情况

- 传统的NMS方法是基于分类分数的，只有最高分数的预测框能留下来，但是大多数情况下IoU和分类分数不是强相关，很多分类标签置信度高的框都位置都不是很准

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201140630.png)

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201140657.png)

![image-20181201140711790](/Users/prince/Library/Application Support/typora-user-images/image-20181201140711790.png)

### Soft-NMS

发现了这些问题，让我们想想如何避免：

首先说第一个问题：物体重叠是由于输出多个框中存在某些其实是另一个物体，但是也不小心被NMS去掉了。这个问题的解法最终是要落在“将某个候选框删掉”这一步骤上，我们需要找到一种方法，更小心的删掉S中的框，而不是暴力的把所有和最高分框删掉，于是有了这个想法：

对于与最高分框overlap大于阈值t的框M，我们不把他直接去掉，而是将他的置信度降低，这样的方法可以使多一些框被保留下来，从而一定程度上避免overlap的情况出现。

那么读者可能会问，如果只是把置信度降低，那么可能原来周围的多个表示同一个物体的框也没办法去掉了。Soft-NMS这样来解决这个问题，同一个物体周围的框有很多，每次选择分数最高的框，抑制其周围的框，与分数最高的框的IoU越大，抑制的程度越大，一般来说，表示同一个物体的框（比如都是前面的马）的IoU是会比另一个物体的框（比如后面的马）的IoU大，因此，这样就会将其他物体的框保留下来，而同一个物体的框被去掉。

接下来就是Soft NMS的完整算法：

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201143935.png)

红色的部分表示原始NMS算法，绿色部分表示Soft-NMS算法，区别在于，绿色的框只是把si降低了，而不是把bi直接去掉，极端情况下，如果f只返回0，那么等同于普通的NMS。

接下来说一下f函数：f函数是为了降低目标框的置信度，满足条件，如果bi和M的IoU越大，f(iou(M, bi))就应该越小，Soft-NMS提出了两种f函数：

线性函数：

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201144442.png)

这个比较好理解，当iou条件满足时，si乘上一个1-iou，线性变小
$$
s_i = s_i e^{-\frac{iou(\mathcal{M, b_i})^2}{\sigma}}, \forall b_i \notin \mathcal{D}
$$

高斯函数惩罚，越接近高斯分布中心，惩罚力度越大

#### 分析

Soft-NMS的效果也比较明显：重叠的物体被更大程度的保留下来，以下是效果图

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201144831.png)

不过Soft-NMS还是需要调整阈值的，由于没有“去掉”某些框的操作，因此，最后所有框都会被加入D中，只是那些被去掉的框的置信度明显降低，所以需要一个阈值（通常很小）来过滤D中的输出框，从而输出最后的预测框。所以Soft-NMS还是需要一些调参的



这里放一个注释版的Cython实现：

```python
def cpu_soft_nms(np.ndarray[float, ndim=2] boxes, float sigma=0.5, float Nt=0.3, float threshold=0.001, unsigned int method=0):
    cdef unsigned int N = boxes.shape[0]
    cdef float iw, ih, box_area
    cdef float ua
    cdef int pos = 0
    cdef float maxscore = 0
    cdef int maxpos = 0
    cdef float x1,x2,y1,y2,tx1,tx2,ty1,ty2,ts,area,weight,ov

    for i in range(N):
        
        # 在i之后找到confidence最高的框，标记为max_pos
        maxscore = boxes[i, 4]
        maxpos = i

        tx1 = boxes[i,0]
        ty1 = boxes[i,1]
        tx2 = boxes[i,2]
        ty2 = boxes[i,3]
        ts = boxes[i,4]

        pos = i + 1
	    # 找到max的框
        while pos < N:
            if maxscore < boxes[pos, 4]:
                maxscore = boxes[pos, 4]
                maxpos = pos
            pos = pos + 1
        
        # 交换max_pos位置和i位置的数据
	    # add max box as a detection 
        boxes[i,0] = boxes[maxpos,0]
        boxes[i,1] = boxes[maxpos,1]
        boxes[i,2] = boxes[maxpos,2]
        boxes[i,3] = boxes[maxpos,3]
        boxes[i,4] = boxes[maxpos,4]

	    # swap ith box with position of max box
        boxes[maxpos,0] = tx1
        boxes[maxpos,1] = ty1
        boxes[maxpos,2] = tx2
        boxes[maxpos,3] = ty2
        boxes[maxpos,4] = ts

        tx1 = boxes[i,0]
        ty1 = boxes[i,1]
        tx2 = boxes[i,2]
        ty2 = boxes[i,3]
        ts = boxes[i,4]
        # 交换完毕
        
        # 开始循环
        pos = i + 1
        
        while pos < N:
            # 先记录内层循环的数据bi
            x1 = boxes[pos, 0]
            y1 = boxes[pos, 1]
            x2 = boxes[pos, 2]
            y2 = boxes[pos, 3]
            s = boxes[pos, 4]
            
            # 计算iou
            area = (x2 - x1 + 1) * (y2 - y1 + 1)
            iw = (min(tx2, x2) - max(tx1, x1) + 1) # 计算两个框交叉矩形的宽度，如果宽度小于等于0，即没有相交，因此不需要判断
            if iw > 0:
                ih = (min(ty2, y2) - max(ty1, y1) + 1) # 同理
                if ih > 0:
                    ua = float((tx2 - tx1 + 1) * (ty2 - ty1 + 1) + area - iw * ih) #计算union面积
                    ov = iw * ih / ua #iou between max box and detection box

                    if method == 1: # linear
                        if ov > Nt: 
                            weight = 1 - ov
                        else:
                            weight = 1
                    elif method == 2: # gaussian
                        weight = np.exp(-(ov * ov)/sigma)
                    else: # original NMS
                        if ov > Nt: 
                            weight = 0
                        else:
                            weight = 1

                    boxes[pos, 4] = weight*boxes[pos, 4]
		    
		            # if box score falls below threshold, discard the box by swapping with last box
		            # update N
                    if boxes[pos, 4] < threshold:
                        boxes[pos,0] = boxes[N-1, 0]
                        boxes[pos,1] = boxes[N-1, 1]
                        boxes[pos,2] = boxes[N-1, 2]
                        boxes[pos,3] = boxes[N-1, 3]
                        boxes[pos,4] = boxes[N-1, 4]
                        N = N - 1
                        pos = pos - 1

            pos = pos + 1

    keep = [i for i in range(N)]
    return keep
```



### Softer-NMS

Soft-NMS只解决了三个问题中的第一个问题，那么剩下两个问题如何解决呢？

我们先分析第三个问题，对于分类置信度和框的IoU不是强相关的问题，我们需要找到一种方法来衡量框的“位置置信度”，对于第二个问题我们也有办法：只需要让多个框加权合并生成最终的框就可以了，本文提出这两个想法：

1. 构建一种IoU的置信度，来建模有多大把握认为当前框和GT是重合的
2. 提出一种方法根据IoU置信度加权合并多个框优化最终生成框

 

#### IoU置信度

Softer-NMS文章对预测框建模，以下公式中$x$表示偏移前的预测框，$x_e$表示偏移后的预测框，输出的$x_g$表示GT框，使用高斯函数对预测框建模：
$$
P_\Theta(x) = \frac1{2\pi\sigma^2}e^{-\frac{(x-x_e)^2}{2\sigma^2}}
$$


GT框建模：使用delta分布：
$$
P_D(x) = \delta(x - x_g)
$$


上个式子里面的delta分布是当高斯分布的$\sigma$趋于0时的极限分布，大概就是这样：

![Dirac_function_approximation](media/Dirac_function_approximation.gif)

当$\sigma$越小，曲线越瘦高，当$\sigma$非常小的时候，几乎是一条竖直的线，同时$\sigma$越小，表示越确定，因此$1 - \sigma$可以作为置信度的，网络结构图如下，上面是原始fasterRCNN的结构，下面是softer-nms结构，多加了一个sigma预测，也就是box std，而Box的预测其实就是上面公式中的$x_e$

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201151510.png)

计算过程：

- 通过$x_e$，$x$的2范数距离和$\sigma$算出$P_\Theta(x)$
- 通过$x_g$和$x$的2范数距离算出$P_D$
- 使用$P_D$和$P_\Theta$计算KLs散度作为loss，最小化KLLoss

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201151425.png)

上面是$P_\Theta$和$P_D$的说明图，蓝色部分是标准差为$\sigma$的高斯分布，橙色是当$\sigma$为0时的delta分布，高斯分布的中心表示最终预测框的位置，$\sigma$表示置信度

#### 分析

我们分析一下，Softer-NMS使用了一个技巧：当高斯函数的$\sigma$区域0的时候是非常像delta分布的，因此在训练的过程中，通过KL散度优化两者的距离，其实是告诉网络，调整置信度$\sigma$和$x_e$，使网络输出的分布尽量输出高置信度，而且接近GT的偏移框；

接下来做一些数学分析：

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201152145.png)

以上是KL散度的计算，后面两个和预测出来的$x_e$是无关的（在这一步x已经确定），因此后面两项可以去掉；

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201152237.png)

继续看，求梯度发现梯度中$\sigma$在分母:

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201152303.png)

容易导致梯度爆炸，而且置信度“越高越好”才比较符合常理，所以这里把$1/\sigma$替换为了$\alpha=1/\sigma^2$

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201152333.png)

并加上一个极小量来防止分母为0，这样梯度就不会为0了，在$x_g$和$x_e$偏移太大的时候，文章也是用了和smoothl1算法类似方法，把loss由2次降为1次，来防止离群点对模型影响过大。

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201152353.png)

首先，网络预测出的东西和普通的faster_RCNN有所不同：$\{x1_i , y1_i , x2_i , y2_i , s_i , \sigma_{x1,i} , \sigma_{y1,i} , \sigma_{x2,i} , \sigma_{y2,i }\}$，后面几个是四个坐标的$\sigma$

上表中的蓝色的是soft-nms，只是降低了S的权值，重点看绿色的，绿字第一行表示拿出所有与B的IoU大于Nt的框（用idx表示），然后将所有这些框做一个加权，B[idx]/C[idx]其实是B[idx] * 1/C[idx]，后者是置信度$1/\sigma^2$，并使用sum做了归一化。需要注意的是，Softer-NMS算法中，B是不变的，softer-nms只调整每个框的位置，而不筛选框；(还没来得及阅读源码，应该在这之后又进行了框的筛选，不过这个时候已经通过softer-nms使那些分类置信度较高的框有更好的localtion输出了，所以效果应该会好很多)

下图表示Softer-NMS将右边woman的一个不准确的框移动为相对准确的框，蓝色数字表示置信度$\sigma$

![](http://princepicbed.oss-cn-beijing.aliyuncs.com/blog_20181201155716.png)

### 总结

本文总结了Soft-NMS和Softer-NMS两篇文章，这两篇文章都是基于传统NMS上的改进，目的在于解决目标检测中的物体重叠难以预测，分类分数与IoU不匹配的问题，可以尤其是Soft-NMS，能够在不重新训练模型的前提下，只通过修改少量的代码就能带来不少的提升，可以说是非常易用了。Softer-NMS的实现稍显复杂，需要网络重新训练以预测出四个坐标（x1, x2, y1, y2）的坐标置信度，然后用此置信度作为权值加权。实现了”多框合一“，其中使用的高斯分布和delta分布建模的思想很有意思，给置信度的计算多了一些新的思路。