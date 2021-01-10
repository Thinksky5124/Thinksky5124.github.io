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
cover: https://stanford-cs221.github.io/autumn2020/plugins/stanford_seal.gif
top_img:
mathjax: 
katex: true
aside:
aplayer:
highlight_shrink:
categories: 
- 人工智能学习
- 斯坦福CS221人工智能原理与技术
---

课程网址：https://github.com/stanford-cs221/autumn2019

注：本作业解答是2019年秋季学期的

# Foundations解答
## Probelm 1: Optimization and probability
### a题

#### 题目

Let x1,…,xn be real numbers representing positions on a number line. Let w1,…,wn be positive real numbers representing the importance of each of these positions. Consider the quadratic function: f(θ)=12∑ni=1wi(θ−xi)2. What value of θ minimizes f(θ)? What problematic issues could arise if some of the wi's are negative?

Note: You can think about this problem as trying to find the point θ that's not too far away from the xi's. Over time, hopefully you'll appreciate how nice quadratic functions are to minimize.

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
#### 题目
In this class, there will be a lot of sums and maxes. Let's see what happens if we switch the order. Let f(x)=∑di=1maxs∈{1,−1}sxi and g(x)=maxs∈{1,−1}∑di=1sxi, where x=(x1,…,xd)∈Rd is a real vector. Does f(x)≤g(x), f(x)=g(x), or f(x)≥g(x) hold for all x? Prove it.

Hint: You may find it helpful to refactor the expressions so that they are maximizing the same quantity over different sized sets.

#### solution


$$ f(x)=\sum_{i=1}^d max_{S\in\{ {1,2 } \}} sx_i \leq d max|x_i|$$
$$ g(x)=\max_{S\in\{ {1,2 } \}}\sum_{i=1}^d sx_i \leq max d|x_i|\leq dmax|x_i|$$
$$ g(x) \leq f(x)$$

### c题

#### 题目
Suppose you repeatedly roll a fair six-sided die until you roll a 1 (and then you stop). Every time you roll a 2, you lose a points, and every time you roll a 6, you win b points. You do not win or lose any points if you roll a 3, 4, or a 5. What is the expected number of points (as a function of a and b) you will have when you stop?

Hint: it is recommended to think of defining a recurrence.

#### solution
{% mermaid %}
graph TD;
    开始-->1;
    开始-->3\4\5;
    开始-->2;
    开始-->6;
    3\4\5-->L1_1;
    3\4\5-->L1_3\4\5;
    3\4\5-->L1_2;
    3\4\5-->L1_6;
    2-->L2_1;
    2-->L2_3\4\5;
    2-->L2_2;
    2-->L2_6;
    6-->L3_1;
    6-->L3_3\4\5;
    6-->L3_2;
    6-->L3_6;
{% endmermaid %}

$$ E(n) = \frac{1}{6}*0 + \frac{1}{3} E(n-1) + \frac{1}{6}[-a + E(n-1)] + \frac{1}{6}[b + E(n-1)] $$
$$ E(n) = \frac{2}{3} E(n-1) + \frac{b-a}{6} $$
$$ E(n) = \frac{b-a}{6} + \frac{2}{3}*\frac{b-a}{6} + (\frac{2}{3})^{2}*\frac{b-a}{6} + \cdots + (\frac{2}{3})^{n-2}*\frac{b-a}{6} $$
$$ E(n) = \frac{b-a}{6}*[1 + \frac{2}{3} + (\frac{2}{3})^{2} + (\frac{2}{3})^{3} + \cdots + (\frac{2}{3})^{n-2}] $$
$$ E(n) = \frac{b-a}{6}*(\frac{1- (\frac{2}{3})^{n-1}}{1- \frac{2}{3}}) $$
$$ \lim_{n \to \infty} E(n) = \lim_{n \to \infty}[ \frac{b-a}{6}*(\frac{1- (\frac{2}{3})^{n-1}}{1- \frac{2}{3}})] = \frac{b-a}{2} $$

### d题

#### 题目
Suppose the probability of a coin turning up heads is 0<p<1, and that we flip it 7 times and get {H,H,T,H,T,T,H}. We know the probability (likelihood) of obtaining this sequence is L(p)=pp(1−p)p(1−p)(1−p)p=p4(1−p)3. What value of p maximizes L(p)? What is an intuitive interpretation of this value of p?
Hint: Consider taking the derivative of logL(p). You can also directly take the derivative of L(p), but it is cleaner and more natural to differentiate logL(p). You can verify for yourself that the value of p which maximizes logL(p) must also maximize L(p) (you are not required to prove this in your solution).

#### solution

当$p=\frac{4}{7}$时$logL(p)$最大，求解：
$$ logL(p) = log p^4(1-p)^3 = 4logp+3log(1-p) $$
$$ \frac{dL(p)}{dp} = \frac{4-7p}{p(1-p)} = 0 , p \in [0,1] $$
$$ 易知 p = \frac{4}{7} 时,logL(p)最小 $$

### e题
#### 题目

Let's practice taking gradients, which is a key operation for being able to optimize continuous functions. For w∈Rd (represented as a column vector) and constants ai,bj∈Rd (also represented as column vectors) and λ∈R, define the scalar-valued function
f(w)=∑i=1n∑j=1n(a⊤iw−b⊤jw)2+λ∥w∥22,
where the vector is w=(w1,…,wd)⊤ and ∥w∥2=∑dk=1w2k−−−−−−−√ is known as the L2 norm. Compute the gradient ∇f(w).
Recall: the gradient is a d-dimensional vector of the partial derivatives with respect to each wi:
∇f(w)=(∂f(w)∂w1,…∂f(w)∂wd)⊤.
If you're not comfortable with vector calculus, first warm up by working out this problem using scalars in place of vectors and derivatives in place of gradients. Not everything for scalars goes through for vectors, but the two should at least be consistent with each other (when d=1). Do not write out summation over dimensions, because that gets tedious.

#### solution

$$ \bigtriangledown f(w) = 2 \sum_{i=1}^n \sum_{j=1}^n (a_i^{\top}w-b_j^{\top}w)(a_i^{\top}-b_j^{\top}) + 2 \lambda \sum_{k=1}^d w_k $$

## Problem 2: Complexity

### a题

#### 题目

Suppose we have an image of a human face consisting of n×n pixels. In our simplified setting, a face consists of two eyes, two ears, one nose, and one mouth, each represented as an arbitrary axis-aligned rectangle (i.e. the axes of the rectangle are aligned with the axes of the image). As we'd like to handle Picasso portraits too, there are no constraints on the location or size of the rectangles. How many possible faces (choice of its component rectangles) are there? In general, we only care about asymptotic complexity, so give your answer in the form of O(nc) or O(cn) for some integer c.

#### solution


### b题

#### 题目

Suppose we have an n×n grid. We start in the upper-left corner (position (1,1)), and we would like to reach the lower-right corner (position (n,n)) by taking single steps down and right. Define a function c(i,j) to be the cost of touching position (i,j), and assume it takes constant time to compute. Note that c(i,j) can be negative. Give an algorithm for computing the minimum cost in the most efficient way. What is the runtime (just give the big-O)?

#### solution