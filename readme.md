# 光电门测速
## 硬件
开发板：大疆C板，STM32F407\
接线：找到两个 IO 口，用于接收光电门传来的信号，此光电门是传来上升沿信号，所以配置 IO 口时选择***上升沿触发，同时选择下拉***（注意板子上面的 IO 口及其抽象，有些 IO 口虽然可以配置成`EXTI`，但是是进入不了中断函数的，反正很抽象 T_T ）\
此次测速，我选择的是`PF0`和`PF1`
## 软件
### 配置定时器
$$f=\frac{Tclk}{(PSC+1)(ARR+1)}$$
$$Tclk=84MHz$$
$$PSC=84-1$$
$$ARR=100-1$$
于是我们得到，每隔 $10^{-4} s$ 进入一次中断
```C++
Warning：
1.  在写代码时，要在main.c里面开启中断（忘记真的想死 sTs）
	HAL_TIM_Base_Start_IT(&htim6);
2.  Tclk指的是时钟挂载 APB 总线的频率
	Hclk是系统时钟，注意区分清楚
```
然后哩，进到中断里面去写个flag让它计数，然后我们就可以得到时间辣！\
在`xx_it.c`里面找到定时器中断函数
### 配置 IO 口中断
在`xx_it.c`介个文件里面哩
```C++
void EXTI0_IRQHandler(void)
{
    /* USER CODE BEGIN EXTI0_IRQn 0 */
	a = flag;
	// 将flag的格式转化为字符串
	sprintf(buffer, "a=%d\r\n", a);
	HAL_UART_Transmit(&huart6, (uint8_t *)buffer, strlen(buffer), HAL_MAX_DELAY);
	//printf("a=%d\r\n", a);

    /* USER CODE END EXTI0_IRQn 0 */
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
    /* USER CODE BEGIN EXTI0_IRQn 1 */

    /* USER CODE END EXTI0_IRQn 1 */
}
```
```C++
void EXTI1_IRQHandler(void)
{
    /* USER CODE BEGIN EXTI1_IRQn 0 */
	b = flag;
	speed = 10000 * 0.1 / (float)(b - a);

	// 将flag的格式转化为字符串
	sprintf(buffer, "b=%d, speed=%f\r\n", b, speed);
	HAL_UART_Transmit(&huart6, (uint8_t *)buffer, strlen(buffer), HAL_MAX_DELAY);
	//printf("a=%d, speed=%f\r\n", a, speed);

    /* USER CODE END EXTI1_IRQn 0 */
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_1);
    /* USER CODE BEGIN EXTI1_IRQn 1 */

    /* USER CODE END EXTI1_IRQn 1 */
}
此处 speed 的计算就是，将光电门两次的计数相减，然后得到时间，距离除以时间就能得到速度啦！
```

遇到的问题：
在使用usart给电脑发数据的时候
* 最初采用重定向`printf()`，但是发现，这样单片机会卡死，debug的时候必须在第三次全速运行时才能跑起来，正常跑也接收不到数据
* 后来采用`HAL_UART_Transmit()`，这样只需要将我们的数值类型转换一下，就可以通过这个函数发出去辣！