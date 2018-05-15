---
title:  "On the Integration of Optical Flow and Action Recognition 阅读笔记"   
date:   2018-05-5 13:52:23  
categories: [ActivityRecognition]  
tags: [ReadingNote]  

---

<script type="text/javascript"
   src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
<script type="text/x-mathjax-config"> MathJax.Hub.Config({ TeX: { equationNumbers: { autoNumber: "all" } } }); </script>


---

## Abstract

### 本文结论
本文深入揭示了光流和行为识别的结合，说明了为什么光流信息有用，怎样让光流信息更有用。
使用了不同光流算法和输入图像变换来理解光流如何影响state-of-the-art的方法。在UCF-101上使用了两种端到端的学习模型。得到了五点结论：
> 1.光流对行为识别非常有用，因为它 **invariant to appearance** （外观不变性） 
> 2.光流方法通过减小 end-point-error(EPE)可以得到优化，但是现有方法的EPE和识别性能联系不密切(well correlated)。  
> 3.对于光流方法测试，边界的准确率和小的偏移和行为识别性能关系很大。  
> 4.训练时，让光流最小化分类错误而不是EPE可以提高识别性能。  
> 5.行为识别学习到的光流信息和传统在人体内和身体边界的光流差别很大。

**EPE：average Euclidean distance between the ground truth and the estimated flow.**

### 关于光流的假设
在视频域，有两个问题，运动估计和行为识别， 运动估计即计算光流信息，使用EPE来计算得到？而后和raw image一起输入到网络进行识别，而这些基于几个未经论证的假设：

> **Hypothesis 1: 两帧图像之间的光流信息是用来做视频分类的很好的特征。**  
> **Hypothesis 2: 光流的准确率和行为识别的准确率相关联。**  
> **Hypothesis 3: 光流是对于行为识别最好的运动表示。**  


#### 假设1说明:	
大多行为都属于human-object interactions,可以从单帧图像中识别得到。因为现有数据集关于scenes/objects 和分类之间的高关联性，如Guitar和Playing Guitar，而光流不行。但实验结果是通常单个flow识别甚至比raw image高，因此自然要问**运动信息为何在视频分类里如此有用**

##### 实验工作：  
>1.随机shuffling了一些光流信息，发现准确率只下降了很小一些，证明了运动轨迹不是光流里信息的主要来源。  
>2.随机打乱了输入图像的次序，正确率大大下降，然鹅仍然比随机猜测高出60倍。  
>3.为了证明flow 的invariant to appearance,更改了输入图像的外观，发现raw image降低了50%，而optical flow降低了1%。

在文献[7]中证明了当在非常大的数据集上时，使用光流信息的网络比只使用图像的网络正确率低。因为随着数据集变的非常庞大，逐渐变得对外观变换不敏感。

![实验结果](http://owvctf4l4.bkt.clouddn.com/image_2018_5_5.png)
#####总结：  
运动轨迹不是光流有效的原因，至于光流为什么对运动表示如此有用还是个疑问。

#### 假设2说明：

##### 实验工作：
> 1.使用不同的光流方法作为baseline system的输入，并且测试了 stanndard optical flow benchmarks的准确率。

#####总结：
光流方法的EPE和系统的分类性能关系不大，而在运动边界和小偏移的准确估计和分类性能关系很大。
#### 假设3说明：
##### 实验工作
使用FlowNet和SpyNet,可以近似运动估计，发现比原始光流效果好，FlowNet计算开销大。



