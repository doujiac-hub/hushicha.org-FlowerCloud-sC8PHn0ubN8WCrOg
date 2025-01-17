

> 来源：晓飞的算法工程笔记 公众号，转载请注明出处


**论文: A Simple Image Segmentation Framework via In\-Context Examples**


![](https://i-blog.csdnimg.cn/img_convert/4c23adfd4d3a01b403e43c89c7a8974d.png)


* **论文地址：[https://arxiv.org/abs/2410\.04842](https://github.com)**
* **论文代码：[https://github.com/aim\-uofa/SINE](https://github.com):[wgetcloud全球加速器服务](https://wgetcloud6.org)**


# 创新点




---


* 探索了通用的分割模型，发现现有方法在上下文分割中面临任务模糊性的问题，因为并非所有的上下文示例都能准确传达任务信息。
* 提出了一个利用上下文示例的简单图像分割框架`SINE`（`Segmentation framework via IN-context Examples`），利用了一个`Transformer`编码\-解码结构，其中编码器提供高质量的图像表示，解码器则被设计为生成多个任务特定的输出掩码，以有效消除任务模糊性。
* `SINE`引入了一个上下文交互模块，以补充上下文信息，并在目标图像与上下文示例之间产生关联，以及一个匹配`Transformer`，使用固定匹配和匈牙利算法消除不同任务之间的差异。
* 完善了当前的上下文图像分割评估系统，实验结果表明，`SINE`可以处理广泛的分割任务，包括少量样本的语义分割、少量样本的实例分割和视频目标分割。


# 内容概述




---


图像分割涉及在像素级别上定位和组织概念，比如语义分割、实例分割、全景分割、前景分割和交互分割。然而，现有的大多数分割方法都是针对特定任务量身定做的，无法应用于其他任务。


![](https://i-blog.csdnimg.cn/img_convert/8ae8fb275c94307cbbdd1a22a7e77d87.png)


最近一些工作探索了通用分割模型，通过上下文学习解决多样且无限的分割任务。上下文分割模型需要理解上下文示例传达的任务和内容信息，并在目标图像上分割相关概念，但并不是所有的上下文示例都能准确传达任务信息。例如当提供一个特定个体的照片，是仅限于个体本身、涵盖所有人的实例分割，还是集中于语义分割？模糊的上下文示例可能使传统的上下文分割模型难以清晰地定义不同任务之间的边界，从而导致不期望的输出。


为了解决这个问题，论文提出了基于上下文示例的简单图像分割框架`SINE`（`Segmentation framework via IN-context Examples`）。受到`SAM`模型的启发，`SINE`预测针对不同复杂度任务定制的多个输出掩码。这些任务包括相同物体、实例到整体语义概念。`SINE`统一了现有的各种粒度的分割任务，旨在实现更广泛的任务泛化。


与`SegGPT`相比，`SINE`能够在可训练参数更少的情况下有效地解决上下文分割中的任务模糊性问题，而`SegGPT`仅输出语义分割结果。此外，论文进一步将少样本实例分割引入当前的评估系统，以便全面评估这些模型。


# `SINE`




---


![](https://i-blog.csdnimg.cn/img_convert/9414fd3c5c1c18905c970f1092dc7e40.png)


`SINE`是一个基于查询的分割模型，遵循`DETR`和`Mask2Former`的设计。使用相同对象（`ID`）查询 \\(\\textbf{q}\_{id}\\) 来识别和定位目标图像中与参考图像中具有相同对应关系的对象，使用可学习的实例查询 \\(\\textbf{q}\_{ins} \\in \\mathbb{R}^{S \\times C}\\) 来识别和定位目标图像中与参考图像具有相同语义标签的对象。


`SINE`基于经典的`Transformer`结构，引入了一些针对上下文分割任务的有效设计，包括一个冻结的预训练图像编码器、一个上下文交互模块和一个轻量级匹配`Transformer` (`M-Former`) 解码器。


## 上下文交互


![](https://i-blog.csdnimg.cn/img_convert/ba54e5ff2353feac3fff5636eda2b868.png)


上下文交互的目的是补充上下文信息，并在参考图像特征和目标图像特征之间产生关联。


* ### 掩码池化


为每个掩码分配不同的`ID`标签，将参考掩码 \\(\\textbf{m}\_r\\) 转换为`ID`掩码 \\(\\textbf{m}\_{id} \\in \\mathbb{R}^{N \\times H \\times W}\\) ，通过将具有相同类别标签的掩码合并来得到语义掩码 \\(\\textbf{m}\_{sem} \\in \\mathbb{R}^{M \\times H \\times W}\\) ，其中 \\(N\\) 和 \\(M\\) 分别是`ID`掩码和语义掩码的数量。


然后，使用这些掩码对参考特征 \\(\\textbf{F}\_r\\) 进行池化，获得提`ID`标记 \\(\\textbf{t}\_{id} \\in \\mathbb{R}^{N \\times C}\\) 和语义标记 \\(\\textbf{t}\_{sem} \\in \\mathbb{R}^{M \\times C}\\) 。


* ### 上下文融合模块


上下文融合模块该模块是一个`Transformer`块，包括自注意力机制、交叉注意力机制和前馈网络，实现参考特征和目标特征之间的上下文关联：


\\\[\\begin{equation}
\\begin{split}
\\left\<\\textbf{q}\_{id}, \\textbf{p}\_{sem}, \\textbf{F}\_t^{'}\\right\> \= InContextFusion\\left(\\textbf{t}\_{id}, \\textbf{t}\_{sem}, \\textbf{F}\_{t} ;\\theta \\right),
\\end{split}
\\end{equation}
\\]这些标记 ( \\(\\textbf{t}\_{id}\\) 和 \\(\\textbf{t}\_{sem}\\) ) 和目标特征 ( \\(\\textbf{F}\_{t}\\) ) 通过这个共享模块进行融合，在交叉注意力中它们彼此作为键和值，从而可以获得增强后的目标特征 \\(\\textbf{F}\_t^{'}\\) 、`ID`查询 \\(\\textbf{q}\_{id}\\) 和语义原型 \\(\\textbf{p}\_{sem}\\) 。


## 匹配`Transformer`


为了有效地进行上下文分割并消除任务模糊性，`M-Former`实现了一个双路径的`Transformer`解码器，共享自注意力层。一路径用于与查询（ \\(\\textbf{q}\_{id}\\) 和 \\(\\textbf{q}\_{ins}\\) ）交互，提取与目标图像中的上下文示例相关的特征。第二路径用于增强语义原型 \\(\\textbf{p}\_{sem}\\) 以实现更准确的匹配。这两条路径共享自注意力层，以便将语义从 \\(\\textbf{p}\_{sem}\\) 分配给 \\(\\textbf{q}\_{ins}\\) 。


`M-Former`共有 N 个块，整体的过程如下：


\\\[\\begin{equation}
\\begin{split}
\\left\<\\textbf{q}\_{id}^l, \\textbf{q}\_{ins}^l, \\textbf{p}\_{sem}^l\\right\> \= MFormer\_l\\left(\\textbf{q}\_{id}^{l\-1}, \\textbf{q}\_{ins}^{l\-1}, \\textbf{p}\_{sem}^{l\-1} ; \\theta^l, \\textbf{F}\_t^{'} \\right),
\\end{split}
\\end{equation}
\\]对于实例分割，使用更新后的语义原型 \\(\\textbf{p}\_{sem}\\) 作为分类器，并让 \\(\\hat{\\textbf{y}}\_{ins}\=\\{\\hat{y}\_{ins}^i\\}\_{i\=1}^S\\) 表示 \\(S\\) 个实例预测的集合。使用匈牙利损失来学习`SINE`，通过计算预测 \\(\\hat{y}\_{ins}^i\\) 和`GT` \\(y^j\\) 之间的分配成本以解决匹配问题，即 \\(\-p\_i(c^j)\+\\mathcal{L}\_\\text{mask}(\\hat{m}\_{ins}^i,m^j)\\) ，其中 \\((c^j, m^j)\\) 是`GT`对象的类别和掩码， \\(c^j\\) 可能为 \\(\\varnothing\\) 。 \\(p\_i(c^j)\\) 是第 \\(i\\) 个实例查询对应类别 \\(c^j\\) 的概率， \\(\\hat{m}\_{ins}^i\\) 表示其预测的掩码。 \\(\\mathcal{L}\_\\text{mask}\\) 是一种二元掩码损失和`Dice`损失：


\\\[\\begin{equation}
\\begin{split}
\\mathcal{L}\_{\\text{Hungarian}}(\\hat{\\textbf{y}}\_{ins}, \\textbf{y}) \= \\sum\\nolimits\_{j\=1}^S \\left\[\-\\log p\_{\\sigma(j)}(c^j)
\+ \\mathbb{1}\_{c^j\\neq\\varnothing} \\mathcal{L}\_{\\text{mask}}(\\hat{m}^{\\sigma(j)}\_{ins},m^j) \\right],
\\end{split}
\\end{equation}
\\]其中 \\(\\sigma(j)\\) 表示二分匹配的结果索引。


为了赋予`SINE`预测同一对象的能力，使用图像中同一实例的不同裁剪视图作为参考\-目标图像对。设 \\(\\hat{\\textbf{y}}\_{id}\=\\{\\hat{y}\_{id}^i\\}\_{i\=1}^N\\) 表示 \\(N\\) 个`ID`预测的集合。


由于参考`ID`和目标`ID`之间的关系是固定的且可以准确确定，可以在预测和`GT`之间执行固定匹配，损失可以写为：


\\\[\\begin{equation}
\\begin{split}
\\mathcal{L}\_{\\text{ID}}(\\hat{\\textbf{y}}\_{id}, \\textbf{y}) \= \\sum\\nolimits\_{i\=1}^N \\left\[\-\\log p\_i(c^i)
\+ \\mathbb{1}\_{c^i\\neq\\varnothing} \\mathcal{L}\_{\\text{mask}}(\\hat{m}^i\_{id},m^i) \\right],
\\end{split}
\\end{equation}
\\]其中 \\((c^i, m^i)\\) 是`GT`的类别和掩码， \\(c^i \\in \\{1, \\varnothing\\}\\) ， \\(c^i\=1\\) 表示一个对象同时出现在参考图像和目标图像中。总损失为 \\(\\mathcal{L}\=\\mathcal{L}\_{\\text{Hungarian}}\+\\mathcal{L}\_{\\text{ID}}\\) 。


一旦训练完成，`SINE`的全部能力在推理过程中得以释放，能够解决上下文示例中的模糊性并为不同的分割任务输出预测。


# 主要实验




---


![](https://i-blog.csdnimg.cn/img_convert/884db99baf3c2f1da623ed70828a8810.png)


![](https://i-blog.csdnimg.cn/img_convert/3f4ace27c912d57609a8d1aceb317ce9.png)


 
 
 



> 如果本文对你有帮助，麻烦点个赞或在看呗～
> 更多内容请关注 微信公众号【晓飞的算法工程笔记】


![work-life balance.](https://i-blog.csdnimg.cn/img_convert/9c0cf685ed20b05b4802fc70cddeebef.webp?x-oss-process=image/format,png)


