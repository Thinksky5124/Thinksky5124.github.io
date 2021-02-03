---
title: CPU并行计算
date: 2021-01-07 18:27:06
tags:
- 并行计算
- CMU课程
updated:
type:
comments:
description: 个人学习CMU并行计算课程时的笔记、总结等
keywords: 并行计算 CMU课程
cover: /img/cover_one.png
top_img:
mathjax: 
katex: true
aside:
aplayer:
highlight_shrink:
categories: 
- 并行计算
---
# Lecture 2 A Modern Multi-Core Processor
- 理解并行计算的形式
- 理解延迟(latency)和带宽(bandwidth)

## Parallel Execution

### 单线程执行 - 编译器定义
例程:使用泰勒公式计算$sin(x)$。
```C++
void sinx(int N,int terms, float* x,float* result){
    for(int i=0;i<N;i++){
        float value = x[i];
        float numer = x[i] * x[i] *x[i];
        int denom = 6;
        int sign = -1;

        for(int j=1;j<= terms;j++){
            value += sign * numer / denom;
            numer *= x[i] * x[i];
            denom *= (2*j+2)*(2*j+3);
            sign *= -1;
        }

        result[i]=value;
    }
}
```

{% gallery %}
![单核单线程单流水处理模型](./CPU并行计算/Single_chip_excution.jpg)
{% endgallery %}

- 解码模块(Fetch/Decode)读取指令
- ALU模块负责执行
- 上下文存储器(Exceution Context)负责存储执行数据

### 多线程执行 - 用户定义

概念：指令级并行 instruction level parallelism (ILP)

{% gallery %}
![超标量处理器模型](./CPU并行计算/Superscalar_processor.jpg)
![多核处理器模型](./CPU并行计算/Two_cores.jpg)
{% endgallery %}

例程：
```C++
typedef struct{
    int N;
    int terms;
    float *x;
    float result;
} my_args;

void parallel_sinx(int N,int terms, float* x,float* result){
    pthread_t thread_id;
    my_args args;

    args.N=2/N;
    args.terms=terms;
    args.x=x;
    args.result=result;

    pthread_create(&thread_id, NULL, my_thread_start, &args);//launch thread
    sinx(N-args.N,terms,x+args.N,result+args.N);
    pthread_join(thread_id, NULL);
}

void sinx(int N,int terms, float* x,float* result){
    for(int i=0;i<N;i++){
        float value = x[i];
        float numer = x[i] * x[i] *x[i];
        int denom = 6;
        int sign = -1;

        for(int j=1;j<= terms;j++){
            value += sign * numer / denom;
            numer *= x[i] * x[i];
            denom *= (2*j+2)*(2*j+3);
            sign *= -1;
        }

        result[i]=value;
    }
}
```
上述例程表示线程级并行的代码描述，通过将整个工作分散分配到不同的处理器核中，以获得加速的效果。工作的分配方式将很大程度上影响加速的效果。

### 数据并行 - 用户定义、编译器定义
{% gallery %}
![SMID](./CPU并行计算/SMID.jpg)
{% endgallery %}

概念：SMID 单指令多数据处理


使用AVX指令集的代码如下：
```C++
#include <immintrin.h>
void sinx(int N, int terms, float* x, float* sinx)
{
    float three_fact = 6; // 3!
    for (int i=0; i<N; i+=8)
    {
        __m256 origx = _mm256_load_ps(&x[i]);
        __m256 value = origx;
        __m256 numer = _mm256_mul_ps(origx, _mm256_mul_ps(origx, origx));
        __m256 denom = _mm256_broadcast_ss(&three_fact);
        int sign = -1;
        for (int j=1; j<=terms; j++)
        {
            // value += sign * numer / denom
            __m256 tmp = _mm256_div_ps(_mm256_mul_ps(_mm256_broadcast_ss(sign),numer),denom);
            value = _mm256_add_ps(value, tmp);
            numer = _mm256_mul_ps(numer, _mm256_mul_ps(origx, origx));
            denom = _mm256_mul_ps(denom, _mm256_broadcast_ss((2*j+2) * (2*j+3)));
            sign *= -1;
        }
        _mm256_store_ps(&sinx[i], value);
    }
}
```
按照上图所示CPU模型，通过使用SMID单个核可以一次指令同时处理8个数据，达到并行处理数据的效果。

另一种自动优化代码的写法，前提是循环loop之间相互独立

```C++
void sinx(int N, int terms, float* x, float* result)
{
    // declare independent loop iterations
    forall (int i from 0 to N-1)
    {
        float value = x[i];
        float numer = x[i] * x[i] * x[i];
        int denom = 6; // 3!
        int sign = -1;
        for (int j=1; j<=terms; j++)
        {
            value += sign * numer / denom
            numer *= x[i] * x[i];
            denom *= (2*j+2) * (2*j+3);
            sign *= -1;
        }
        result[i] = value;
    }
}
```

### 条件执行

假设数据并行处理器需要执行如下代码
```c++
float x = A[i];
if(x>0){
    float tmp = exp(x,5.f);
    x = tmp + kMyConst2;
}else{
    float tmp = kMyConst1;
    x = 2.f * tmp;
}
result[i] = x;
```
在处理器进行分支预测之后，处理器会首先并行执行值未真的那些ALU，再之后执行那些值为假的ALU，以达到并行处理的效果。
{% gallery %}
![在数据并行时进行条件执行](./CPU并行计算/SIMD_Condition.png)
{% endgallery %}

### SMID的有关概念
- 要使用并行处理需要程序手动进行特定编码，比如使用：SSE、AVX等代码。
- 但是一旦使用并行处理，编译器无法检查与保证循环的独立性，需要自行保证
- GPU的数据并行处理性能要高于CPU的数据并行处理性能

比如：
{% gallery %}
![CPU与GPU的SMID能力对比](./CPU并行计算/Processor_Compare.png)
{% endgallery %}

## Accessing Memory

### 术语
- 内存延时：内存延迟是指等待对系统内存中存储数据的访问完成时引起的延期。 单位：100机器周期、100毫秒
- 内存带宽：内存系统可以提供给处理器数据的速度。 单位：20GB/s
- 吞吐量(throughput):芯片单位时间处理数据的多少

现代处理器中不可避免因为内存延时而降低CPU处理速率，但是可以通过一些办法来“隐藏”内存延时，比如：
- 多级缓存
- 预储存
- 多线程技术
- 存储执行上下文

{% gallery %}
![CPU与GPU的内存结构对比](./CPU并行计算/CPUvsGPU.png)
{% endgallery %}