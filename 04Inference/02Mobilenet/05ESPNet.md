<!--Copyright © XcodeHw 适用于[License](https://github.com/chenzomi12/AISystem)版权许可-->

# ESPNet系列

本章节会介绍ESPNet系列，与前面的网络不太一样的是该网络主要应用在高分辨率图像下的语义分割，在计算内存占用、功耗方面都非常高效，重点介绍一种高效的空间金字塔卷积模块(ESP Module)，而在ESPNetV2上则是会更进一步给大家呈现如何利用分组卷积核，深度空洞分离卷积学习巨大有效感受野，进一步降低浮点计算量和参数量。

## ESPNet V1

**ESPNetV1**：是一种快速高效的卷积神经网络，主要应用在高分辨图像下的语义分割，在计算、内存占用、功耗方面都非常高效。主要贡献在于基于传统卷积模块，提出一种高效空间金子塔卷积模块(ESP Module)，有助于减少模型运算量和内存、功率消耗 ，来提升终端设备适用性，方便部署到移动端。

### 设计思路

#### Efficient spatial pyramid

ESPNet基于卷积因子分解的原则，ESP模块将标准卷积分解成point-wise卷积和空洞卷积金字塔(spatial pyramid of dilated convolutions)，point-wise卷积将输入映射到低维特征空间，就是采用K个1x1xM的小卷积核对原图进行卷积操作，1x1卷积的作用其实就是为了降低维度，这样就可以减少参数。空洞卷积金字塔使用K组空洞卷积的同时下采样得到低维特征这种分解方法能够大量减少ESP模块的参数和内存，并且保证了较大的感受野(如下图a所示)。

![ESP结构](./images/05.espnet_01.png)

下面来计算下一共包含的参数，其实在效果上，以这种轻量级的网络作为backbone效果肯定不如那些重量级的，比如Resnet，但是在运行速度上有很大优势。

如上图所示，对Efficient spatial pyramid第一部分来说，$d$个$1\times1\times M$的卷积核，将M维的输入feature map降至d维。此时参数为:$M*N/K$ ,第二部分参数量为$K*n^{2}*(N/K)^{2}$，和标准卷积结构相比，参数数量降低很多。

为了减少计算量，又引入了一个简单的超参数K,它的作用是统一收缩网络中各个ESP模块的特征映射维数。Reduce对于给定K,ESP模块首先通过逐点卷积将特征映射从m维空间缩减到$N/K$维空间(上图a中的步骤1)。split然后将低维特征映射拆分到K个并行分支上。Transform:然后每个分支使用$2^{k-1},k=1,...,k-1$给出的$n\times\n$个扩张速率不同的卷积核同时处理这些特征映射(上图a中的步骤2)。merge:然后将这K个并行扩展卷积核的输出连接起来，产生一个n维输出特征map。上图b展示了ESP模块采用的减少-分裂-转换-合并策略。

```python
class ESPModule(nn.Module):
    def __init__(self, in_channels, out_channels, K=5, ks=3, stride=1,act_type='prelu',):
        super(ESPModule, self).__init__()
        self.K = K
        self.stride = stride
        self.use_skip = (in_channels == out_channels) and (stride == 1)
        channel_kn = out_channels // K
        channel_k1 = out_channels - (K -1) * channel_kn
        self.perfect_divisor = channel_k1 == channel_kn

        if self.perfect_divisor:
            self.conv_kn = conv1x1(in_channels, channel_kn, stride)
        else:
            self.conv_kn = conv1x1(in_channels, channel_kn, stride)
            self.conv_k1 = conv1x1(in_channels, channel_k1, stride)

        self.layers = nn.ModuleList()
        for k in range(1, K+1):
            dt = 2**(k-1)       # dilation
            channel = channel_k1 if k == 1 else channel_kn
            self.layers.append(ConvBNAct(channel, channel, ks, 1, dt, act_type=act_type))

    def forward(self, x):
        if self.use_skip:
            residual = x

        transform_feats = []
        if self.perfect_divisor:
            x = self.conv_kn(x)     # Reduce
            for i in range(self.K):
                transform_feats.append(self.layers[i](x))   # Split --> Transform
                
            for j in range(1, self.K):
                transform_feats[j] += transform_feats[j-1]      # Merge: Sum
        else:
            x1 = self.conv_k1(x)    # Reduce
            xn = self.conv_kn(x)    # Reduce
            transform_feats.append(self.layers[0](x1))      # Split --> Transform
            for i in range(1, self.K):
                transform_feats.append(self.layers[i](xn))   # Split --> Transform

            for j in range(2, self.K):
                transform_feats[j] += transform_feats[j-1]      # Merge: Sum

        x = torch.cat(transform_feats, dim=1)               # Merge: Concat

        if self.use_skip:
            x += residual

        return x
```



#### Hierarchical feature fusion (HFF)

虽然将扩张卷积的输出拼接在一起会给ESP模块带来一个较大的有效感受野，但也会引入不必要的棋盘或网格假象，如下图所示。

![HFF结构](./images/05.espnet_02.png)

上图(a)举例说明一个网格伪像，其中单个活动像素(红色)与膨胀率r = 2的3×3膨胀卷积核卷积。

上图(b)具有和不具有层次特征融合(HFF)的ESP模块特征图可视化。ESP中的HFF消除了网格伪影。彩色观看效果最佳。

为了解决ESP中的网格问题，使用不同膨胀率的核获得的特征映射在拼接之前会进行层次化添加(上图b中的HFF)。该解决方案简单有效，且不会增加ESP模块的复杂性，这与现有方法不同，现有方法通过使用膨胀率较小的卷积核学习更多参数来消除网格误差[Dilated residual networks,Understanding convolution for semantic segmentation]。为了改善网络内部的梯度流动，ESP模块的输入和输出特征映射使用元素求和[Deep residual learning for image recognition]进行组合。

### 网络结构

ESPNet使用ESP模块学习卷积核以及下采样操作，除了第一层是标准的大步卷积。所有层(卷积和ESP模块)后面都有一个批归一化和一个PReLU非线性，除了最后一个点卷积，它既没有批归一化，也没有非线性。最后一层输入softmax进行像素级分类。

ESPNet的不同变体如下图所示。第一个变体，ESPNet-A(图a)，是一种标准网络，它以RGB图像作为输入，并使用ESP模块学习不同空间层次的表示，以产生一个分割掩码。第二种ESP - b(图b)通过在之前的跨步ESP模块和之前的ESP模块之间共享特征映射，改善了ESPNet-A内部的信息流。第三种变体，ESPNet-C(图c)，加强了ESPNet-B内部的输入图像，以进一步改善信息的流动。这三种变量产生的输出的空间维度是输入图像的1 / 8。第四种变体，ESPNet(图d)，在ESPNet- c中添加了一个轻量级解码器(使用reduceupsample-merge的原理构建)，输出与输入图像相同空间分辨率的分割mask。

![ESP网络结构](./images/05.espnet_03.png)

从ESPNet- a到ESPNet的路径。红色和绿色色框分别代表负责下采样和上采样操作的模块。空间级别的l在(a)中的每个模块的左侧。本文将每个模块表示为(#输入通道，#输出通道)。这里，conv-n表示n × n卷积。

为了在不改变网络拓扑结构的情况下构建具有较深计算效率的边缘设备网络，超参数α控制网络的深度;ESP模块在空间层次l上重复$α_{l}$次。在更高的空间层次(l = 0和l = 1)， cnn需要更多的内存，因为这些层次的特征图的空间维数较高。为了节省内存，ESP和卷积模块都不会在这些空间级别上重复。

## ESPNet V2

**ESPNetV2**：是由ESPNetV1改进来的，一种轻量级、能耗高效、通用的卷积神经网络，利用分组卷积核深度空洞分离卷积学习巨大有效感受野，进一步降低浮点计算量和参数量。同时在图像分类、目标检测、语义分割等任务上检验了模型效果。

### 设计思路

与V1版本相比，其特点如下：

1.将原来ESPNet的point-wise convolutions替换为group point-wise convolutions；

2.将原来ESPNet的dilated convolutions替换为depth-wise dilated convolution；

3.HFF加在depth-wise dilated separable convolutions和point-wise (or 1 × 1)卷积之间，去除gridding artifacts；

4.使用group point-wise convolution 替换K个point-wise convolutions；

5.加入平均池化（average pooling ）,将输入图片信息加入EESP中；

6.使用级联（concatenation）取代对应元素加法操作（element-wise addition operation ）

####  Depth-wise dilated separable convolution

深度分离空洞卷积分两步：

- 对每个输入通道执行空洞率为r的DDConv，从有效感受野学习代表性特征。

- 标准1x1卷积学习DDConv输出的线性组合特征。

深度分离空洞卷积与其他卷积的参数量与感受野对比如下表所示。

| Convolution type             | Parameters | Eff.receptive field |
| ---------------------------- | ---------- | ------------------- |
| Standard                     |$n^{2}c\hat{c}$|$n\times n$       |
| Group                        |$\frac{n^{2}c\hat{c}}{g}$|$n\times n$|
| Depth-wise separable         |$n^{2}c+c\hat{c}$|$n\times n$|
| Depth-wise dilated separable |$n^{2}c+c\hat{c}$|$n_{r}\times n_{r}$|



#### EESP unit

EESP模块结构如下图，图b中相比于ESPNet，输入层采用分组卷积，DDConv+Conv1x1取代标准空洞卷积，依然采用HFF的融合方式，（c）是（b）的等价模式。当输入通道数M=240，g=K=4, d=M/K=60，EESP比ESP少7倍的参数。

![EESP结构](./images/03cnn/03CNN_05.png)

描述了一个新的网络单元EESP，它利用深度可分离扩张和组逐点卷积设计，专为边缘设备而设计。该单元受ESPNet架构的启发，基于ESP模块构建，使用了减少-分割-变换-合并的策略。通过组逐点和深度可分离扩张卷积，该单元的计算复杂度得到了显著的降低。进一步，描述了一种带有捷径连接到输入图像的分层EESP单元，以更有效地学习多尺度的表示。

如上图中b所示，能够降低$\frac{Md+n^{2}d^{2}K}{\frac{Md}{g}+(n^{2}+d)dK}$倍计算复杂度，K为空洞卷积金字塔层数。考虑到单独计算K个point-wise卷积等同于单个分组数为K的point-wise分组卷积，而分组卷积的在实现上更高效，于是改进为上图c的最终结构。

```python
class EESP(nn.Module):
    '''
    This class defines the EESP block, which is based on the following principle
        REDUCE ---> SPLIT ---> TRANSFORM --> MERGE
    '''

    def __init__(self, nIn, nOut, stride=1, k=4, r_lim=7, down_method='esp'):                     #down_method --> ['avg' or 'esp']
        '''
        :param nIn: number of input channels
        :param nOut: number of output channels
        :param stride: factor by which we should skip (useful for down-sampling). If 2, then down-samples the feature map by 2
        :param k: # of parallel branches
        :param r_lim: A maximum value of receptive field allowed for EESP block
        :param g: number of groups to be used in the feature map reduction step.
        '''
        super().__init__()
        self.stride = stride
        n = int(nOut / k)
        n1 = nOut - (k - 1) * n
        assert down_method in ['avg', 'esp'], 'One of these is suppported (avg or esp)'
        assert n == n1, "n(={}) and n1(={}) should be equal for Depth-wise Convolution ".format(n, n1)

        self.proj_1x1 = CBR(nIn, n, 1, stride=1, groups=k)

        # (For convenience) Mapping between dilation rate and receptive field for a 3x3 kernel
        map_receptive_ksize = {3: 1, 5: 2, 7: 3, 9: 4, 11: 5, 13: 6, 15: 7, 17: 8}
        self.k_sizes = list()
        for i in range(k):
            ksize = int(3 + 2 * i)
            # After reaching the receptive field limit, fall back to the base kernel size of 3 with a dilation rate of 1
            ksize = ksize if ksize <= r_lim else 3
            self.k_sizes.append(ksize)
        # sort (in ascending order) these kernel sizes based on their receptive field
        # This enables us to ignore the kernels (3x3 in our case) with the same effective receptive field in hierarchical
        # feature fusion because kernels with 3x3 receptive fields does not have gridding artifact.
        self.k_sizes.sort()
        self.spp_dw = nn.ModuleList()
     
        for i in range(k):
            d_rate = map_receptive_ksize[self.k_sizes[i]]
            self.spp_dw.append(CDilated(n, n, kSize=3, stride=stride, groups=n, d=d_rate))
            
        self.conv_1x1_exp = CB(nOut, nOut, 1, 1, groups=k)
        self.br_after_cat = BR(nOut)
        self.module_act = nn.PReLU(nOut)
        self.downAvg = True if down_method == 'avg' else False

    def forward(self, input):
        '''
        :param input: input feature map
        :return: transformed feature map
        '''

        # Reduce --> project high-dimensional feature maps to low-dimensional space
        output1 = self.proj_1x1(input)
        output = [self.spp_dw[0](output1)]
        # compute the output for each branch and hierarchically fuse them
        # i.e. Split --> Transform --> HFF
        for k in range(1, len(self.spp_dw)):
            out_k = self.spp_dw[k](output1)
            # HFF
            # We donot combine the branches that have the same effective receptive (3x3 in our case)
            # because there are no holes in those kernels.
            out_k = out_k + output[k - 1]
            #apply batch norm after fusion and then append to the list
            output.append(out_k)
        # Merge
        expanded = self.conv_1x1_exp( # Aggregate the feature maps using point-wise convolution
            self.br_after_cat( # apply batch normalization followed by activation function (PRelu in this case)
                torch.cat(output, 1) # concatenate the output of different branches
            )
        )
        del output
        # if down-sampling, then return the concatenated vector
        # as Downsampling function will combine it with avg. pooled feature map and then threshold it
        if self.stride == 2 and self.downAvg:
            return expanded

        # if dimensions of input and concatenated vector are the same, add them (RESIDUAL LINK)
        if expanded.size() == input.size():
            expanded = expanded + input

        # Threshold the feature map using activation function (PReLU in this case)
        return self.module_act(expanded)
```

#### Strided EESP

为了在多尺度下能够有效地学习特征，对上图1c的网络做了四点改动（如下图所示）：

1.对DDConv添加stride属性。

2.右边的shortcut中带了平均池化操作，实现维度匹配。

3.将相加的特征融合方式替换为concat形式，增加特征的维度。

4.融合原始输入图像的下采样信息，使得特征信息更加丰富。具体做法是先将图像下采样到与特征图的尺寸相同的尺寸，然后使用第一个卷积，一个标准的3×3卷积，用于学习空间表示。再使用第二个卷积，一个逐点卷积，用于学习输入之间的线性组合，并将其投影到高维空间。

![Strided EESP结构](./images/05.espnet_05.png)

```python

class DownSampler(nn.Module):
    '''
    Down-sampling fucntion that has two parallel branches: (1) avg pooling
    and (2) EESP block with stride of 2. The output feature maps of these branches
    are then concatenated and thresholded using an activation function (PReLU in our
    case) to produce the final output.
    '''

    def __init__(self, nin, nout, k=4, r_lim=9, reinf=True):
        '''
            :param nin: number of input channels
            :param nout: number of output channels
            :param k: # of parallel branches
            :param r_lim: A maximum value of receptive field allowed for EESP block
            :param g: number of groups to be used in the feature map reduction step.
        '''
        super().__init__()
        nout_new = nout - nin
        self.eesp = EESP(nin, nout_new, stride=2, k=k, r_lim=r_lim, down_method='avg')
        self.avg = nn.AvgPool2d(kernel_size=3, padding=1, stride=2)
        if reinf:
            self.inp_reinf = nn.Sequential(
                CBR(config_inp_reinf, config_inp_reinf, 3, 1),
                CB(config_inp_reinf, nout, 1, 1)
            )
        self.act =  nn.PReLU(nout)

    def forward(self, input, input2=None):
        '''
        :param input: input feature map
        :return: feature map down-sampled by a factor of 2
        '''
        avg_out = self.avg(input)
        eesp_out = self.eesp(input)
        output = torch.cat([avg_out, eesp_out], 1)
        if input2 is not None:
            #assuming the input is a square image
            w1 = avg_out.size(2)
            while True:
                input2 = F.avg_pool2d(input2, kernel_size=3, padding=1, stride=2)
                w2 = input2.size(2)
                if w2 == w1:
                    break
            output = output + self.inp_reinf(input2)

        return self.act(output) #self.act(output)
```
### 网络结构

ESPNetv2网络使用EESP单元构建。在每个空间级别，ESPNetv2重复多次EESP单元以增加网络的深度。其中在每个卷积层之后使用batch normalization和PRelu，但在最后一个组级卷积层除外，在该层中，PRelu是在element-wise sum操作之后应用的。

## 小结

ESPNet系列的核心在于空洞卷积金字塔，每层具有不同的dilation rate，在参数量不增加的情况下，能够融合多尺度特征，相对于深度可分离卷积，深度可分离空洞卷积金字塔性价比更高。另外，HFF的多尺度特征融合方法也很值得借鉴。

## 本节视频

<iframe src="https://player.bilibili.com/player.html?bvid=BV1DK411k7qt&as_wide=1&high_quality=1&danmaku=0&t=30&autoplay=0" width="100%" height="500" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

## 参考文献

1.[Zhao, H., Shi, J., Qi, X., Wang, X., Jia, J.: Pyramid scene parsing network. In: CVPR. (2017)](https://arxiv.org/pdf/1612.01105.pdf)

2.[He, K., Zhang, X., Ren, S., Sun, J.: Spatial pyramid pooling in deep convolutional networks for visual recognition. In: ECCV. (2014)]( https://arxiv.org/abs/1406.4729)

3.[Ess, A., Muller, T., Grabner, H., Van Gool, L.J.: Segmentation-based urban traffic scene understanding. In: BMVC. (2009)](https://www.bmva.org/bmvc/2009/Papers/Paper350/Abstract350.pdf)

4.[Menze, M., Geiger, A.: Object scene flow for autonomous vehicles. In: CVPR. (2015)](https://ieeexplore.ieee.org/document/7298925/)

5.[Xiang, Y., Fox, D.: DA-RNN: Semantic mapping with data associated recurrent neural networks. Robotics: Science and Systems (RSS) (2017)](https://arxiv.org/abs/1703.03098v1)

6.[Chollet, F.: Xception: Deep learning with depthwise separable convolutions. CVPR (2017)](https://arxiv.org/abs/1610.02357)

7.[Yu, F., Koltun, V.: Multi-scale context aggregation by dilated convolutions. ICLR (2016)](https://arxiv.org/pdf/1511.07122.pdf)

8.[Yu, F., Koltun, V., Funkhouser, T.: Dilated residual networks. CVPR (2017)](https://arxiv.org/pdf/1705.09914.pdf )

9.[Zhao, H., Qi, X., Shen, X., Shi, J., Jia, J.: Icnet for real-time semantic segmentation on high-resolution images. arXiv preprint arXiv:1704.08545 (2017)](https://arxiv.org/abs/1704.08545)

10.[Dai, J., He, K., Sun, J.: Convolutional feature masking for joint object and stuff segmentation.In: CVPR. (2015)](https://arxiv.org/abs/1412.1283v2)

11.[Tao Lei, Yu Zhang, and Yoav Artzi. Training rnns as fast as cnns. In EMNLP, 2018. 8](https://arxiv.org/pdf/1709.02755)

12.[Ilya Loshchilov and Frank Hutter. Sgdr: Stochastic gradient descent with warm restarts. In ICLR, 2017. 5](https://arxiv.org/abs/1608.03983)

13.[Bharath Hariharan, Pablo Arbelaez, Lubomir Bourdev, Subhransu Maji, and Jitendra Malik. Semantic contours from inverse detectors. In ICCV, 2011. 6](https://www.mendeley.com/catalogue/7ff08f6f-384f-3129-9f54-b7327bdd4276/)

14.[Barret Zoph and Quoc V Le. Neural architecture search with reinforcement learning. arXiv preprint arXiv:1611.01578,2016.2](https://ieeexplore.ieee.org/document/6126343)

15.[M. Siam, M. Gamal, M. Abdel-Razek, S. Yogamani, and M.Jagersand. rtseg: Real-time semantic segmentation comparative study. In 2018 25th IEEE International Conference on Image Processing (ICIP).7](https://arxiv.org/pdf/1803.02758v4)
