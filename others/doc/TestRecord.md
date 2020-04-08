# 2020.4.8 第一轮测试
## 测试用例
* MapReduce任务：http://localhost:7700/job/save?appId=1&concurrency=5&executeType=MAP_REDUCE&groupName=null&instanceRetryNum=3&instanceTimeLimit=4545454545&jobDescription=jobDescription&jobName=testJob&jobParams=%7B%22a%22%3A%22b%22%7D&maxInstanceNum=1&processorInfo=com.github.kfcfans.oms.processors.TestMapReduceProcessor&processorType=EMBEDDED_JAVA&status=1&taskRetryNum=3&taskTimeLimit=564465656&timeExpression=0%20*%20*%20*%20*%20%3F%20&timeExpressionType=CRON

## 问题记录
#### 任务执行成功，释放资源失败
第一个任务执行完成后，释放资源阶段（删除本地H2数据库中所有记录）报错，堆栈如下：
```text
2020-04-08 10:09:19 INFO  - [ProcessorTracker-1586311659084] mission complete, ProcessorTracker already destroyed!
2020-04-08 10:09:19 ERROR - [TaskPersistenceService] deleteAllTasks failed, instanceId=1586311659084.
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.github.kfcfans.common.utils.CommonUtils.executeWithRetry(CommonUtils.java:34)
	at com.github.kfcfans.oms.worker.persistence.TaskPersistenceService.execute(TaskPersistenceService.java:297)
	at com.github.kfcfans.oms.worker.persistence.TaskPersistenceService.deleteAllTasks(TaskPersistenceService.java:269)
	at com.github.kfcfans.oms.worker.core.tracker.task.TaskTracker.destroy(TaskTracker.java:231)
	at com.github.kfcfans.oms.worker.core.tracker.task.TaskTracker$StatusCheckRunnable.innerRun(TaskTracker.java:421)
	at com.github.kfcfans.oms.worker.core.tracker.task.TaskTracker$StatusCheckRunnable.run(TaskTracker.java:467)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
2020-04-08 10:09:19 WARN  - [TaskTracker-1586311659084] delete tasks from database failed.
2020-04-08 10:09:19 INFO  - [TaskTracker-1586311659084] TaskTracker has left the world.
```
随后，Server派发下来的第二个任务也无法完成创建，异常堆栈如下：
```text
2020-04-08 10:10:08 ERROR - [TaskPersistenceService] save taskTaskDO{taskId='0', jobId='1', instanceId='1586311804030', taskName='OMS_ROOT_TASK', address='10.37.129.2:2777', status=1, result='null', failedCnt=0, createdTime=1586311808295, lastModifiedTime=1586311808295} failed.
2020-04-08 10:10:08 ERROR - [TaskTracker-1586311804030] create root task failed.
[ERROR] [04/08/2020 10:10:08.511] [oms-akka.actor.internal-dispatcher-20] [akka://oms/user/task_tracker] create root task failed.
java.lang.RuntimeException: create root task failed.
	at com.github.kfcfans.oms.worker.core.tracker.task.TaskTracker.persistenceRootTask(TaskTracker.java:208)
	at com.github.kfcfans.oms.worker.core.tracker.task.TaskTracker.<init>(TaskTracker.java:81)
	at com.github.kfcfans.oms.worker.actors.TaskTrackerActor.lambda$onReceiveServerScheduleJobReq$2(TaskTrackerActor.java:138)
	at java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1660)
	at com.github.kfcfans.oms.worker.core.tracker.task.TaskTrackerPool.atomicCreateTaskTracker(TaskTrackerPool.java:30)
	at com.github.kfcfans.oms.worker.actors.TaskTrackerActor.onReceiveServerScheduleJobReq(TaskTrackerActor.java:138)
```
***
原因及解决方案：destroy方法调用了scheduledPool.shutdownNow()方法导致调用该方法的线程池被强制关闭，该方法也自然被中断，数据删到一半没删掉，破坏了数据库结构，后面的insert自然也就失败了。