# 1. 接线
以RoboMaster C型开发板和天地飞ET08A遥控器为例
![[Pasted image 20260920143021.png|651]]
我们需要将遥控器接收机的SBUS接口接到C板上的DBUS接口，注意线序一定要接对。
# 2. CubeMX配置
由手册得知，单片机通过UART3串口接受遥控器的数据，我们需要在CubeMX中打开UART3的RX引脚，为PC11，并将波特率改为100kbps，开启串口DMA
![[Pasted image 20260920143323.png]]

![[PixPin_2026-09-20_15-13-26.png|623]]
![[PixPin_2026-09-20_15-14-14.png|397]]
只需开启接收的DMA通道即可
![[PixPin_2026-09-20_15-14-40.png|542]]
配置完后生成代码
# 3. 代码移植
参考2026赛季无人机的云台代码，需要将`/User/MCUDriver/REMOTEIO`下的文件粘贴到自己的工程，并删除代码中自己工程中不存在的头文件的引用
![[PixPin_2026-09-20_15-21-13.png]]

在`main.cpp`里引用`remoteio.hpp`，并在主函数的初始化里面调用函数
```cpp
REMOTEIO_Init(sbus_rx_buf[0], sbus_rx_buf[1], SBUS_RX_BUF_NUM);
```
在`/Core/Src/stm32f4xx_it.cpp`文件中引用找到
```cpp
/**  
  * @brief This function handles USART3 global interrupt.  */void USART3_IRQHandler(void)  
{  
  /* USER CODE BEGIN USART3_IRQn 0 */  
  
  /* USER CODE END USART3_IRQn 0 */  
  HAL_UART_IRQHandler(&huart3);  
  /* USER CODE BEGIN USART3_IRQn 1 */  

  /* USER CODE END USART3_IRQn 1 */  
}
```
并在其中插入遥控器接收数据处理函数
```cpp
  REMOTE_UartIrqHandler();  
```
如下
![[PixPin_2026-09-20_15-27-06.png|633]]

然后可以在`subs_to_rc()`函数中打印，看是否成功接收并正确解码遥控器的数据，例如
![[PixPin_2026-09-20_15-28-55.png]]

通过调用`RC_ctrl_t`结构体的实例`rc_ctrl`来使用遥控器的参数
```cpp
typedef struct  
{  
    struct  
    {  
        /* 遥控器摇杆和拨杆 */        
        int16_t ch[8];  
        char s[4];  
    } rc;  
  
    SBUS_Status_t sbus_status;  
    uint16_t      rc_live_cnt_WFLY;  /*!< 在线倒计时，由 REMOTEIO_UpdateStatus() 递减 */
} RC_ctrl_t;
```
遥控器共有8个通道，四个摇杆和四个拨杆，解码后的数值为-1024~+1024，回中值为0

如果回中值不为0一般有两个原因：
1. 不小心动了遥控器上的微调量，需要保证为0，如果不为0可以通过调整T1 T2 T3 T4来修改
![[PixPin_2026-09-20_15-36-16.png]]

2. 遥控器有零漂，手动调整头文件中的宏定义参数修改
![[PixPin_2026-09-20_15-37-43.png]]