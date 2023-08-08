# PWM控制电机转速与编码器读取


## STM32CubeMX配置

The configuration of timer is similar with the fomer chapters, and you need to configure **timer** in cubeMX.
- C芯片：STM32F103RCT6
- C设置RCC：设置高速外部时钟HSE，选择外部时钟源
- LED配置：通过观察小灯亮灭判断是否进入定时器中断
- USART1配置
- PWM配置
> 使用定时器1通道1和通道4，频率10KHz
> **PWM频率计算公式f = 定时器时钟频率/【（Period+1）*（Prescaler）】= 72M/（7199 + 1）*1 = 10k HZ**
> 占空比计算公式 = P=pulse/per *100%
- C定时器中断配置：使用定时器2（TIM2），周期设为10ms，即10ms进一次定时器中断；打开更新定时器中断
- STM32自带编码器配置，使用定时器4（TIM4_CH1和TIM4_CH2），打开更新定时器中断
- C中断优先级配置：因为编码器中断要发生在定时10ms中断内，故编码器中断的抢占优先级要大于定时10ms
- C配置时钟:F1系列芯片系统时钟为72MHzs
- C输出文件、创建工程文件

## main.c源代码
/* USER CODE BEGIN Includes */
#include "stdio.h"
#include "math.h"
/* USER CODE END Includes */
/* USER CODE BEGIN PV */
unsigned int MotorSpeed;  // 电机当前速度数值，从编码器中获取
int MotorOutput;		  // 电机输出
/* USER CODE END PV */
/* USER CODE BEGIN 0 */
int fputc(int ch, FILE *p)
{
	while(!(USART1->SR & (1<<7)));    //检查SR标志位是否清零
	USART1->DR = ch;                  //将内容赋给DR位输出
	
	return ch;
}
/* USER CODE END 0 */
  /* USER CODE BEGIN 2 */
  HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);	    // TIM1_CH1(pwm)
  HAL_TIM_Encoder_Start(&htim4, TIM_CHANNEL_1); // 开启编码器A
  HAL_TIM_Encoder_Start(&htim4, TIM_CHANNEL_2); // 开启编码器B	
  HAL_TIM_Base_Start_IT(&htim2);                // 使能定时器2中断
  /* USER CODE END 2 */
  /* USER CODE BEGIN 4 */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    static unsigned char i = 0;
    if (htim == (&htim2))
    {
        //获取电机速度
        MotorSpeed = (short)(__HAL_TIM_GET_COUNTER(&htim4)/18);   
        // TIM4计数器获得电机脉冲，该电机在10ms采样的脉冲/18则为实际转速的rpm
        __HAL_TIM_SET_COUNTER(&htim4,0);  // 计数器清零
        
      //2.将占空比导入至电机控制函数
        MotorOutput=3600; // 3600即为50%的占空比
        __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, MotorOutput);
        MotorOutput=MotorOutput*100/7200; 
        // 占空比（按最高100%计算，串口打印）
        i++;
        if(i>100)
        {
          // 打印定时器4的计数值，short（-32768——32767）
          printf("Encoder = %d moto = %d \r\n",MotorSpeed,MotorOutput);	
          i=0;
        }
    }
}
/* USER CODE END 4 */

