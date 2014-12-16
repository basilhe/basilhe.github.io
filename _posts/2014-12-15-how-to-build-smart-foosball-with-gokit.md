---
layout: post
title: "How To Build Smart FoosBall with GoKit"
description: ""
category: iot
tags: [foosball,gokit]
---
{% include JB/setup %}

{{toc}}

# 缘起

。。。 了解、申请、拿到 GoKit，激动 -> 灵感 -> 动手改造桌上足球 。。。

# 数据点定义

主要解决以下问题：

* 红蓝双方的进球自动探测、自动计数(goal)
* 实时显示当前双方进球数（blue_goals, red_goals）
* 实时显示当前双方胜负局数(blue_score, red_score)
* 进球后标记进球位置，默认为进攻方前锋进球，可修改记录后卫进球或者对方前锋、后卫乌龙球(goal_member)
* 清除当前计分，全新开始(game_control:start)
* 双方交换场地进行下一局(exchange_position，通过按键操作)
* 交换前锋、后卫位置(exchange_member，通过按键操作)
* 取消进球（game_control:cancel_red_goal&cancel_blue_goal）

最终定义如下(数值类型分辨率均为 1，增量均为 0)：

| 标识名  | 显示名称 | 读写类型 | 数据类型 | 取值范围 |
| ------------ | ------------- | ------------ | ------------ | ------------ |
| blue_goals        | 蓝方进球数   | 只读 | uint8 | 0 - 10 |
| red_goals         | 红方进球数   | 只读 | uint8 | 0 - 10|
| blue_score        | 蓝方赢局数   | 只读 | uint8 | 0 - 5 |
| red_score         | 红方赢局数   | 只读 | uint8 | 0 - 5 |
| goal              | 进球        | 只读 | 枚举  | none,red,blue |
| goal_member       | 进球队员     | 只读 | 枚举  | none,red_van,red_rear,blue_van,blue_rear |
| exchange_member   | 交换球员位置 | 只读 | 枚举  | none,red,blue |
| exchange_position | 双方交换场地 | 只读 | 布尔值 |
| game_id           | 场次         | 可写 | uint8| 0 - 254 |
| game_control      | 游戏控制     | 可写 | 枚举  | going,start,cancel_red_goal,cancel_blue_goal |


## 步骤

* 使用开发者账号登录机智云开发者网站 http://site.gizwits.com
* 点击新建设备接入按钮
* 输入设备名称和其他信息，点击添加按钮并一直选择下一步直到完成
![image](/assets/images/foosball/new_product.png)
* 完成后，选择该产品，并在左边菜单项中数据点进入数据点编辑页面，再使用右边的新建数据点按钮按个添加数据点
* 编辑完成之后，选择左侧菜单项“产品开发资源”，滚动页面到“MCU 开发资源”部分，点击“机智云接入串口通信协议文档”下载 MCU 开发文档，就可以着手开始硬件开发了

# 硬件开发

## 硬件准备

* 机智云免费申请的 GoKit 一块
* 3路 IO 口的四位数据管模块一只，用于比分显示
* ST188 反射式红外光电传感器(与 GoKit 内置红外传感器型号一致)两个，用于进球检测
* 40P杜邦线若干
* 面包板和电阻若干
![image](/assets/images/foosball/hardware.png)

## 开发资料和环境准备

* 数据点部分下载的“机智云接入串口通信协议文档”
* [GoKit 原理图](http://club.gizwits.com/forum.php?mod=viewthread&tid=62&extra=page%3D1)
* [Keil Microcontroller Development Kit Version 4.72](http://www.keil.com/)
* [4位串行595数码管资料.zip](http://pan.baidu.com/s/1pJyoFLh)
* SEGGER J-Link 一条，接口 20PIN JTag
* [GoKit MCU 源码](https://github.com/gizwits/gokit-mcu)

## 硬件连接

根据《GoKit 原理图》，GoKit 板上的 P5 扩展接口是提供给 MCU 的 IO 引脚，包括 PA4-PA7、PB10 和 PB11 共6个引脚，如图所示：

![image](/assets/images/foosball/p5.png)

先将4位数码管的 DIO、RCLK、SCLK 分别连接到 PA6、PA7、PA5 口并连接 VCC 和 GND，如下图：

![image](/assets/images/foosball/led.png)

然后按照如下电脑连接 ST188 在面包板上：

![image](/assets/images/foosball/ir.png) 

并和 GoKit 相连，红蓝红外探测器的 AOUT 分别连接 P5 扩展接口中的 PA4 和 PB10 引脚：

![image](/assets/images/foosball/whole.png) 

最后将 JLink 连接到 GoKit 的 20PIN 插槽上即可。

## 上代码

### 项目准备

首先克隆 GoKit 的 MCU 代码到本地 smartfoosball-mcu 目录：

	git clone https://github.com/gizwits/gokit-mcu smartfoosball-mcu
	
使用 MDK 打开子目录 MDK_Project 下的 Project 项目，配置 "Target 1" 的 Debug 和 Utilities 选项中的设备连接方式为 “J-LINK / J-Trace Cortex”。

![image](/assets/images/foosball/debug.png) 

![image](/assets/images/foosball/options.png) 

使用 JLink 连接设备的时候如果提示需要驱动的话可以到 “C:\Keil\ARM\Segger” 目录下寻找对应的 JLink 驱动程序。

### 增加 LED 模块封装

在项目子目录 User\Hal_Driver 中增加 hal_led.h 和 hal_led.c 文件，并增加到项目的 user 分组下面

![image](/assets/images/foosball/hal_led.png) 

参照“4位串行595数码管资料”和 GoKit 源码中的 hal_rgb_led 模块，定义 hal_led.h 如下：

	#ifndef _HAL_LED4_H
	#define _HAL_LED4_H

	#include <stdio.h>
	#include <stm32f10x.h>
	#include "protocol.h"

	#define DIO_LOW GPIO_ResetBits(GPIOA,GPIO_Pin_6) // 通过 PA6 发送低电平 给 DIO 
	#define DIO_HIGH GPIO_SetBits(GPIOA,GPIO_Pin_6) // 通过 PA6 发送高电平 给 DIO	
	#define RCLK_LOW GPIO_ResetBits(GPIOA,GPIO_Pin_7) // 通过 PA7 发送低电平 给 RCLK
	#define RCLK_HIGH GPIO_SetBits(GPIOA,GPIO_Pin_7)  // 通过 PA7 发送高电平 给 RCLK

	#define SCLK_LOW GPIO_ResetBits(GPIOA,GPIO_Pin_5) // 通过 PA5 发送低电平 给 SCLK
	#define SCLK_HIGH GPIO_SetBits(GPIOA,GPIO_Pin_5)  // 通过 PA5 发送高电平 给 RCLK

	void LED4_Init(void); // 初始化 IO 口信息
	
	// 输出 4 个 数字到 LED 显示屏，取值范围 从 0 到 15(F)
	void LED4_Display(uint8_t d1, uint8_t d2, uint8_t d3, uint8_t d4);

	#endif /*_HAL_LED4_H*/
	
并实现如下：

	#include "hal_led.h"

	//-----------------------------------------------------------------------------
	// 函数原形定义
	#define uchar unsigned char

	void LED_OUT(uchar X);				// LED单字节串行移位函数
	unsigned char LED_0F[];		// LED字模表

	void LED_OUT(uchar X)
	{
		uchar i;
		for(i=8;i>=1;i--)
		{
			if (X&0x80) DIO_HIGH; else DIO_LOW;
			X<<=1;
			SCLK_LOW;
			SCLK_HIGH;
		}
	}

	unsigned char LED_0F[] = 
	{// 0	 1	  2	   3	4	 5	  6	   7	8	 9	  A	   b	C    d	  E    F    -
		0xC0,0xF9,0xA4,0xB0,0x99,0x92,0x82,0xF8,0x80,0x90,0x8C,0xBF,0xC6,0xA1,0x86,0xFF,0xbf
	};

	void LED4_Init(void)
	{
		GPIO_InitTypeDef GPIO_InitStruct;
		RCC_APB2PeriphClockCmd( RCC_APB2Periph_GPIOA,ENABLE);
		GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;
		GPIO_InitStruct.GPIO_Pin = GPIO_Pin_6|GPIO_Pin_7|GPIO_Pin_5;
		GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
		GPIO_Init(GPIOA, &GPIO_InitStruct);
	}

	void LED4_Display(uint8_t d1, uint8_t d2, uint8_t d3, uint8_t d4)
	{
		unsigned char *led_table;          // 查表指针
		uchar i;
		//显示第1位
		led_table = LED_0F + d1;
		i = *led_table;

		LED_OUT(i);			
		LED_OUT(0x01);		

		RCLK_LOW;
		RCLK_HIGH;
		//显示第2位
		led_table = LED_0F + d2;
		i = *led_table;

		LED_OUT(i);		
		LED_OUT(0x02);		

		RCLK_LOW;
		RCLK_HIGH;
		//显示第3位
		led_table = LED_0F + d3;
		i = *led_table;

		LED_OUT(i);			
		LED_OUT(0x04);	

		RCLK_LOW;
		RCLK_HIGH;
		//显示第4位
		led_table = LED_0F + d4;
		i = *led_table;

		LED_OUT(i);			
		LED_OUT(0x08);		

		RCLK_LOW;
		RCLK_HIGH;
	}

### 改造红外探测模块

GoKit 的红外探测模块 hal_infrared 只支持板上内置的一个红外探测器，而我们新增了两个，为了重用之前的红外探测模块代码，做了如下改动：

	#ifndef _HAL_INFRARED_H
	#define _HAL_INFRARED_H

	#include <stdio.h>
	#include <stm32f10x.h>
	#include "delay.h"
	#include "hal_uart.h"

	#define IR_BOARD 0 // 增加了三个的红外设备标识，此为 GoKit 内置在板上的
	#define IR_BLUE 1 // 这个是蓝方的进球探测器
	#define IR_RED 2 // 这个是红方的进球探测器

	// 以下函数全部增加了 id 参数，取值范围如上, IR_BOARD | IR_BLUE | IR_RED
	void IR_EXTI_Init(uint8_t id);
	void IR_TIM_Init(uint8_t id);
	void IR_TIM_Init(uint8_t id);

	void IR_Init(uint8_t id);
	void IR_Handle(uint8_t id);
	void IR_GPIO_Init(uint8_t id);
	#endif /*_HAL_INFRARED_H*/

在 hal_infrared.c 中，我们也针对不同的设备设置了不同的 IO 口和中断回调函数。
由于本人对回调不是很熟悉，暂时只能同时处理两路回调，IR_BOARD 和 IR_BLUE 使用了同一中断 EXTI15_10_IRQn，所以这两者只能起用一个。

回调函数在 user/stm32f10x_it.c 中定义：

	void EXTI15_10_IRQHandler(void) 
	{
		EXTI->EMR &= (uint32_t)~(1<<1);   									//屏蔽中断事件

		while(EXTI_GetITStatus(EXTI_Line10)!= RESET ) 
		{		
			if(GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_10))
			{
				if ((l_timestamp - last_timestamp) > 50) { // 增加延时处理，避免重复计分
					handleGoal(SIDE_BLUE); // 蓝方进球
					last_timestamp = l_timestamp;
				}
			}
			else
			{		
				last_timestamp = l_timestamp;
			}	
		
			EXTI_ClearITPendingBit(EXTI_Line10);
		}	
		
		EXTI->EMR |= (uint32_t)(1<<1);  										//开启中断事件  
	}

	void EXTI4_IRQHandler(void) 
	{
		EXTI->EMR &= (uint32_t)~(1<<1);   									//屏蔽中断事件

		while(EXTI_GetITStatus(EXTI_Line4)!= RESET ) 
		{		
			if(GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_4))
			{
				if ((l_timestamp - last_timestamp) > 50) {
					handleGoal(SIDE_RED); // 红方进球
					last_timestamp = l_timestamp;
				}
			}
			else
			{		
				last_timestamp = l_timestamp;
			}	
		
			EXTI_ClearITPendingBit(EXTI_Line4);
		}	
		
		EXTI->EMR |= (uint32_t)(1<<1);  										//开启中断事件  
	}
	
### 初始化模块

在 main.c 主函数 main 中，引入相关头文件，并增加初始化如下：

	Motor_Init();	
	RGB_LED_Init();
	DHT11_Init();
	//IR_Init(IR_BOARD); // 关闭主板红外模块
	
	LED4_Init(); // 初始化数码管模块
	//LED4_Display(0, 1, 2, 3);
	
	McuStatusInit();
	
	IR_Init(IR_BLUE); // 初始化蓝方进球红外探测器
	IR_Init(IR_RED); // 初始化红方进球红外探测器

### 处理数据点协议

根据下载的“机智云接入串口通信协议文档”，数据上报和下发控制的数据包协议分别如下：

#### 数据上报

| 字节序 | 位序 | 数据内容 | 说明 |
| ------------ | ------------- | ------------ | ------------ |
| byte0 | bit7 bit6 ... bit1 bit0 | 0b00000011 | 游戏控制,值为3:字段bit2 ~ bit0,字段值为0b11; |
| byte1 | | 0xFE | 场次字段值为254; 实际值计算公式y=1.000000*x+(0.000000) x最小值为0,最大值为254
byte2 | bit7 bit6 ... bit1 bit0 | 0b10100101 |双方交换场地,值为true:字段bit0,字段值为0b1; 进球,值为2:字段bit2 ~ bit1,字段值为0b10; 进球队员,值为4:字段bit5 ~ bit3,字段值为0b100; 交换球员位置,值为2:字段bit7 ~ bit6,字段值为0b10; |
| byte3 | | 0x0A | 蓝方进球数字段值为10; 实际值计算公式y=1.000000*x+(0.000000) x最小值为0,最大值为10 |
| byte4 | | 0x0A | 红方进球数字段值为10; 实际值计算公式y=1.000000*x+(0.000000) x最小值为0,最大值为10 |
| byte5 | | 0x05 | 蓝方赢局数字段值为5; 实际值计算公式y=1.000000*x+(0.000000) x最小值为0,最大值为5 |
| byte6 | | 0x05 | 红方赢局数字段值为5; 实际值计算公式y=1.000000*x+(0.000000) x最小值为0,最大值为5 |

#### 下发控制

| 字节序 | 位序 | 数据内容 | 说明 |
| ------------ | ------------- | ------------ | ------------ |
| byte0 | bit7 bit6 ... bit1 bit0 | 0b00000011 | 游戏控制,值为6:字段bit2 ~ bit0,字段值为0b11; |
| byte1 | | 0xFE | 场次字段值为254; 实际值计算公式y=1.000000*x+(0.000000) x最小值为0,最大值为254 |

#### 修改产品标识

每个产品都有自己的标识，如果你定义了自己的产品，则需要修改产品标识，修改 protocol.h 中宏定义 PRODUCT_KEY 的值为机智云网站“产品信息”页面中的标识码字符串内容即可

	#define		PRODUCT_KEY				  "ef992d473159465ba9d70d4d1a14aa39"
	
#### 定义只读和可写结构体

GoKit 源码中分别针对以上上报和下发的数据结构定义了 _status_readonly 和 _status_writable 结构体，下发控制时 _status_writable 会收到下发的数据，而上报的数据则是 _status_writable 加上_status_readonly。

根据协议，在 protocol.h 中修改 _status_writable 和 _status_readonly 如下：

	__packed	struct	_status_writable
	{
		uint8_t							game_control; // bit 0=going,1=start,2=cancel_red_goal,3=cancel_blue_goal
		uint8_t							game_id;
	};

	__packed	struct	_status_readonly
	{
		uint8_t							actions; // exchange position=0, goal=1-2, goal_member=3-5, exchange_member=6-7
		uint8_t							blue_goals;
		uint8_t							red_goals;
		uint8_t							blue_score;
		uint8_t							red_score;
	};

以上结构体在程序中对应的变量名分别为 m_w2m_controlMcu.status_w 和 m_w2m_controlMcu.status_r

#### 下发控制入口

MCU 收到 Wi-Fi 的控制指令，编程入口在 protocol.c 的 CmdSendMcuP0 函数中，根据协议修改如下：

	/*******************************************************************************
	* Function Name  : CmdSendMcuP0
	* Description    : mcu接收到wifi的控制命令，此部分是需要mcu开发者重点实现的
	* Input          : buf：串口接收缓冲区地址
	* Output         : None
	* Return         : None
	* Attention		   : None
	*******************************************************************************/
	void	CmdSendMcuP0(uint8_t *buf)
	{
		uint8_t		tmp_cmd_buf;
		
		if(buf == NULL) return ;
		
		memcpy(&m_w2m_controlMcu, buf, sizeof(w2m_controlMcu));                                                                                                                                                                                            
		
		//上报状态
		if(m_w2m_controlMcu.sub_cmd == SUB_CMD_REQUIRE_STATUS) ReportStatus(REQUEST_STATUS);
		
		//控制命令，操作字段顺序依次是： game_control, game_id
		if(m_w2m_controlMcu.sub_cmd == SUB_CMD_CONTROL_MCU){
			//先回复确认，表示收到合法的控制命令了
			SendCommonCmd(CMD_SEND_MCU_P0_ACK, m_w2m_controlMcu.head_part.sn);
			
			//控制命令标志按照协议表明哪个操作字段有效（对应的位为1）want to control game 
			if((m_w2m_controlMcu.cmd_tag & 0x01) == 0x01)
			{
				//0 ongoing, 1: start, 2: cancel_red_goal, 3: cancel_blue_goal
				uint8_t control = m_w2m_controlMcu.status_w.game_control & 0x07;
				if(control == 0x01) // start 
				{
					//控制 game id
					if((m_w2m_controlMcu.cmd_tag & 0x02) == 0x02)
					{
						resetGame(m_w2m_controlMcu.status_w.game_id);
					} else {
						resetGame(m_m2w_mcuStatus.status_w.game_id + 1);
					}
				} if (control == CANCEL_RED_GOAL || control == CANCEL_BLUE_GOAL) {
					cancelBall(control);
				}
				m_m2w_mcuStatus.status_w.game_control = GAME_STATUS_GOING;
			}

			ReportStatus(REPORT_STATUS);
		}
	} 

#### 状态上报

MCU 通常会在以下情况调用 ReportStatus(tag）上报设备状态：

* 收到状态查询指令或者控制指令，如上 CmdSendMcuP0 中会调用
* CheckStatus(void) 方法发现设备状态变化时上报
* CheckStatus(void) 方法固定周期定时上报

主要实现在 ReportStatus(tag) 方法中，需要注意的是如果你的数值长度是 16 位或以上，在上报时需要调用 exchangeBytes(value) 方法转换为大端序

#### 业务逻辑实现

桌上足球的业务逻辑实现入口主要有：

*  protocol.c 的下发控制入口 CmdSendMcuP0
*  stm32f10x_it.c 红外探测器终端回调 EXTI15_10_IRQHandler 和 EXTI4_IRQHandler
*  hal_key.c 的 KeyHandle(void) 长短按键回调
*  protocol.c 的 CheckStatus(void) 中，更新数码管显示

相关代码请参见以上入口，按键相关功能如下：

* 短按 KEY1, KEY2, KEY3, KEY4，表示标记进球队员， 依次是红方前锋，红方后卫，蓝方前锋，蓝方后卫
* 长按 KEY3，双方交换场地，比分清零重新开下一局
* 长按 KEY4,  比分清零，重新开局。
* 当计分板上有分数时
  * 长按 KEY1 表示取消红方进球
  * 长按 KEY2 表示取消蓝方进球
* 手机控制
  * 游戏控制设置为开始，比分清零，重新开局
  * 设置为取消红方进球，则取消红方进球数一个
  * 设置为取消蓝方进球，同理取消蓝方进球数一个，如果进球数为零则不做任何改动
  * 通过修改 game_id 可以修改游戏场次

## 联调安装

到机智云网站产品开发资源中即可下载 iOS 或 Android 的 Demo App 进行联调

![image](/assets/images/foosball/demo.png) 

结果示意图

![image](/assets/images/foosball/onboard.png) 

