---
title: CS221_2019_autumn assigment1
date: 2020-12-29 16:26:06
tags:
- 人工智能学习
- 斯坦福课程
- 人工智能导论
updated:
type:
comments:
description:
keywords: 斯坦福CS221人工智能基础课
cover: /img/Artificial_intelligence.jpg
top_img:
mathjax: 
katex: true
aside:
aplayer:
highlight_shrink:
categories: 
- 人工智能学习
---
# Foundations解答
## Probelm 1: Optimization and probability
### a题
#### 题目
$$ f(\theta)= \frac{1}{2} \sum_{i=1}^n w(\theta-x_i)^2 $$ 
求使 $f(\theta)$ 最小的 $\theta$ 值
#### solution
$$ \frac{\partial f(\theta)}{\partial \theta} = \sum_{i=1}^n w_i(\theta-x_i) $$
令$\frac{\partial f(\theta)}{\partial \theta}=0$有
$$ \theta=\frac{\sum_{i-1}^n w_i x_i}{\sum_{i-1}^n w_i} $$
其二阶导形式如下
$$ \frac{\partial ^n f(\theta)}{\partial \theta ^n} = \sum_{i=1}^n w_i\theta $$
可见如果$w_i$不为正实数会导致函数$f(\theta)$没有最小值
### b题
switch sum or maxes order


solution


$$ f(x)=\sum_{i=1}^d max_{S\in\{ {1,2 } \}} sx_i \leq d max|x_i|$$
$$ g(x)=\max_{S\in\{ {1,2 } \}}\sum_{i=1}^d sx_i \leq max d|x_i|\leq dmax|x_i|$$
$$ g(x) \leq f(x)$$

### c题

