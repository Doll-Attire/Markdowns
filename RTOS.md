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
### 堆
定义：一块空闲的内存，例如c：
```
char heap_buf[ 1024 ] ;
int pos =0 ;

void* my_malloc( int size )
{
	int old_pos = pos ;
	pos += size ;
	return &heap_buf[pos] ;

}
```
h
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI1ODk2MDIyOCwxOTg5MzMyNDQ0LDMyMT
E4OTA2MywzMTcxNzQzNDksLTEzMTY2OTIwNzgsLTExMjgyOTIz
NTRdfQ==
-->