
>## SSD1306和I2C驱动的128*64OLED显示屏学习

 总体流程:
 1. 通过引出引脚,将内容发给驱动芯片
 2. 芯片存起来(自带RAM)
 3. 芯片自带时钟和扫描电路,根据显示存储器的数据自动刷新到屏幕

---
> SSD1306介绍
- 内置显示存储器GDDRAM: 128 * 64bit的SRAM,也即128 * 8 字节
- 两个电源 VDD1.65 ~ 3.3V用于ic逻辑;  VCC7~15V用于面板驱动,模块内部自带升压和降压芯片,因此不用考虑
- 支持8位并行接口,但是比较繁琐,一般用``串行``的3或4线的SPI接口或者I2C接口 ``一次发送一个bit,发送8次一字节`` 

引脚中值得注意的有BS0 ~ BS2为选择通信接口 , D0-D7为不同通信接口所需数据线,8位并行全部使用,串行的如I2C部分使用，D0为SCL，D1为SDAin，D2为SDAout。D/C#引脚在I2C下配置从机地址最低位，对命令和数据的筛别处理在数据线上。

> I2C

基础的略过。起始条件，终止条件，发送|接受数据，发送|接受应答--六大时序单元。在之前我们提到过，这分别是7位从设备地址+1位R/W#读写位，然后加上一个ACK信号，随即是8位寄存器地址，然后ACK。CO置1，可以连续发送命令+数据的组合，反之，只接受一次命令，随后连续接受数据

具体步骤是，起始之后SCL在低电平，主机在SCL低电平时，允许改变SDA的电平，随后主机松手SCL，使其回弹高电平，从机需要再此时迅速读取SDA，不断重复这个过程8次，传输的数据高位先行。（区别串口低位先行）此外，由于时钟线的存在，哪怕主机传输一半去处理中断了，SCL和SDA都会因此停止，时序不断拉长等他回来。读取时反之，SCL高电平期间主机读取SDA上的数据	。因此当SCL高电平时只要胆敢修改SDA，则就是对应的起始与终止条件。

实验中使用了软件I2C模拟，这里给出时序图与核心代码可供参考：[301650f4d15e24b923cee8029682d1a5.png (1488×555)](https://i-blog.csdnimg.cn/blog_migrate/301650f4d15e24b923cee8029682d1a5.png)
```c

void MyI2C_Init(void)
{
    // 使能GPIOB时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
    
    // 配置GPIO参数
    GPIO_InitTypeDef GPIO_InitStructure;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_OD;  // 开漏输出模式，适合I2C
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10 | GPIO_Pin_11;  // PB10=SCL, PB11=SDA
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOB, &GPIO_InitStructure);
    
    // 初始状态设置为高电平（I2C空闲时SCL和SDA都为高）
    GPIO_SetBits(GPIOB, GPIO_Pin_10 | GPIO_Pin_11);
}



void MyI2C_Start(void)
{
    MyI2C_W_SDA(1);  // SDA先置高
    MyI2C_W_SCL(1);  // SCL置高，此时总线处于空闲状态
    MyI2C_W_SDA(0);  // 在SCL高电平时，SDA从高到低跳变，产生起始信号
    MyI2C_W_SCL(0);  // SCL置低，准备发送数据
}


void MyI2C_Stop(void)
{
    MyI2C_W_SDA(0);  // SDA先置低
    MyI2C_W_SCL(1);  // SCL置高
    MyI2C_W_SDA(1);  // 在SCL高电平时，SDA从低到高跳变，产生停止信号
}


void MyI2C_SendByte(uint8_t Byte)
{
    uint8_t i;
    for (i = 0; i < 8; i ++)
    {
        // 从高位到低位依次发送（I2C规定先传最高位）
        MyI2C_W_SDA(!!(Byte & (0x80 >> i)));
        MyI2C_W_SCL(1);  // SCL高电平期间，数据有效
        MyI2C_W_SCL(0);  // SCL低电平期间，准备下一位数据
    }
}


uint8_t MyI2C_ReceiveByte(void)
{
    uint8_t i, Byte = 0x00;
    MyI2C_W_SDA(1);  // 释放SDA总线，由从机控制
    for (i = 0; i < 8; i ++)
    {
        MyI2C_W_SCL(1);  // SCL高电平，读取数据
        if (MyI2C_R_SDA()){Byte |= (0x80 >> i);}  // 读取当前位并保存
        MyI2C_W_SCL(0);  // SCL低电平，从机准备下一位
    }
    return Byte;
}




```

> OLED的设置
> 
本次操作使用的128*8Byte的显示屏，在纵向上的y轴自然也分为了8个一Byte长的“页Page”，顾名思义横向x轴有128bit。我们定义左上角为原点。展开后高位在y轴下方，例如写入0x55，则下到上为``0101 0101``。每次写完一位,内部地址指针自动右移一个单位。写到127行后，默认返回设定Page横轴的开头。可以配置寻址模式来实现自动到下一Page。但是你会发现，如果想要实现Y轴任意指定，会涉及到跨Page的情况，需要并行模式下读取GDDRAM的能力，串行模式下，需要缓存数组来中转最后一起写入，这里暂时不表。

>命令表

我们可以通过修改D/C位为0来写入命令给芯片，SSD1306会查找命令表并执行操作，命令可以由一个以上的字节组成。分为``寻址命令，基础命令，滚屏命令，硬件配置命令，时间与驱动命令``五大类。
这里给出三个常用单字节寻址命令
``00~0f，也即0000 xxxx，在页寻址模式下设置起始列地址低位``
``10~1f，和上面这个配合使用，设置高位，防止命令重合``
``B0~B7，1011 0xxx，设置页地址0_to_7``
这时候就有人说了哎哟我怎么这么多命令，上电得写死过去。还好搞硬件就是得会抄，可以参照手册配置发挥一下开源技术。

> ## 代码编写部分

分为两种。第一种不使用缓存区，直接改变GDDRAM，但也不能跨Page操作。第二种使用缓存区和占用一部分RAM空间，换来自定义的刷新和显示。

## 直接写入GDDRAM

这里需要涉及到软件I2C的驱动,参考[[10-3] 软件I2C读写MPU6050_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1th411z7sn?spm_id_from=333.788.videopod.episodes&vd_source=740e28c14abb598f1fe8be2d12484cbb&p=33)

## 汉字显示的实现

这里基于已经实现了使用ascii作为索引可以直接在一个数组里面实现英文和数字字模的存放，参考源码这里略过。汉字不管是在GB规范还是UFT8格式下都需要不止一字节，因此考虑使用结构体来存放：
```
typedef struct
{
	char Index[4];
	uint8_t Data[32];


}ChineseCell_T;
```
我们把这个结构体嵌套进数组里:
```
const ChineseCell_T OLED_CF16X16[] = {
"你"	,	0x00 , 0x00  ...三十二个

}
```
这时可以编写汉字显示脚本了，思路是先拆分字符串为独立的汉字，然后遍历汉字字模，匹配数据，最后显示出来。
```
void OLED_ShowChinese(uint8_t X , uint8_t Page , char *Chinese)
{


char SigleChinese[4] = {0};			///读到的字,这里使用GB标准,两个字节(16bit)一个汉字
uint8_t pChanese = 0; 			////取到第几个字了
uint8_t pIndex;

for (uint8_t i = 0; Chinese[i]!= '\0' ; i++ )
  {
SigleChinese[pChinese] = Chinese[i];
pChinese ++;
if (pChinese >= 3)
	  { 
		pChinese = 0;
		
			for (pIndex = 0; 
			strcmp(OLED_CF16X16[pIndex]->Index , "") != 0 ;
			 pIndex ++ )
			 {
				if (strcmp(OLED_CF16X16[pIndex]->Index , "") == 0 )
			{break;}
			  }
////上面这里我们首先在字模结构体的最下方定义一个""空字符作为没找到结果的默认返回值，然后从输入数据的Chinese数组里面遍历至结束符，将其中的每一位都写入SigleChinese中
		
OLED_ShowImage
(
 X + ((i+1)/3 - 1) * 16 ,
 Page , 
 16 ,
 2 ,
 OLED_CF16X16[pIndex] -> Data
 )


		}
  }
}

```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyOTE4MzY2NzMsLTg4ODQ2MTI2MywxMT
YxNjY0ODUzLDI5NTY0MDM5OSwxNjEwNTYxNzE5LC02NzMwMzAz
OTEsLTE0NTAyODU0NjcsMjI3MDQ0NTQ3LDY2MTkwODI0MCwyMD
kxMDAyOTY4XX0=
-->