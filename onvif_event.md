# ONVIF的事件分析
## 1. 注册
### 变量

1. 全局变量 *onvif_EventService_g*  
`onvif_EventService_St onvif_EventService_g;`
2. 全局变量 *alarm_mng_t *ALARM_MANAGER*  
3. 全局变量，链表头*savedmsgList*，链表类型为ONVIF_NewEvent_Msg_ST

### 类型

1. *onvif_EventService_g* 变量的类型声明
```c
typedef struct
{
	unsigned int magic; // =0xf3ab3254
	int AlarmSubcribeID; // =
	onvif_PropertyMap_St propertymap; // => 类型1.1
	struct list_head EventSubcriber; //未使用
}onvif_EventService_St;    
// 类型1.1 ,onvif_PropertyMap_St
typedef struct
{
	struct list_head onvif_PropertyList;//链表，链表类型onvif_PropertyList_St ==>
	int propertylist_update;
	char *propertybuf; //分配内存，根据事件的主类型进行分类，构建xml节点，最后格式输出为字符串
	int propertybufsize; //propertybuf的长度
}onvif_PropertyMap_St;
typedef struct onvif_PropertyList_St
{
	struct list_head list;
	xmlNodePtr property; //分配内存，事件完整的xml描述节点
}onvif_PropertyList_St;
```  

2. *ALARM_MANAGER*的类型说明
```c
typedef struct alarm_mng
{
	struct list_head  alarmSourceList;//报警源链表,类型为alarm_source_t
	UINT32 alarmSourceMaxID;//报警源id
	struct list_head  subscriberList;//报警响应链表，类型为subscriberMngList_t
	UINT32 subscriberMaxID;//报警响应方式id
	struct list_head  alarmquene;//报警输入链表，类型alarmin_t
	pthread_mutex_t lock_alarmMng;//互斥锁
	sem_t alarmsem;//信号量
}alarm_mng_t;
//报警源信息.IO/MOTION/NET/OBS/Videoloss
typedef struct alarm_source
{
	struct list_head  list;
	UINT32 alarmID;//报警源id
	alarmState_E state;//报警状态，未使用，未初始化
	UINT32 priority;//报警优先级
	struct timespec monotime;//系统启动时间
	struct timeval alarmtime;//报警时间
	char *topicExpress;//分配内存
	struct list_head  subscriberList;//订阅者id链表，类型为subscriberIDList_t，分配内存
	AlarmInputMode_E InputMode;//未使用，未初始化
	xmlDocPtr msg;
}alarm_source_t;
//订阅者id信息，对应响应方式id信息
typedef struct subscriberIDList
{
	struct list_head  list;
	UINT32 subscriberID;//响应方式id
}subscriberIDList_t;
typedef struct subscriberMngList
{
	struct list_head  list;
	char *name;//响应方式名字，分配内存
	void *arg;//回调函数私有参数
	alarmActionfun  action_fun;//回调函数
	void *arg2;//回调参数
	alarmActionfun2  action2_fun;//回调函数
	struct list_head  topicExpressList;//规则表达式链表
	UINT32 subscriberID;//响应方式id
	AlarmPfmMode_E pfmmode;//执行方式，关闭（默认）/每天/自定义
	UINT32 pfmday[7];//执行工作日选择
	UINT32 starttime;//hour*60+minute，=0
	UINT32 endtime;//hour*60+minute,=0
}subscriberMngList_t;
typedef struct alarmin
{
	struct list_head  list;
	UINT32 alarmID;//报警id
	AlarmInputMode_E InputMode;//报警类型，simple/withMsg
	alarmState_E state;//报警状态
	UINT32 priority;//报警优先级，优先级越高（大），越快处理
	struct timespec monotime;//系统启动时间
	struct timeval alarmtime;//报警输入时间
	float zoneValue;//与北京时区的偏差，以小时为单位，未初始化
	UINT32 daylight_saving;//夏令时，未初始化
	xmlDocPtr msg; //分配内存
}alarmin_t;

```  
3. *savedmsgList*的类型
```c
typedef struct
{
	struct list_head list;
	char topic[100];
	struct timespec monotime; //未使用，未初始化
	struct timeval alarmtime; //未使用，未初始化
	int msgstateflag; //0 is "Changed", 1 is "Initialized"，未初始化
	xmlNodePtr msg; // =NULL
```


### 方法
1. GetXmlDocFromFile() //根据xml文件注册相应的事件   
> AlarmSourceRegister_API 根据事件的属性alarmSourceID注册事件源  
 ONVIF_Add_EventProperty_API 保存事件的完xml节点描述到变量onvif_EventService_g  
2. ONVIF_EventServiceWithXml_Init_2
> SubscriberRegister2_API 注册事件组，名字为onvifWithMsg，回调函     数Onvif_EventNotifyAction_Callback，参数为onvif_EventService_g变量   
subscribeAlarmSource_FUN // 在事件源中添加事件组ID链表
启动线程ONVIF_NewEventNotify_Thread

### 事件xml描述

```xml
	<wstop:eventset>
		<alarmSourceID>/majortype:infextend/minortype:heartbeat</alarmSourceID>
		<topicURL>tns1:infextend/heartbeat</topicURL>
		<wstop:TopicSet>
			<tns1:infextend>
				<heartbeat wstop:topic="true">
					<tt:MessageDescription IsProperty="true">
						<tt:Source>
							<tt:SimpleItemDescription Name="Address" Type="tt:anyURI" />
						</tt:Source>
						<tt:Data>
							<tt:SimpleItemDescription Name="State" Type="xs:boolean" />
						</tt:Data>
					</tt:MessageDescription>
				</heartbeat>
			</tns1:infextend>
		</wstop:TopicSet>
	</wstop:eventset>

```
