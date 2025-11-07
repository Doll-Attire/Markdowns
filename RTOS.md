### 必要性
裸机的单片机程序都建立在while1中执行，处理复杂外设调度和时序时极其容易冲突，使用rtos可以避免繁杂的状态机和时间戳判断，也即是直接放心大胆用HAL_Delay 

在多任务系统中，根据程序的功能，把这个程序主体分割成一个个独立的，无限循环且不能返回的子程序，称之为任务。每个任务都是独立的，互不干扰的，且具备自身的优先级，它由操作系统调度管理。加入操作系统后，开发人员不需要关注每个功能模块之间的冲突，重心放在子程序的实现。缺点是整个系统随之带来的额外RAM开销

### CMSIS
一个统一的接口层，可以使用其API统一调用FreeRTOS、RT_Thread等不同操作系统的实现统一功能的不同函数。
```c
///free
xTaskCreat()
///RT
rt_thread_creat()

///CMSIS统一调用
osThreadNew()


```

### ARM架构
  属于精简指令集，SOC( 片上系统 )自带CPU与内存、flash。后者保存程序，cpu运行，运行的过程中用到内存。ARM芯片有以下特点：
  - 内存没有计算功能，只能被读写内存
  - 所有计算在CPU中执行
  - 使用RISC指令集，CPU复杂度小，易于设计。
  ```
 例如,执行一个A+B
 首先,cpu从内存中读取a,b
 然后cpu从flash中读取机器码，内部计算单元计算a+b
 最后得到的结果写回内存中
 ```
### 堆Heap
定义：一块空闲的内存，可以从中分配处一个小buffer,用完后再放回去。例如c：


-   malloc：从堆里划出一块空间给程序使用
-   free：用完后，再把它标记为"空闲"的，可以再次使用
```
char heap_buf[ 1024 ] ;
int pos =0 ;

void* my_malloc( int size )
{
	int old_pos = pos ;
	pos += size ;
	return &heap_buf[pos] ;

}

void my_free( void *buf)
{
//实现释放堆中内存,这个堆管理函数因为没有头部来储存堆大小等信息,因此free只是示意
}

int main(void)
{
char ch = 65 ;//'A'
int i ;
char *buf = my_malloc(100);
unsigned cahr uch = 200 ;

for (i = 0; i<26 ; i++)
	{
	buf[i] = 'A' + i ;
	}

}
```
### 栈Stack
也是内存空间，CPU的SP寄存器指向他，可以用于函数调用、局部变量、多任务系统保存现场。
C语言中的函数调用使用汇编BL指令，主要完成两个任务：
>将下一个指令的地址（返回地址）放入LR寄存器 --回哪里
>将目标函数的地址放入PC寄存器 --去哪里

因为函数之间的嵌套调用会导致LR被覆盖，所以函数入口一般默认用PUSH指令保存LR与必要的寄存器（保存在内存栈区，专门一块A函数的栈 ），被执行完后，栈被回收，sp指针回到之前的位置。
 
### 内存管理
FreeRTOS中内存管理的接口函数为：pvPortMalloc 、vPortFree，对应于C库的malloc、free。 文件在FreeRTOS/Source/portable/MemMang下，它也是放在portable目录下，表示你可以提供自己的函数，有五种，参考[第8章 内存管理 | 百问网](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter8.html#_8-2-freertos%E7%9A%845%E4%B8%AD%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%96%B9%E6%B3%95)

### 任务
核心要素：
- 要做什么 - 函数
- 栈
- 优先级

在FreeRTOS中，任务就是一个函数，原型如下：
```c
void ATaskFunction( void *pvParameters )
{
	/* 对于不同的任务，局部变量放在任务的栈里，有各自的副本 */
	int32_t lVariableExample = 0;
	
    /* 任务函数通常实现为一个无限循环 */
	for( ;; )
	{
		/* 任务的代码 */
	}

    /* 如果程序从循环中退出，一定要使用vTaskDelete删除自己
     * NULL表示删除的是自己
     */
	vTaskDelete( NULL );
      
    /* 程序不会执行到这里, 如果执行到这里就出错了 */
}
```
我们使用链表来管理任务，对于每个任务，都有TCB（TaskControlBlock）结构体，

[基于STM32F407ZGT6的硬件平台，（可选CubeMX） + PlatformIO软件开发的FreeRTOS部署指南_platformio freertos-CSDN博客](https://blog.csdn.net/charlie114514191/article/details/146242902?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522ce4833ffb60ffe1381485e7524d1cfe2%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=ce4833ffb60ffe1381485e7524d1cfe2&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-146242902-null-null.142^v102^pc_search_result_base6&utm_term=platformio%20freertos&spm=1018.2226.3001.4187)
 
### 任务切换 --[【FreeRTOS】动画搞懂任务切换到底是怎么回事！_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ErWyziEtw?spm_id_from=333.788.player.switch&vd_source=740e28c14abb598f1fe8be2d12484cbb)
PC寄存器存放着Flash中存储的"要执行的下一条指令"的地址，我们进行多任务的原理就是依靠修改寄存器所实现的FreeRTOS的时间片机制。
创建任务时，就是对每个任务创建任务栈，其功能之一就是存放“时间片”耗尽切换任务时寄存器的值。
但这时会发现，时间片机制很可能会被延时等因素影响从而导致浪费CPU，应该怎么解决？

##### 任务状态
分为
| 就绪态 | 运行态  |阻塞态|挂起态
|--|--|--|--|
| rt,创建任务后也会自动进入 | 如题 |延时等待|等待数据或状态|
本质上是时间片的放弃和获得。
 
##### 任务优先级
字面意思。需要注意的是当高优先级任务进入运行态后，会有恶劣影响，哪怕它执行完了进入就绪态，也会立刻获得时间片从而导致低优先级的进入“任务饥饿”，由此衍生出抢占与协调两种模式，显然抢占更符合RealTime的宗旨
**因此设计任务时就要考虑让其通过延时或者等待数据等方法离开运行和就绪态。**


###  临界段
定义
---
>一段在执行的时候不能被中断的代码段。在FreeRTOS里面， 这个临界段最常出现的就是对全局变量的操作

<!--stackedit_data:
eyJoaXN0b3J5IjpbNjU1MTA1NDMsLTU5NzMwMzc2OSwtMjM2OT
E2MDIxLC0xMDUzMTc1ODE1LC04OTY3MDg4MjIsLTEyMTY1MzU4
NTcsLTExNjcwOTk3NTYsOTU3MDU0MjA3LDE2MjYzNzM3NzIsLT
E0NzgyMjcxMjAsNTg2NDY2MzIzLDE4NjczNTQzNDEsNDg1NDg2
NjAsLTE0NTY3MDAyNjYsNjgzMzI4MTM4LDEyMTE5MTI4OTAsLT
E1NjMyNzk2ODYsLTE4MjQ4MTYxNDIsOTMwNzY3NzAsNDI0MTc0
ODEyXX0=
-->