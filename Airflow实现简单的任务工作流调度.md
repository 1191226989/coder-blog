---
title: Airflow实现简单的任务工作流调度
created: '2023-10-04T06:59:18.214Z'
modified: '2023-10-07T08:20:10.639Z'
---

# Airflow实现简单的任务工作流调度

https://github.com/apache/airflow

https://airflow.apache.org/

#### Docker-compose 快速搭建测试环境

https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html
```
├── dags
│   ├── __pycache__
│   └── go_hello_airflow.py
├── docker-compose.yaml
├── logs
└── plugins
    └── go_hello_airflow # go build 可执行二进制文件
```
```
NAME                         COMMAND                  SERVICE             STATUS              PORTS
docker-airflow-init-1        "/bin/bash -c 'funct…"   airflow-init        exited (0)
docker-airflow-scheduler-1   "/usr/bin/dumb-init …"   airflow-scheduler   running (healthy)   8080/tcp
docker-airflow-triggerer-1   "/usr/bin/dumb-init …"   airflow-triggerer   running (healthy)   8080/tcp
docker-airflow-webserver-1   "/usr/bin/dumb-init …"   airflow-webserver   running (healthy)   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
docker-airflow-worker-1      "/usr/bin/dumb-init …"   airflow-worker      running (healthy)   8080/tcp
docker-postgres-1            "docker-entrypoint.s…"   postgres            running (healthy)   5432/tcp
docker-redis-1               "docker-entrypoint.s…"   redis               running (healthy)   6379/tcp
```

#### Aiflow dag 任务编排 go_hello_airflow.py
```py
from datetime import datetime, timedelta
import pendulum

from airflow import DAG
# from airflow.utils import dates
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator



def default_options():
	default_args = {
		"owner": "tester",
		# "start_date": dates.days_ago(1), # 任务第一次开始执行的时间
		"start_date": pendulum.yesterday(), # 任务第一次开始执行的时间
		"retries": 1, # 失败重试次数
		"retry_delay": timedelta(seconds=5) # 失败重试间隔
	}
	return default_args

def task1(dag):
	t = "date"
	# operator 支持多种类型，这里使用BashOperator
	task = BashOperator(
		task_id= "Mytask1",
		bash_command=t, # 指定要执行的命令
		dag=dag	# 指定归属的dag
	)
	return task

def hello_airflow():
	current_time = str(datetime.today())
	print("hello airflow at {}".format(current_time))
	
def task2(dag):
	# python operator
	task = PythonOperator(
		task_id="Mytask2",
		python_callable=hello_airflow, # 指定要执行的函数
		dag= dag
	)
	return task

def task3(dag):
	# go build 可执行二进制文件
	t = "/opt/airflow/plugins/go_hello_airflow"
	task = BashOperator(
		task_id="Mytask3",
		bash_command=t,
		dag=dag
	)
	return task

# 定义DAG
with DAG(
	"go_hello_airflow", # dag_id
	default_args=default_options(),# 指定默认参数
	schedule_interval="*/5 * * * *"
) as d:
	task1=task1(d)
	task2=task2(d)
	task3=task3(d)

task1 >> task2 #task2需要等待task1完成以后再执行

task1 >> task3
task2 >> [task3] #task3需要等待task1和task2完成以后再执行
```
| 概念 | 解释 |
| --- | --- |
| A >> B | B 依赖于 A，A 先执行 B 后执行 |
| A << B | A 依赖于 B，B 先执行 A 后执行 |
| A.set_downstream(B) |	等同于 A >> B |
| A.set_upstream(B) |	等同于 B >> A |


go_hello_airflow.go
```go
package main

import (
        "fmt"
        "log"
)

// go build -o go_hello_airflow hello_airflow.go
func main() {
        fmt.Println("golang say hello to airflow")
        log.Println("golang say hello to airflow")
}
```

dags 测试
```sh
docker exec -it docker-airflow-worker-1 /bin/bash
```
```
# dags 列表
airflow@777b1729f34f:/opt/airflow/dags$ airflow dags list
# 测试dags task代码
airflow@777b1729f34f:/opt/airflow/dags$ airflow dags test go_hello_airflow 20220906

  FutureWarning,
[2022-09-06 03:54:26,199] {dagbag.py:508} INFO - Filling up the DagBag from /opt/airflow/dags
[2022-09-06 03:54:26,422] {base_executor.py:91} INFO - Adding to queue: ['<TaskInstance: go_hello_airflow.Mytask4 scheduled__2022-09-06T00:00:00+00:00 [queued]>']
[2022-09-06 03:54:31,560] {subprocess.py:62} INFO - Tmp dir root location:
 /tmp
[2022-09-06 03:54:31,561] {subprocess.py:74} INFO - Running command: ['/bin/bash', '-c', '/opt/airflow/plugins/go_hello_airflow']
[2022-09-06 03:54:31,566] {subprocess.py:85} INFO - Output:
[2022-09-06 03:54:31,567] {subprocess.py:92} INFO - golang say hello to airflow
[2022-09-06 03:54:31,567] {subprocess.py:92} INFO - 2022/09/06 03:54:31 golang say hello to airflow
[2022-09-06 03:54:31,568] {subprocess.py:96} INFO - Command exited with return code 0
[2022-09-06 03:54:31,598] {backfill_job.py:378} INFO - [backfill progress] | finished run 0 of 1 | tasks waiting: 1 | succeeded: 1 | running: 0 | failed: 0 | skipped: 0 | deadlocked: 0 | not ready: 1
[2022-09-06 03:54:31,612] {base_executor.py:91} INFO - Adding to queue: ['<TaskInstance: go_hello_airflow.Mytask5 scheduled__2022-09-06T00:00:00+00:00 [queued]>']
[2022-09-06 03:54:36,434] {subprocess.py:62} INFO - Tmp dir root location:
 /tmp
[2022-09-06 03:54:36,435] {subprocess.py:74} INFO - Running command: ['/bin/bash', '-c', '/opt/airflow/plugins/go_hello_airflow']
[2022-09-06 03:54:36,440] {subprocess.py:85} INFO - Output:
[2022-09-06 03:54:36,441] {subprocess.py:92} INFO - golang say hello to airflow
[2022-09-06 03:54:36,442] {subprocess.py:92} INFO - 2022/09/06 03:54:36 golang say hello to airflow
[2022-09-06 03:54:36,442] {subprocess.py:96} INFO - Command exited with return code 0
[2022-09-06 03:54:36,463] {dagrun.py:567} INFO - Marking run <DagRun go_hello_airflow @ 2022-09-06 00:00:00+00:00: scheduled__2022-09-06T00:00:00+00:00, state:running, queued_at: 2022-09-06 00:03:00.661823+00:00. externally triggered: False> successful
[2022-09-06 03:54:36,464] {dagrun.py:627} INFO - DagRun Finished: dag_id=go_hello_airflow, execution_date=2022-09-06 00:00:00+00:00, run_id=scheduled__2022-09-06T00:00:00+00:00, run_start_date=2022-09-06 00:03:00.693728+00:00, run_end_date=2022-09-06 03:54:36.464463+00:00, run_duration=13895.770735, state=success, external_trigger=False, run_type=backfill, data_interval_start=2022-09-06 00:00:00+00:00, data_interval_end=2022-09-06 00:03:00+00:00, dag_hash=3c2cd45388b59985723331f8f2e83c48
[2022-09-06 03:54:36,465] {backfill_job.py:378} INFO - [backfill progress] | finished run 1 of 1 | tasks waiting: 0 | succeeded: 2 | running: 0 | failed: 0 | skipped: 0 | deadlocked: 0 | not ready: 0
[2022-09-06 03:54:36,468] {backfill_job.py:879} INFO - Backfill done. Exiting.
```


- golang 开发定时任务业务系统，部署 go build 二进制可执行文件
- Airflow 实现简单的可依赖的任务工作流调度，Python 编写简单的 DAG 任务编排文件

