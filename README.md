# 基于4G云服务器智能家居实现
通过阿里云平台搭建的控制系统。主要实现利用手机远程控制开发板上的LED、蜂鸣器、直流电机和步进电机，分别对应家居中的灯光、报警、窗帘和风扇等设备的控制。
## 一、流程控制图
![image](https://github.com/user-attachments/assets/7694e96a-d841-445f-81cf-359b47e74c68)
## 二、系统硬件设计
### 2.1 温湿度传感器

![image](https://github.com/user-attachments/assets/ef66663e-cc57-400c-840d-b28b2aea14a7)

### 2.2 光照传感器

![image](https://github.com/user-attachments/assets/0f546afb-d1f5-487e-9f6e-b989866347fd)

### 2.3 步进电机
![image](https://github.com/user-attachments/assets/c5a40feb-ecb2-47dc-96ef-4cc5e25b0f23)

### 2.4 直流电机
![image](https://github.com/user-attachments/assets/1a604adb-676b-43a5-a6df-11cb62c86a0d)

### 2.5 LED
![image](https://github.com/user-attachments/assets/eea8bc11-5951-44c5-bb13-e0d5af95e32f)

![image](https://github.com/user-attachments/assets/06f9109c-201c-4f2f-8b26-5a9de5454d76)

### 2.6 蜂鸣器
![image](https://github.com/user-attachments/assets/b2a363cb-721e-486c-b16a-73a65871f8c0)

![image](https://github.com/user-attachments/assets/96d3d802-688d-4b98-8489-fd8ab3cd0189)

## 三、系统功能设计
### 3.1 初始化
``` Delay_Init(84);		//调用延时初始化
LED_Init();			//调用LED的初始化函数
KEY_Init();			//初始化按键
BEEP_Init();  		//蜂鸣器初始化
Usart1_Init(115200);//串口1初始化
AIR724_Init(115200);//4G模块初始化
Sht3x_Init();		//初始化SHT3x 
BH1750_Init();		//光照模块初始化
RGB_LED_Init();		//初始化RGB
DC_Motor_PWMInit(840,100);	//直流电机初始化
AIR724_Connect();   //4G模块连接阿里云
Step_Motor_Init();	//步进电机初始化
Timer2_Init(10000,8400);	//设置定时器每秒进入一次中断
Step_Motor_Run_Contral(75*10);  //让电机一定最左边
 ```
### 3.2 采集温湿度和光照
```
void AIR724_PubHandle(void)
{
	double  light;
	Sht3x_TypeDef HT = {0};
	BH1750_GetLight(&light);				//获取光照
	Sht3x_Read_TemperatureHumidity(&HT);  //获取温湿度
	//printf("T:%0.2f,H:%0.2f,L:%0.2f \r\n", HT.temperature, HT.humidity,light);
	memset(buf_cmd,0,sizeof(buf_cmd));
	sprintf(buf_cmd,"AT+MPUB=\"%s\",0,0,\"{\\22method\\22:\\22thing.service.property.set\\22,\
			\\22id\\22:\\221711505059\\22,\
	\\22params\\22:{\\22LightLux\\22:%0.2f,\\22CurrentTemperature\\22:%0.1f,\\22CurrentHumidity\\22:%0.1f},\
			\\22version\\22:\\221.0.0\\22}\"\r\n",\
			MQTT_PUB_TOPIC,
			(float)light,
			(float)HT.temperature,
			(float)HT.humidity); 
	Usart2_SendString((u8*)buf_cmd);
```
### 3.3 上传到采集的数据到云平台，每5秒钟上传一次
```
if(timeout % 500 == 0) //上传数据
{
	timeout = 0;
	AIR724_PubHandle();
}
delay_ms(10);
timeout++;
```
### 3.4 当APP发出指令，串口接收
```
if(Usart2.RecFlag == 1) // 标志位为1代表串口收到数据
{
	printf("%s",Usart2.RxBuff);	//将数据打印到PC端，这就是回显			
	DataSeparation(Usart2.RxBuff);	//调用分离函数
	AIR724_Execute();				//调用执行函数
	Usart2.RecLen = 0;	//数据下标，也要清除
	Usart2.RecFlag = 0;	//清除这一次的标志位，下一次数据可以正常获取
	memset(Usart2.RxBuff,0,sizeof(Usart2.RxBuff));
}
```
### 3.5 在执行函数中的具体操作
```
void AIR724_Execute(void)
{
 if(strcmp((char *)logo,"Buzzer")==0)      //如果是蜂鸣器
	{
		if(strcmp((char *)value,"1")==0) BEEP = 1;	//开
		else if(strcmp((char *)value,"0")==0) BEEP = 0;	//关
	}
	if(strcmp((char *)logo,"LightSwitch")==0)         //如果是主灯
	{
		if(strcmp((char *)value,"1")==0) LED1 = 0;	//开
		else if(strcmp((char *)value,"0")==0) LED1 = 1;	//关
	}
	if(strcmp((char *)logo,"NightLightSwitch")==0) //如果是夜灯
	{
		if(strcmp((char *)value,"1")==0) LED2 = 0;	//开
		else if(strcmp((char *)value,"0")==0) LED2 = 1;	//关
	}
	if(strcmp((char *)logo,"StepMotor")==0) //如果是步进电机
	{
		if(strcmp((char *)value,"0")==0) open_value=0;
		if(strcmp((char *)value,"1")==0) open_value=75*1;
		if(strcmp((char *)value,"2")==0) open_value=75*2;
		if(strcmp((char *)value,"3")==0) open_value=75*3;
		if(strcmp((char *)value,"4")==0) open_value=75*4;
		if(strcmp((char *)value,"5")==0) open_value=75*5;
		if(strcmp((char *)value,"6")==0) open_value=75*6;
		if(strcmp((char *)value,"7")==0) open_value=75*7;
		if(strcmp((char *)value,"8")==0) open_value=75*8;
		if(strcmp((char *)value,"9")==0) open_value=75*9;
		if(strcmp((char *)value,"10")==0) open_value=75*10;
		if(open_value!=position)
		{
			if(open_value>position)	distance=-(open_value-position);
			if(open_value<position) distance=position-open_value;
			Step_Motor_Run_Contral(distance);
		}
		position=open_value;
		open_value=0;
	}
	if(strcmp((char *)logo,"KitchenVentilator_MotorStall")==0) //如果是直流电机
	{
		if(strcmp((char *)value,"0")==0) DC_Motor_Ctrl(0);
		if(strcmp((char *)value,"1")==0) DC_Motor_Ctrl(10);
		if(strcmp((char *)value,"2")==0) DC_Motor_Ctrl(50);
		if(strcmp((char *)value,"3")==0) DC_Motor_Ctrl(100);
	}
	if(strcmp((char *)logo,"AlarmSwitch")==0)      //如果是报警灯
	{
		if(strcmp((char *)value,"1")==0) LED_State = 1;	//开
		else if(strcmp((char *)value,"0")==0) LED_State = 0;	//关
	}
}
```
### 3.6 报警灯的实现
```
if(timeout%10==0)
{
	if(LED_State==1) LED2=!LED2;
	else if(LED_State==0) LED2=1;
}
```
### 3.7 设置当光照达到1300自动关闭窗帘
```
if(light>1300)
	{
		Step_Motor_Run_Contral(780);
		st_f=1;
	}
	else
	{           
		  if(st_f==1)
			{
				printf("3");
				Step_Motor_Run_Contral(-780);
				st_f=0;
			}
	}
```
### 3.8 设置当温度达到25度步进电机转动
```
if(HT.temperature>25)
	{
		DC_Motor_Ctrl(50); 
	}
	else
		DC_Motor_Ctrl(0);
}
```
## 四、手机端界面
![image](https://github.com/user-attachments/assets/44556710-4a21-403d-b348-3e8e4d667bf2)

