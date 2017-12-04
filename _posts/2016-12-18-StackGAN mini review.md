---
layout: post
title: "StackGAN mini review"
date: 2016-12-18
abstract: "记录一点见解"
excerpt: ""
comments: true
category: blogs

---

<!-- # StackGAN mini review -->
[arxiv 传送门](https://arxiv.org/abs/1612.03242v1)

最近 NIPS 上有一篇关于 GAN 的论文很受关注，在 [reddit](https://www.reddit.com/r/MachineLearning/comments/5i23wt/r_stackgan_text_to_photorealistic_image_synthesis/) 上也有不少讨论，因为它的生成结果实在很 impressive  ，如图：

![](http://ww2.sinaimg.cn/large/006tNc79gw1fatr0fwl63j30pt0d40w7.jpg)

> 最下一排是 stackGAN ，生成的图片不仅分辨率最高，而且最真实。



stackGAN 的主要的 motivation 是，如果我们没办法一次生成高分辨率又 plausible 的图片，那么可以分两次生成，第一阶段生成。分阶段生成图片的想法并不鲜见，*Denton et all* 的 [LapGAN](https://arxiv.org/abs/1506.05751) 的想法就是将生成的低分辨率的图片一次次 refine ，*XiaoLong et all* 的 [S^2GAN](https://arxiv.org/abs/1603.05631) 也是将生成图片的过程分成了 Structure and Style 两个阶段。当然这两者都没有 caption 作为 condition ，没法比较。

关于 text to image 这一任务，之前就有人用 GAN 做，比如 [Generative Adversarial Text to Image Synthesis](https://arxiv.org/abs/1605.05396) ，所用的方法思路与 stackGAN 基本一样，都是在 generator 与 discriminator 前加入 text 的embedding 作为condition 。但 stackGAN 的结构要 fancy 一点，如下。

![](http://ww2.sinaimg.cn/large/006tNc79gw1fats5ma4jmj30zu0j10xr.jpg)

#### 第一阶段：

从 embedding 开始，stackGAN 没有直接将 embedding 作为 condition ，而是用 embedding 接了一个 FC 层得到了一个正态分布的均值和方差，然后从这个正态分布中 sample 出来要用的 condition 。之所以这样做的原因是，embedding 通常比较高维，而相对这个维度来说， text 的数量其实很少，如果将 embedding 直接作为 condition，那么这个 latent variable 在 latent space 里就比较稀疏，这对我们的训练不利。我的理解是，如果 text 的数量较少，那么即使我们有比较高维的 latent variable ，但其中一大部分都是离散的 text embedding ，相当于真正的随机变量相对变少了，因此生成数据流形就会变得不连续（因为比较低维），这是我们不想看到的。而从参数化的正态分布中 sample 出要用的 condition 的话，相当于 embedding 周围的点也会被作为 condition ，这就增加了 text 的数量和 condition 的维数。为了防止这个分布 degenerate 或者方差太大的情况，generator 的 loss 里面加入了对这个分布的正则化：$D_{KL}(\mathcal{N} (\mu(\phi_t), \Sigma(\phi_t)) \|\| \mathcal{N} (0, I))$ 。

generator 使用的并不是常用的 Deconv ，而是若干个上采样加保持大小不变的 3x3 的 conv 的组合，我还是第一次见到这种架构。

discriminator 是若干步长为 2 的 conv ，再与 resize 的 embedding 合起来，接一个 FC。

#### 第二阶段：

第二阶段的 generator 并没有噪声输入，而是将第一阶段的 sample downsample 以后与 augmented embedding (sampled from gaussian) 合起来作为输入。经过若干 residual blocks ，进行与第一阶段相同的上采样过程得到图片。

第二阶段的 discriminator 与第一阶段大体相同。



#### 我的看法

说实在，stackGAN 并没有太多让人眼前一亮的新颖观点与方法，但是它将 two-phased generation, sentence-embedding, semi-supervised learning 结合到了一起，实验也做的很好（这样的结构我觉得应该很难训练）。看到 reddit 上有人说，unsupervised learning 本来是以不用辛辛苦苦的标 label 为目的而发展的，但有点讽刺的是，给越多 label ，它 work 得越好。这当然是必然的，给越多 label ，generator 就可以将一个复杂的分布（譬如 imagenet ），分解成若干简单的，维数低的分布进行建模，discriminator 也可以分别做判别。

另外，paper 只在最后给了一些失败的 sample ，一部分是图文不符，一部分是太像生成样本，前者我倾向于认为可能是 embedding 的失误。鉴于结构的复杂性，我觉得不会看到开源实现（如果作者不放出源代码的话），我确实挺关心他的正确率。