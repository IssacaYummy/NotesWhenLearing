# 1. 接线
以RoboMaster C型开发板为例
![[Pasted image 20260920143021.png|651]]
我们需要将遥控器接收机的SBUS接口接到C板上的DBUS接口，注意线序一定要接对。
# 2. CubeMX配置
由手册得知，单片机通过UART3串口接受遥控器的数据，我们需要在CubeMX中打开UART3的RX引脚，为PC11，并将波特率改为100kbps，开启串口DMA
![[Pasted image 20260920143323.png]]


