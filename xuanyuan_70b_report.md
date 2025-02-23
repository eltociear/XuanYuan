
# 概览：

XuanYuan-70B 金融大模型是基于Llama2-70b模型进行增量预训练得到的一个基座模型，预期在中文能力和金融能力方面，相比原始模型得到较大提升，技术优化点包括：

（1）**数据质量**

- 我们设计了一套数据清洗流水线，精心准备了各类通用数据（互联网网页、百科、论坛、社交媒体、问答等）以及金融相关数据（金融资讯、公司公告、金融百科、金融书籍、证书试题等）高质量数据
- 中英数据：首先llama2的英文能力足够优秀，所以为了保证英文能力不降，我们扩充词表之后，使用高质量的中英语料进行增量预训练，其中中英配比为3:1；
- 通用金融数据：为了提升模型在金融能力上效果，预训练过程中通用语料与金融预料比例为9:1，且随着训练进行，逐步提升金融语料的占比。

（2）**模型训练**

- 训练效率：我们采取了一系列的加速优化策略， 包括对底层数据加载和分布式训练框架的多处优化，使用flash attention2替代self-attention模块，使用基于CPP CUDA的Fused算子替代原始llama的python实现等
- 上下文长度：基于上述的优化方式，同时考虑到金融场景长上下文情景较多，我们能够在预训练阶段把llama2原始上下文4k的长度扩展到8k和16k；

我们在100台8卡A800(80G)的GPU集群中，训练情况如下：

| 模型         | 上下文长度 | 吞吐量           | 显卡利用 |
| ------------ | ---------- | ---------------- | -------- |
| XuanYuan-70B | 8192       | 340 tokens/s/gpu | 190TFOPS |

备注：（1）训练没有开梯度累计；（2）原始llama2-70b在4k上下文长度下的的吞吐量为323 tokens/s/gpu，说明我们的训练效率达到当前领先水平。

# 数据质量

数据质量是影响大模型的训练效果的最关键因素，我们针对各类数据设计了一套通用的数据清洗流水线，主要包含如下的数据预处理：

- 文本抽取：原始数据中存在大量的互联网页面 和 PDF数据（如研报、公告、书籍等），我们针对这两类源数据做了详细的内容抽取工作，HTML标签移除、PDF内容深度解析等技术，特殊格式如表格使用markdown存储，数学公式使用latex等，保证文本内容尽可能全而准的抽取出来。
- 数据清洗：基于抽取的文本内容，我们设计了更深层次的数据清洗模块，包括词级别、句子级别、文档级别的数据清洗，根据不同数据集的分布特点，制定几十类不同的清洗规则和过滤阈值，这一步实现了语法、词法、基本语义的清洗。
- 数据去重：这一步我们使用MinHashLSH算法，采取单类别的局部去重+全类别的全局去重两阶段策略，来去除重复数据。
- 质量筛选：为了进一步过滤有毒、有害的信息，我们通过人工标注+关键词抽样的构造方式，形成了一批有害样本，涵盖暴力、色情、歧视、反动、博彩、价值观错误等多种有害信息，基于上述样本，我们训练了一个文档质量分类模型，来对语料进行质量过滤。

数据清洗完之后，下一步预训练数据的配比。首先我们认为Llama2-70b本身的中文理解能力是足够的，而中文知识比较匮乏，因此我们在增量预训练着重训练知识类数据（比如百科、书籍、教材、试题、论文等），之后是综合类数据（网页文本、新闻资讯、社交媒体类等），同时为了保持英文能力，我们在预训练中加入了部分英文数据集SimPajama。

同时我们积累了海量的金融语料，涵盖了研报，财报，公告，资讯，论坛，金融百科、金融书籍试题等，因此为了提升模型的金融理解能力，我们也在增量预训练过程中，加入了高质量的金融语料数据。

综合中英、通用金融来说，我们的增量预训练过程中数据整体配比介绍如下：

- 中英数据配比大致在3:1的比例，英文数据从SimPajama中类别中进行抽样得到；中文数据则包含通用和金融领域数据。
- 分阶段调整数据配比：前期中文数据以知识类为主，预期让模型学到更多中文知识，随着训练的进行，我们逐步加大其他类型的中文数据，来增加多样性。 同时金融数据的比例也是随着训练的进行逐步提升，最初设置为1:9。训练的最终阶段，金融与通用数据的比例大概为1:4左右



# 模型训练

本部分主要包含我们的在模型训练层面的细节，主要目的是（1）如何能够快速高效的训练llama2-70b，（2）提升模型的上下文长度。

### **（1）增量预训练**

考虑到原生llama2的中文能力较弱，需要进行中文增强。首先llama2的原始词表包含32000个token，其中中文字符很少，导致大部分的中文内容，需要解码成多个unicode字符，因此同样的文本内容，编码序列会变长，解码速度会降低。

因此首先需要扩充中文词表，将常见的中文字符加入到词表中。然后基于新词表进行中文增量预训练。 

首先是词表扩充，一般有基于词粒度和字粒度的扩充，各有优劣：

- 词粒度：词表扩充幅度大，一般需要20k以上，对原始模型形成破坏更多，且对字粒度的任务不友好；不过解码效率会更高，词表压缩率高，同样的内容，tokenize之后的序列更短。
- 字粒度：中文字粒度数量少，一般需要5-10k，相对来说，对原始模型破坏较少。 不过相比词粒度，压缩率会偏低，解码效率也偏低。

综合考虑llama2-70b的参数量，保险起见选择了字粒度扩充，一共新加入约7k的字符，共达到39k左右的词表大小。

之后是增量预训练策略，常见也有多种方法：

- 扩充之后只更新模型开始的embeding和最后的解码线性层，其他参数保持不更新
- 扩充之后全量更新所有模块的参数。

我们选择两者结合训练方法：

- 一阶段使用较少的数据仅仅更新模型的词表特征以及解码线性层，模型其他参数保持不变，这一阶段目的是让模型适应新加入的token，纠正原始模型的中文解码方式。本身llama2的中文理解能力已经足够，只是中文知识缺乏。因此这一步的目的是纠正中文的解码方式。
- 二阶段则使用大量的中英数据，对模型进行全参数更新，目的是保持英文能力不下降的同时，中文能力通过参数更新获得提升

### （2）模型训练加速

本部分主要介绍关于训练速度优化方面的工作，从如下三个层面展开：

**（1）数据加载优化**

我们将文本数据提前做tokenize，预处理存储为二进制格式，预训练阶段直接加载二进制索引，可以大幅提升数据加载速度，降内存占用。相比直接加载文本数据，更加高效，单机内存可以轻松快速加载TB级别的数据。

**（2）模型结构优化**

本部分介绍在模型本身上的优化措施，具体包括：

- 使用Flash Attention2 替代self-attention, 相比原始实现，Flash Attention2能够大幅降低显存占用，提升训练速度。
- 算子融合操作：使用一些cuda融合算子来替代原始Llama2的python实现，可以提升训练速度

备注：由于模型上下文长度较长，我们也修复了llama2模型中因为bfloat 16导致rotary embdding的参数冲突的问题。



**（3）分布式训练底层优化**

我们使用DeepSpeed来作为基本的分布式训练框架，并且从多个方面来进行了优化增强：

- 优化分布式训练网络配置，提升多机多卡分布式通信效率，包括RoCE高性能网络优化及网络拓扑感知，提升网络吞吐及降低时延，实现多机加速比达到90%以上（如40台吞吐351tokens/s，100台吞吐340tokens/s/gpu，340/351=96.8%)
- 通过故障自愈和弹性扩缩容策略，实现快速发现故障（GPU硬件，机器，网络等）及训练恢复，提升训练稳定性和健壮性。降低训练中断时间，70B模型的故障平均恢复时间在1小时内。



# 总结

本次我们开源的XuanYuan-70B模型弥补了原始Llama2-70B的中文较弱的缺点，此外为了更好服务金融场景，我们加入了更多的金融领域数据，提升金融理解的能力。

近期我们会开源XuanYan-70B-Chat模型，在sft阶段加入更多高质量的金融指令和通用问答数据，以及更长上下文的XuanYuan-70B-16K版本。此外我们也会持续分享数据和模型训练方面的实践细节，也欢迎大家加入我们主页的微信群进行交流。

