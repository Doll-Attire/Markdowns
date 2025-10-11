### C语言进阶
---
# 数据存储_数据类型

常见数据类型的内存大小
```
char	1
short	2
int		4
long	4/8取决平台位数
longlong	8		C99规范下
float		4
double	8
```
类型可以归类为五个基本情况:
> ## 整形家族
>  包括char , short , int ,  long的有符号和无符号版本

>## 浮点数家族
>float , double等

>## 构造类型
>如数组类型（个数不同也属于不同类型），结构体struct，枚举类型enum，联合类型union。

>## 指针类型


>## 空类型
>void，表示无类型。注意void*是属于指针类型的。

下面我们进行区别的讲解：

----

## 整形在内存中的存储

正数的二进制标售有三种
1. 正数的三者相同
2. 负数的三者需要计算

- 原码 : 直接写出的二进制序列
- 反码 : 符号位不变 , 其他按位取反
- 补码 : 反码+1

为什么要用补码？--》可以把符号位和数字域统一处理，
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MjU5NzQ4MiwtNDA5Nzk4NTQsLTY5Nz
c3MzgzMCwtMjEyMTM2Njg4NF19
-->