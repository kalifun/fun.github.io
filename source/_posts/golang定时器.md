---
title : Golang 定时器
date: 2021-02-19 11:39:12
categories: Golang
tags :
  - go
  - golang
---

# Golang 定时器

> 有一批业务数据需要定时处理，业务是采取的分布式架构，部署了多个节点，但是不想因为这一部分的功能开新的节点，或者利用redis来处理。所以在后台代码中新增了一个定时处理的逻辑。  

痛点：

1. 多节点启动时会造成数据处理2遍。
2. 开设单独节点新增维护成本。  

## 锁

我们采用锁的形式来解决问题。

![](https://image.kalifun.top/upload/2102/97a9ea7a117d9a12.png)

1. 多节点去抢锁。抢锁成功的执行具体函数。
2. 失败的则在下个节点继续抢锁。

## 实现逻辑

库：

- github.com/robfig/cron/v3 （定时任务）  

数据库模型：

```go
/*
TaskLock表
 */
type TaskLock struct {
	Id      int64  `json:"id"`
	LockKey string `json:"lock_key" orm:"unique"`
}

```



加锁：

```go
// 加锁方法，bool值表示是否成功
func AddLock(lockValue string) bool {
	insertData := models.TaskLock{
		Id:      1,   //避免一直递增超过最大数
		LockKey: lockValue,
	}
	o := orm.NewOrm()
	_, err := o.Insert(&insertData)
	if err != nil {
		global.Oplog.Error(err)
		return false
	}
	return true
}
```

解锁：

```go
// 删除锁方法，只要成功添加了锁，任务执行后，无论成功还是失败都必须调用删除方法
func deleteLock(lockValue string) {
	o := orm.NewOrm()
	_, err := o.QueryTable(new(models.TaskLock)).Filter("lock_key", lockValue).Delete()
	if err != nil {
		global.Oplog.Error(err)
	}
	//if _, err := o.Delete(&models.TaskLock{LockKey: lockValue}); err == nil {
	//	global.Oplog.Error(err)
	//}
	global.Oplog.Info("The aggregation task has been completed")
}
```

装饰器：

```go
// 单节点任务装饰器，被装饰的任务在分布式多节点下同一时间只能运行一次
func SingleTask(wrap func(lockValue string) bool) func(lockValue string) {
	return func(lockValue string) {
		add := wrap(lockValue)
		if add {
			global.Oplog.Info("当前节点正在进行:", lockValue)
			// 执行程序
			//CleanTemporaryData()
			deleteLock(lockValue)
		} else {
			global.Oplog.Info("当前节点获取任务失败")
		}
	}
}
```

主程序：

```go
crontab := cron.New()
task := func() {
  //global.Oplog.Info("Hello Word")
  decorator.SingleTask(decorator.AddLock)("test")
}
// 添加定时任务, * * * * * 是 crontab,表示每分钟执行一次
id, err := crontab.AddFunc("* * * * *", task)
if err != nil {
  global.Oplog.Error(err)
}
```

## 总结

像采用这种方式就不需要担心多节点都会同事处理相同数据了。这也是变相的使用锁的思想。


