---
title: xxl-job 调度过程分析
categories: [xxl-job, 分布式调度]
tags: [分布式]
author: [xiao_e]
---

### 初始化各种内部组件

#### 初始化XxlJobScheduler
```java
@Component
public class XxlJobAdminConfig implements InitializingBean, DisposableBean {

    private XxlJobScheduler xxlJobScheduler;

    @Override
    public void afterPropertiesSet() throws Exception {
        adminConfig = this;

        xxlJobScheduler = new XxlJobScheduler();
        xxlJobScheduler.init(); //开始初始化
    }

		// 忽略其他方法
}
```
{: .file="XxlJobAdminConfig"}

####  创建调度线程
主要看`JobScheduleHelper.getInstance().start();`这行：
```java
// XxlJobScheduler#init
public void init() throws Exception {
        // init i18n
        initI18n();

        // admin trigger pool start
        JobTriggerPoolHelper.toStart();

        // admin registry monitor run
        JobRegistryHelper.getInstance().start();

        // admin fail-monitor run
        JobFailMonitorHelper.getInstance().start();

        // admin lose-monitor run ( depend on JobTriggerPoolHelper )
        JobCompleteHelper.getInstance().start();

        // admin log report start
        JobLogReportHelper.getInstance().start();

        // start-schedule  ( depend on JobTriggerPoolHelper )
        JobScheduleHelper.getInstance().start(); //看这里～～～

        logger.info(">>>>>>>>> init xxl-job admin success.");
    }
```

启动 `scheduleThread`、`ringThread`：
```java
//JobScheduleHelper#start
public void start(){
  // schedule thread
  //获取即将到达执行时间的job，并把job放到 ringData，等待触发
  scheduleThread = new Thread(/**里面的逻辑下面分析**/);
  scheduleThread.setDaemon(true);
  scheduleThread.setName("xxl-job, admin JobScheduleHelper#scheduleThread");
  scheduleThread.start();


  // ring thread
  // 从ringData获取当前秒需要执行的job, 进行触发
  ringThread = new Thread(/**里面的逻辑下面分析**/);
  ringThread.setDaemon(true);
  ringThread.setName("xxl-job, admin JobScheduleHelper#ringThread");
  ringThread.start();
}
```

### 获取调度任务

scheduleThread#run, 先关注正常处理流程，最后分析一些异常等情况：

1. 抢锁
  ```java
  // 抢锁：保证每次获取job时，只有一个调度服务器在处理，其他调度服务器进行等待
  preparedStatement = conn.prepareStatement(  "select * from xxl_job_lock where lock_name = 'schedule_lock' for update" );
  preparedStatement.execute();
  ```
2. 获取需要调度的job

   ```java
   // 获取下次触发时间在未来5秒内(nowTime + PRE_READ_MS)的job， 取 preReadCount 个
   // PRE_READ_MS = 5000
   List<XxlJobInfo> scheduleList = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleJobQuery(nowTime + PRE_READ_MS, preReadCount);
   ```

   对应的sql为：

   ```sql
   SELECT *
   FROM xxl_job_info AS t
   WHERE t.trigger_status = 1
   and t.trigger_next_time <![CDATA[ <= ]]> #{maxNextTime}
   ORDER BY id ASC
   LIMIT #{pagesize}
   ```

   

3. 把每个job放入ringData等待异步触发

   ```java
   //循环job处理
   for (XxlJobInfo jobInfo: scheduleList) {
     if (nowTime > jobInfo.getTriggerNextTime() + PRE_READ_MS) {
       //暂时忽略异常情况
     } else if (nowTime > jobInfo.getTriggerNextTime()) {
       //暂时忽略异常情况
     } else {
       // 1、获取job触发的ringSecond，对60取余
       int ringSecond = (int)((jobInfo.getTriggerNextTime()/1000)%60);
   
       // 2、将jobId放入对应的ringSecond里面，逻辑看下面
       pushTimeRing(ringSecond, jobInfo.getId());
   
       // 3、刷新下次触发时间
       refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));
     }
   }
   ```

   pushTimeRing

   ```java
   // <35, [id,id,id]>
   // <50, [id,id,id]>
   private volatile static Map<Integer, List<Integer>> ringData = new ConcurrentHashMap<>();
   
   private void pushTimeRing(int ringSecond, int jobId){
     // push async ring
     List<Integer> ringItemData = ringData.get(ringSecond);
     if (ringItemData == null) {
       ringItemData = new ArrayList<Integer>();
       ringData.put(ringSecond, ringItemData); //放入 ringData 
     }
     ringItemData.add(jobId);
   }
   ```

   

4. 异步触发job

   ringThread#run

   ```java
   // 获取当前秒需要执行的jobIds
   List<Integer> ringItemData = new ArrayList<>();
   int nowSecond = Calendar.getInstance().get(Calendar.SECOND);   // 避免处理耗时太长，跨过刻度，向前校验一个刻度；
   for (int i = 0; i < 2; i++) {
     List<Integer> tmpData = ringData.remove( (nowSecond+60-i)%60 );
     if (tmpData != null) {
       ringItemData.addAll(tmpData);
     }
   }
   
   if (ringItemData.size() > 0) {
     for (int jobId: ringItemData) {
       // 触发
       JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null, null);
     }
   }
   ```

   触发job
   > com.xxl.job.admin.core.trigger.XxlJobTrigger#processTrigger
	```java
	private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, TriggerTypeEnum triggerType, int index, int total){

        // 忽略部分代码
        // 1、 new XxlJobLog(); ==> save log-id
        // 2、init trigger-param
        // 3、init client address
     // 4、trigger remote executor
        ReturnT<String> triggerResult = null;
        if (address != null) {
            triggerResult = runExecutor(triggerParam, address); //触发触发触发
        } else {
            triggerResult = new ReturnT<String>(ReturnT.FAIL_CODE, null);
        }
   
     // 5、collection trigger info
        // 6、save log trigger-info
        logger.debug(">>>>>>>>>>> xxl-job trigger end, jobId:{}", jobLog.getId());
    }
   
   
   public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
     ReturnT<String> runResult = null;
     try {
       ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);//获取executorBiz，这里实际是：new ExecutorBizClient
       runResult = executorBiz.run(triggerParam); //触发，快到真正触发的地方了
     } catch (Exception e) {
     }
     return runResult;
   }

   //com.xxl.job.core.biz.client.ExecutorBizClient#run
   //实际是发送了一个http请求，请求方法为：run, 
   public ReturnT<String> run(TriggerParam triggerParam) {
     return XxlJobRemotingUtil.postBody(addressUrl + "run", accessToken, timeout, triggerParam, String.class);
   }
   
   ```

到这里调度服务器的job触发就完了，下面分析执行器client接收到调度请求时如何进行任务执行的。

5. 执行器client执行job
执行job的流程就比较简单了，就是执行我们写的execute逻辑。
对应的方法入口为：com.xxl.job.core.biz.impl.ExecutorBizImpl#run

```java
// scheduleThread#run

```