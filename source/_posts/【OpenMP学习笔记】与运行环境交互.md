title: 【OpenMP学习笔记】与运行环境交互
date: 2016-02-26 09:23:49
tags: ["OpenMP", "学习笔记"]
categories: OpenMP
---
## Internal Control Variables
OpenMP标准定义了内部控制变量(internal control variables), 这些变量可以影响程序运行时的行为, 但是它们不能被直接访问或者修改, 我们需要通过OpenMP函数或者环境变量来访问或者修改它们, 下面是被定义的内部变量
* nthread-var : 存储并行域的线程数量
* dyn-var : 控制在并行域执行时是否可以动态调整线程的数量
* nest-var : 控制在并行域执行时是否允许嵌套并行
* run-sched-var : 存储在循环域(loop regions)使用 runtime 调度子句时的调度类型
* def-sched-var : 存储对于循环域默认的调度类型

<!-- more -->
## nthread-var
我们可以通过以下几种方式来设置线程数量  

__OMP_NUM_THREADS__ 
我们可以在命令行(command line)下设置OMP_NUM_THREADS环境变量的值, 而该变量的值用于初始化 nthread-var 变量.  

__omp_set_num_threads__
在程序中我们可以使用omp_set_num_threads函数来设置线程数量, 语法形式为`omp_set_num_threads(integer)`

__num_threads__
最后我们可以在构造并行域的时候使用num_threads子句来控制线程的数量

上面的三种方式优先级依次递增, 另外在程序执行时, 我们可以使用下面几个函数获得线程的数量信息
* omp_get_max_threads : 获得可以使用的最大线程数量, 数量是可以确定的, 与在串行域还是并行域调用无关. 
* omp_get_num_threads: 获得当前运行线程的数量, 如果不在并行域内调用则返回1
* omp_get_thread_num: 获得线程的编号, 从0开始

下面是一个使用示例
```C
void test_numthread() {
    printf("max thread nums is %d\n", omp_get_max_threads());
    printf("omp_get_num_threads: out parallel region is %d\n", omp_get_num_threads());
    
    omp_set_num_threads(2);
    printf("after omp_set_num_threads: max thread nums is %d\n", omp_get_max_threads());
#pragma omp parallel 
    {
        #pragma omp master
        {
            printf("omp_get_num_threads: in parallel region is %d\n\n", omp_get_num_threads());
        }
        printf("1: thread %d is running\n", omp_get_thread_num());
    }
    printf("\n");
#pragma omp parallel num_threads(3)
    {
        printf("2: thread %d is running\n", omp_get_thread_num());
    }
}
```
下面是程序运行结果:
```html
max thread nums is 4
omp_get_num_threads: out parallel region is 1
after omp_set_num_threads: max thread nums is 2
omp_get_num_threads: in parallel region is 2

1: thread 0 is running
1: thread 1 is running

2: thread 0 is running
2: thread 1 is running
2: thread 2 is running
```

## dyn-var
dyn-var控制程序是否在运行中是都可以动态的调整线程的数量, 可以通过下面的两种方式来设置

__OMP_DYNAMIC__
通过OMP_DYNAMIC环境变量来控制, 如果设为true, 则代表允许动态调整, 设为false则不可以

__omp_set_dynamic__
通过omp_set_dynamic函数, omp_set_dynamic(1)表示允许, omp_set_dynamic(0)表示不可以, 注意omp_set_dynamic可以传入其他非负整数, 但是作用和输入1是相同的, 都是表示true.

可以通过omp_get_dynamic来获得dynamic的状态, 返回值为0和1, 下面是一个使用示例:
```C
void test_dynamic() {
    printf("dynamic state is %d\n", omp_get_dynamic());

    omp_set_num_threads(6);

    #pragma omp parallel 
    {
        printf("thread %d is running\n", omp_get_thread_num());
    }

    omp_set_dynamic(1);

    printf("\n");
    printf("dynamic state is %d\n", omp_get_dynamic());
    #pragma omp parallel 
    {
        printf("thread %d is running\n", omp_get_thread_num());
    }
}
```
下面是输出结果:
```html
dynamic state is 0
thread 3 is running
thread 4 is running
thread 0 is running
thread 5 is running
thread 1 is running
thread 2 is running

dynamic state is 1
thread 3 is running
thread 1 is running
thread 2 is running
thread 0 is running
```
当允许动态调整之后, 第二个for循环只打印了四次,即只有四个线程在执行. 一般来说动态调整会根据系统资源来确定线程数量, 大多数情况下会生成和CPU数目相同的线程. 还有一点, 动态调整时生成的线程不会超过当前运行环境所允许的最大线程数量, 在上面的代码中, 如果将`omp_set_num_threads(6)`改为`omp_set_num_threads(2)`, 那么动态调整时最多只会生成两个线程. 

## nest-var
nest-var用来控制是否可以嵌套并行, 可以通过下面两种方式来设置

__OMP_NESTED__
通过设置OMP_NESTED环境变量, true表示允许, false表示不允许

__omp_set_nested__
通过omp_set_nested函数, omp_set_nested(1或其他非负整数)表示允许, omp_set_nested(0)表示不允许.

可以通过omp_get_nested来获得是否可以嵌套并行, 返回值是0或1, 下面是一个使用示例:
```C
void test_nested() {
    int tid;
    printf("nested state is %d\n", omp_get_nested());

    #pragma omp parallel num_threads(2) private(tid)
    {
        tid = omp_get_thread_num();
        printf("In outer parallel region: thread %d is running\n", tid);
        
        #pragma omp parallel num_threads(2) firstprivate(tid)
        {
            printf("In nested parallel region: thread %d is running and outer thread is %d\n", omp_get_thread_num(), tid);
        }
    }

    omp_set_nested(1);
    printf("\n");
    printf("nested state is %d\n", omp_get_nested());

    #pragma omp parallel num_threads(2) private(tid)
    {
        tid = omp_get_thread_num();
        printf("In outer parallel region: thread %d is running\n", tid);
        
        #pragma omp parallel num_threads(2)
        {
            printf("In nested parallel region: thread %d is running and outer thread is %d\n", omp_get_thread_num(), tid);
        }
    }
}
```
下面是程序运行结果:
```C
nested state is 0
In outer parallel region: thread 0 is running
In nested parallel region: thread 0 is running and outer thread is 0
In outer parallel region: thread 1 is running
In nested parallel region: thread 0 is running and outer thread is 1

nested state is 1
In outer parallel region: thread 1 is running
In outer parallel region: thread 0 is running
In nested parallel region: thread 0 is running and outer thread is 0
In nested parallel region: thread 0 is running and outer thread is 1
In nested parallel region: thread 1 is running and outer thread is 1
In nested parallel region: thread 1 is running and outer thread is 0
```
当不允许嵌套并行时, 在并行域内创建的新并行域会以单线程执行, 而允许嵌套并行之后, 会在并行域内创建新的并行域, 为其分配新的线程执行.

## def-sched-var
通过OMP_SCHEDULE环境变量, 可以设置循环调度为runtime时的调度类型, 具体参见[这里](http://localhost:4000/2016/01/25/OpenMP%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E7%BC%96%E8%AF%91%E6%8C%87%E4%BB%A4/#schedule)

## 其它函数
__omp_get_num_procs__
获得程序中可以使用的处理器数量, 是一个全局的值

__omp_in_parallel__
判断是否在一个活跃的并行域(active parallel region)内, 返回0或1.

