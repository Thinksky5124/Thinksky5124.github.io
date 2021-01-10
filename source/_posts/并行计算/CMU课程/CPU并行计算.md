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

## parallel execution

### 单线程执行
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
- ALU模块复制执行
- 上下文存储器(Exceution Context)负责存储执行数据

### 多线程执行
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

