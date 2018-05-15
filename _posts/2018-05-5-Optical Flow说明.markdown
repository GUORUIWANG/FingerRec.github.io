---
title:  "Optical Flow笔记记录"   
date:   2018-05-05 13:52:23  
categories: [ActivityRecognition]  
tags: [ReadingNote]  

---

<script type="text/javascript"
   src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
<script type="text/x-mathjax-config"> MathJax.Hub.Config({ TeX: { equationNumbers: { autoNumber: "all" } } }); </script>


---
>最近总是涉及到光流信息和运动估计，不是很了解，因此单独做个记录。

## Optical Flow

### Ground Truth的获得
1.用摄像机和3Dlaser scanner(3D 激光扫描仪)获得真实光流。但无法捕捉天空之类的远处物体的运动，导致稀疏的Ground Truth。