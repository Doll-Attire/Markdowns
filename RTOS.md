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
  
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3Njg4Mjc3MDYsLTE0MTg1NjU0ODksLT
kxNjEzNzU2XX0=
-->