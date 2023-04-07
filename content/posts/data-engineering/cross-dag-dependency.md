---
title: "[Airflow] Cross-DAG Dependency"
date: 2023-03-30T21:32:33+09:00
draft: false
---

## Introduction

워크플로를 운영/개발하다 보면 서로 다른 DAG에 있는 Task 들간에 의존성을 설정할 필요가 있습니다. 예를 들어, 1시간 주기로 돌아가는 DAG(hourly)의 23시~24시 Dag Run 이 끝난 뒤, 하루 주기로 돌아가는 DAG(daily)가 시작되야하는 경우가 있습니다. 이러한 경우에 가장 간단한 해결법은 hourly DAG가 끝나는 시간을 예상해, 이보다 늦은 시간에 daily DAG의 스케쥴을 설정하는 것입니다.

| DAG    | Schedule     | Duration      |
| ------ | ------------ | ------------- |
| hourly | `0 * * * *`  | About 10 min. |
| daily  | `30 0 * * *` | -             |

위와 같이 hourly DAG 의 통상 소요시간이 10분이라면, daily DAG의 분 스케쥴을 30분으로 설정하는 것만으로 hourly DAG의 23시~24시 Run이 끝난 후에 daily DAG 가 실행될 것입니다. 가장 간단하긴 한데 가장 좋은 해결법은 아닌 것 같습니다.

가장 큰 문제로, 처리량이 늘거나 리소스의 문제로 hourly DAG가 30분 안에 끝나지 않을 수도 있습니다. 이러한 경우 디버깅, DAG 재수행에도 공이 많이 듭니다. 반대로 이를 우려하여 daily DAG의 스케쥴에 더 여유를 주면 작업시간이 너무 늦어질 수 있습니다. 또한, 위와 같이 단순 스케쥴 조절로 의존성을 설정하면 의미 상 두 DAG의 의존성도 알아보기 힘듭니다.

---

## `ExternalTaskSensor`

Airflow 에서 지원하는 Cross-DAG Dependency를 사용하면 서로 다른 DAG에 있는 Task 간에 의존성을 설정할 수 있습니다. 위 예시처럼 두 Task 의 스케쥴이 다른 경우 뿐만 아니라, DAG의 목적이 다르거나 R&R의 이유로 DAG는 분리시키지만 의존성을 설정해야 하는 경우에도 사용할 수 있습니다.

위 예시의 파이썬 스크립트는 아래와 같습니다.

``` Python
# Airflow 2.0.1
from airflow import DAG 
from airflow.operators.dummy import DummyOperator

from datetime import datetime
import pendulum 
KST = pendulum.timezone("Asia/Seoul")

hourly_dag = DAG(
    dag_id="hourly_dag",
    schedule_interval="0 * * * *",
    start_date=datetime(2023, 3, 29, tzinfo=KST),
    tags=['tutorial']
)

daily_dag = DAG(
    dag_id="daily_dag",
    schedule_interval="0 0 * * *",
    start_date=datetime(2023, 3, 29, tzinfo=KST),
    tags=['tutorial']
)

hourly_task = DummyOperator(
    task_id="hourly_task",
    dag=hourly_dag
)

daily_task = DummyOperator(
    task_id="daily_task",
    dag=daily_dag
)

# hourly dag
hourly_task

# daily dag
daily_task
```

Cross-DAG Dependency를 위해 사용할 오퍼레이터는 `ExternalTaskSensor` 입니다.
`hourly_task`의 작업이 끝난 후 `daily_task`를 실행시키고자 아래와 같이 `is_hourly_done` Task를 `daily_dag` 에 추가합니다.

```Python
from airflow.sensors.external_task import ExternalTaskSensor

is_hourly_done = ExternalTaskSensor(
    task_id="is_hourly_done",
    external_dag_id=hourly_task.dag_id,
    external_task_id=hourly_task.task_id,
    allowed_states=["success"], # default is ["success"]
    failed_states=["failed"],  # default is None
    dag=daily_dag
)

# hourly dag
hourly_task

# daily dag
is_hourly_done >> daily_task
```

이제 우리의 기대처럼 `hourly_task` 의 29일 23시~24시 Run이 완료 된 후 29일 `daily_task`가 실행 될까요?

**아닙니다.**

기대와는 달리 `hourly_task` 의 29일 00시~01시 Run 이 완료 된 후  `daily_task`  29일 Run 이 실행됩니다.

_`ExternalTaskSensor`는 여타 Airflow 로직과 동일하게 `execution_date`를 기준으로 작동하기 때문입니다._

| ID  | Task          | Schedule    | exeuction_date | data_interval      | start_date |
| :-- | :------------ | :---------- | :------------- | :----------------- | :--------- |
| 1   | `hourly_task` | `0 * * * *` | 29일 00:00     | 29일 00:00 ~ 00:59 | 29일 01:00 |
| 2   | `hourly_task` | `0 * * * *` | 29일 23:00     | 29일 23:00 ~ 23:59 | 30일 00:00 |
| 3   | `daily_task`  | `0 0 * * *` | 29일 00:00     | 29일 00:00 ~ 23:59 | 30일 00:00 |

Task Instance 마다 처리해야할 데이터 구간인 `data_interval`, Task 예정 시작시간인 `start_date`를 나타낸 테이블입니다.

만약 `ExternalTaskSensor`가 같은 `start_date` 를 기준으로 작동한다면, 우리의 순진한 기대대로 2번, 3번 Task간에 의존성이 설정 될것입니다. 하지만 `execution_date`를 기준으로 작동하기 때문에 1번, 3번 Task간에 의존성이 설정됩니다.

서로 다른 `exeuction_date`를 가진 Task간에 의존성을 설정하려면, `execution_date_fn`나 `execution_delta` 인자를 설정하면 됩니다. `execution_date_fn` 인자가 더 명확하다고 생각되어 이를 사용하겠습니다.

위의 예의 경우 3번 `daily_task`의 `execution_date`는 `29일 00:00`, 2번 `hourly_task`의 `execution_date`는 `29일 23:00`이므로 23시간을 더해주는 함수 `execution_date_fn=lambda dt: dt + timedelta(hours=23)` 를 인자로 주면 됩니다.

```Python
from airflow.sensors.external_task import ExternalTaskSensor

from datetime import datetime, timedelta

is_hourly_done = ExternalTaskSensor(
    task_id="is_hourly_done",
    external_dag_id=hourly_task.dag_id,
    external_task_id=hourly_task.task_id,
    allowed_states=["success"], # default is ["success"]
    failed_states=["failed"],  # default is None
    execution_date_fn=lambda dt: dt + timedelta(hours=23), # execution_delta=timedelta(hours=-23),
    dag=daily_dag
)
```

Airflow Web UI에서 `daily_dag`의 `ExternalTaskSensor` 인스턴스의 로그를 확인해보면, 의존성이 제대로 설정 되었는지 확인할 수 있습니다.

``` None
INFO - Executing <Task(ExternalTaskSensor): sensor> on 2023-03-28T15:00:00+00:00
...
INFO - Poking for hourly_dag.hourly_task on 2023-03-29T14:00:00+00:00 ... 
```

`KST`가 아닌 `UTC`로 로그가 남는 점을 주의해야합니다. 최종 파이썬 스크립트는 아래와 같습니다.

```Python
# Airflow 2.0.1
from airflow import DAG 

from airflow.operators.dummy import DummyOperator
from airflow.sensors.external_task import ExternalTaskSensor

from datetime import datetime, timedelta
import pendulum
KST = pendulum.timezone("Asia/Seoul")


hourly_dag = DAG(
    dag_id="hourly_dag",
    schedule_interval="0 * * * *",
    start_date=datetime(2023, 3, 29, tzinfo=KST),
    tags=['tutorial']
)

daily_dag = DAG(
    dag_id="daily_dag",
    schedule_interval="0 0 * * *",
    start_date=datetime(2023, 3, 29, tzinfo=KST),
    tags=['tutorial']
)

hourly_task = DummyOperator(
    task_id="hourly_task",
    dag=hourly_dag
)

daily_task = DummyOperator(
    task_id="daily_task",
    dag=daily_dag
)

is_hourly_done = ExternalTaskSensor(
    task_id="is_hourly_done",
    external_dag_id=hourly_task.dag_id,
    external_task_id=hourly_task.task_id,
    allowed_states=["success"], # default is ["success"]
    failed_states=["failed"],  # default is None
    execution_date_fn=lambda dt: dt + timedelta(hours=23), # execution_delta=timedelta(hours=-23),
    dag=daily_dag
)

# hourly dag
hourly_task

# daily dag
is_hourly_done >> daily_task
```
  
---

## Timezone 함정

`exectuion_date`에 월(Month) 연산이 들어간다면 Timezone에 영향을 받게되어 원하는 결과가 나오지 않을 수 있습니다.

| ID  | Task         | Schedule    | exeuction_date (KST) | data_interval                       | start_date       | exeuction_date (UTC) |
| :-- | :----------- | :---------- | :------------------- | :---------------------------------- | :--------------- | :------------------- |
| 1   | daily_task   | `0 0 * * *` | 2023-02-28 00:00 KST | 2023-02-28 00:00 ~ 2023-02-28 23:39 | 2023-03-01 00:00 | 2023-02-27 15:00 UTC |
| 2   | daily_task   | `0 0 * * *` | 2023-03-31 00:00 KST | 2023-03-3l 00:00 ~ 2023-03-31 23:39 | 2023-04-01 00:00 | 2023-03-30 15:00 UTC |
| 3   | monthly_task | `0 0 1 * *` | 2023-02-01 00:00 KST | 2023-02-01 00:00 ~ 2023-02-28 23:59 | 2023-03-01 00:00 | 2023-01-31 15:00 UTC |
| 4   | monthly_task | `0 0 1 * *` | 2023-03-01 00:00 KST | 2023-03-01 00:00 ~ 2023-03-31 23:59 | 2023-04-01 00:00 | 2023-02-28 15:00 UTC |

한 달 동안의 `daily_task`가 모두 완료 된후 그 달의 `monthly_task`가 실행되기를 원하는 상황을 가정해봅시다. `data_interval`를 살펴보면 2월의 경우, 1번 Task 종료 후 3번 Task가 실행되어야합니다. 3월의 경우 2번 Task가 끝난 후 4번 Task가 실행되어야합니다. `exeuction_date`를 맞춰주기 위해 한달을 더해주고 하루를 빼주면 될 것 같습니다.

```Python
from datetime import datetime, timedelta
from dateutil.relativedelta import relativedelta

exectuion_date_fn=lambda dt: dt + relativedelta(months=1) - timedelta(days=1)
```

위와 같이 `exectuion_date_fn`를 설정하면 될 것 같습니다.

**하지만, 2월은 원하는대로 작동하지만 3월은 그렇지않습니다**.

이유는 Airflow의 `exectuion_date`는 `UTC`기준이기 때문입니다.

3번 Task의 `execution_date`는 `UTC`로 `2023-01-31 15:00`이고 여기에 `execuiton_date_fn`를 적용하면 `2023-02-27 15:00 UTC`로 1번 Task의 `exectuion_date`로 기대대로 설정됩니다.

반면에, 4번 Task의 `execution_date`는 `UTC`로 `2023-02-28 15:00`이고 여기에 `execuiton_date_fn`를 적용하면 `2023-03-27 15:00 UTC`로 2번 Task의 `exectuion_date`와는 다른 값으로 설정됩니다.

이러한 문제는 간단하게 `exectuion_date_fn` 인자에 `UTC` ↦ `KST` 변환을 추가하면 해결됩니다.

```Python
execution_date_fn=lambda dt: dt.astimezone(KST) + relativedelta(months=1) - timedelta(days=1)
```

상황에 따라 Timezone 영향을 받지 않을 수도 있지만, 오류를 방지하려면 월 연산의 경우 Timezone 설정에 신경을 쓰는 편이 좋겠습니다.

---

## Conclusion

- 서로 다른 DAG에 있는 Task 간에 의존성을 설정할 수 있다. (`from airflow.sensors.external_task import ExternalTaskSensor`)
- Task의 `exectuion_date`를 잘 확인해야 한다.
- `ExternalTaskSensor` 인스턴스의 로그를 통해 의존성을 다시 확인하자.
- Timezone 에 영향을 받을 수도 있다.

<!-- 
## Examples

몇가지 예시를 더 들어보겠습니다.

| ID  | Task         | Schedule     | exeuction_date (KST) | data_interval                       | start_date       | exeuction_date (UTC) |
| :-- | :----------- | :----------- | :------------------- | :---------------------------------- | :--------------- | :------------------- |
| 1   | hourly_task  | `0 * * * *`  | 2023-02-01 00:00 KST | 2023-02-01 00:00 ~ 2023-02-01 00:59 | 2023-02-01 01:00 | 2023-01-31 15:00 UTC |
| 2   | hourly_task  | `0 * * * *`  | 2023-02-28 23:00 KST | 2023-02-28 23:00 ~ 2023-02-28 23:59 | 2023-03-01 00:00 | 2023-02-28 14:00 UTC |
| 3   | hourly_task  | `0 * * * *`  | 2023-02-28 00:00 KST | 2023-02-28 00:00 ~ 2023-02-28 00:59 | 2023-02-28 01:00 | 2023-02-27 15:00 UTC |
| 4   | hourly_task  | `0 * * * *`  | 2023-02-01 23:00 KST | 2023-02-01 23:00 ~ 2023-02-01 23:59 | 2023-02-02 00:00 | 2023-02-07 14:00 UTC |
| 5   | daily_task_1 | `0 0 * * *`  | 2023-02-01 00:00 KST | 2023-02-01 00:00 ~ 2023-02-01 23:59 | 2023-02-02 30:00 | 2023-01-31 15:00 UTC |
| 5   | daily_task_2 | `30 0 * * *` | 2023-02-01 00:00 KST | 2023-02-01 00:00 ~ 2023-02-01 23:59 | 2023-02-02 30:00 | 2023-01-31 15:00 UTC |
| 6   | daily_task_2 | `30 0 * * *` | 2023-02-28 00:30 KST | 2023-02-28 30:00 ~ 2023-02-28 00:29 | 2023-03-01 30:00 | 2023-02-27 15:00 UTC |
| 7   | monthly_task | `0 0 1 * *`  | 2023-02-01 00:00 KST | 2023-02-01 00:00 ~ 2023-02-28 23:59 | 2023-03-01 00:00 | 2023-01-31 15:00 UTC |

2번, 6번, 7번 Task는 모두 같은 `start_date`를 가지지만, `exectuion_date`는 모두 다릅니다. 

`daily_task_2`의 경우, 30분 늦게 스케쥴 되어있습니다.

### Use Case 1

2월 28일 하루 안에 도는 모든 hourly 배치를 끝낸 후에 2월 28일의 데이터를 이용하여 daily_task_2 를 돌릴려면 어떻게 해야할까요. `data_interval`를 확인해보면 2번 Task가 완료된 후에 6번 Task를 돌려야한다는 것을 알 수 있습니다. 
`execution_date_fn` 인자를 설정하지 않으면, 동일한 `exectuion_date`를 가진 1번 Task
2번 task의 execution_date는 `2023-02-28 23:00 KST` 이고, 6번 task의 execution_date는 `2023-02-28 00:00 KST` 이다.

이를 위해 

### Use Case 2
2월 28일 daily 배치를 끝낸 후에 
2월의 데이터를 이용하여 monthly 배치를 돌려보자.****

비즈니스 로직을 보면, 6번 task가 완료된 후에 (2월 28일 데이터 처리가 완료된 후에) 7번 task가 실행되어야 한다. 6번 task의 execution_date는 `2023-02-28 00:00 KST` 이고, 7번 task의 execution_date는 `2023-02-01 00:00 KST` 이다.
이를 위해
`lambda dt: dt + relativedelta(month=1) - timedelta(days=1)` 

하지만!!!,

---

## 
왜 그런지 이해하기 위해서는 DAG Run의 `exeuction_date`의 의미를 알아야합니다.
## `exeuction_date`

`execution_date`를 이해하기 위해서는 DAG Run을 구간으로 생각하는게 도움이 됩니다.

예를 들어, 스케쥴이 `0 * * * * *` 인 `hourly_dag`는

- ...
- `29일 00:00 ~ 00:59`
- `29일 01:00 ~ 01:59`
- ...
- `30일 00:00 ~ 23:59`
- ...
  
등의 DAG Run 구간을 가지고 있습니다. `29일 00:00 ~ 00:59` 구간은 29일 0시부터 1시까지의 데이터를 처리해야한다는 것을 의미합니다. 

이 DAG Run의 `execution_date`는 구간의 시작시간인 `29일 00:00`입니다. 그리고 예정된 작업 시작시간은 구간의 종료시간인 29일 1시입니다.
마찬가지로, 구간이 `29일 01:00 ~ 01:59`의 `execution_date`는 `29일 01:00`이고 예정된 작업시작시간은 29일 2시입니다.

`exectuion_date`는 직역하면 실행시간인데, 실제로 DAG Run이 실행되는 시간이 아닙니다. 이러한 이유로 많은 사용자들을 헷갈리게 했고, Airflow 2.2 버전부터는 더 잘어울리는 `logical_date`로 이름이 바뀝니다.




정리하자면, `exectuion_date`는 

---
스케쥴이 `0 0 * * *` (daily) 인 DAG는 [`29일 00:00 ~ 29일 23:59`, `30일 00:00 ~ 30일 23:59`, `1일 00:00 ~ 1일 23:59`, ...] 등의 DAG Run을 가지고 있습니다. 

DAG 인스턴스의 ID 


---
## 추가
- external_task_id 를 명시하지않으면 dag 전체 
- exeuction_delta 가 없으면 같은 인자



execution_delta or execution_date_fn



# Example
몇가지 예제를 살펴보겟슴둥.

스케쥴이 다르거나, 워크플로 의미 상 목적

naive 한 방법으로는 



| ID  | Task            | Schedule    | exeuction_date (KST) | started          | logical_interval                     | business_interval                    | exeuction_date (UTC) |
| :-- | :-------------- | :---------- | :------------------- | :--------------- | :----------------------------------- | :----------------------------------- | :------------------- |
| 1   | trip_summary    | `0 * * * *` | 2023-02-01 00:00 KST | 2023-02-01 01:00 | 2023-02-01 00:00 ~ 2023-02-01 00:59  | 2023-02-01 00:00 ~  2023-02-01 00:59 | 2023-01-31 15:00 UTC |
| 2   | trip_summary    | `0 * * * *` | 2023-02-28 23:00 KST | 2023-03-01 00:00 | 2023-02-28 23:00 ~  2023-02-28 23:59 | 2023-02-28 23:00 ~ 2023-02-28 23:59  | 2023-02-28 14:00 UTC |
| 3   | trip_summary    | `0 * * * *` | 2023-02-28 00:00 KST | 2023-02-28 01:00 | 2023-02-28 00:00 ~  2023-02-28 00:59 | 2023-02-28 00:00 ~ 2023-02-28 00:59  | 2023-02-27 15:00 UTC |
| 4   | trip_summary    | `0 * * * *` | 2023-02-01 23:00 KST | 2023-02-02 00:00 | 2023-02-01 23:00 ~  2023-02-01 23:59 | 2023-02-01 23:00 ~ 2023-02-01 23:59  | 2023-02-07 14:00 UTC |
| 5   | vin_summary     | `0 0 * * *` | 2023-02-01 00:00 KST | 2023-02-02 00:00 | 2023-02-01 00:00 ~  2023-02-01 23:59 | 2023년 2월 1일                       | 2023-01-31 15:00 UTC |
| 6   | vin_summary     | `0 0 * * *` | 2023-02-28 00:00 KST | 2023-03-01 00:00 | 2023-02-28 00:00 ~  2023-02-28 23:39 | 2023년 2월 28일                      | 2023-02-27 15:00 UTC |
| 7   | monthly_summary | `0 0 1 * *` | 2023-02-01 00:00 KST | 2023-03-01 00:00 | 2023-02-01 00:00 ~  2023-02-28 23:59 | 2023년 2월                           | 2023-01-31 15:00 UTC |


먼저 주의해야할 점은, 2, 6, 7번 task의 started 시간은 같지만 execution_date는 모두 다르다는 점이다.










External TAsk Sensor 가 바라보는 execution_date는 로그에서 확인할 수 있습니다. 

REF https://airflow.apache.org/docs/apache-airflow/stable/howto/operator/external_task_sensor.html

https://blog.bsk.im/2021/03/21/apache-airflow-aip-39/

DAG에는 구간이 있다 라고 이해하는게 젤 쉽다. execution_date 는 

| ID  | Task            | Schedule    | exeuction_date (KST) | start_date       | data_interval                        | exeuction_date (UTC) |
| :-- | :-------------- | :---------- | :------------------- | :--------------- | :----------------------------------- | :------------------- |
| 1   | trip_summary    | `0 * * * *` | 2023-02-01 00:00 KST | 2023-02-01 01:00 | 2023-02-01 00:00 ~ 2023-02-01 00:59  | 2023-01-31 15:00 UTC |
| 2   | trip_summary    | `0 * * * *` | 2023-02-28 23:00 KST | 2023-03-01 00:00 | 2023-02-28 23:00 ~  2023-02-28 23:59 | 2023-02-28 14:00 UTC |
| 3   | trip_summary    | `0 * * * *` | 2023-02-28 00:00 KST | 2023-02-28 01:00 | 2023-02-28 00:00 ~  2023-02-28 00:59 | 2023-02-27 15:00 UTC |
| 4   | trip_summary    | `0 * * * *` | 2023-02-01 23:00 KST | 2023-02-02 00:00 | 2023-02-01 23:00 ~  2023-02-01 23:59 | 2023-02-07 14:00 UTC |
| 5   | vin_summary     | `0 0 * * *` | 2023-02-01 00:00 KST | 2023-02-02 00:00 | 2023-02-01 00:00 ~  2023-02-01 23:59 | 2023-01-31 15:00 UTC |
| 6   | vin_summary     | `0 0 * * *` | 2023-02-28 00:00 KST | 2023-03-01 00:00 | 2023-02-28 00:00 ~  2023-02-28 23:39 | 2023-02-27 15:00 UTC |
| 7   | monthly_summary | `0 0 1 * *` | 2023-02-01 00:00 KST | 2023-03-01 00:00 | 2023-02-01 00:00 ~  2023-02-28 23:59 | 2023-01-31 15:00 UTC | -->