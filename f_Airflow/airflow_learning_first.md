좋아!
# 📌 Apache Airflow 개념 정리 & 오늘 실습 기반 이론 요약

## # 📘 1. Apache Airflow란?

Apache Airflow는 **데이터 작업을 자동화하고 스케줄링하기 위한 오픈소스 워크플로우 오케스트레이션 도구**이다.

Airflow는 다음과 같은 특징을 갖는다:

* **파이썬 코드로 워크플로우(DAG)를 정의**할 수 있다.
* 시간 기반 스케줄(예: 매일 0시, 매 10분)을 자동 실행할 수 있다.
* 작업(Task) 간의 **의존성**을 손쉽게 정의할 수 있다.
* 실행 이력, 실패, 로그를 **읽기 쉬운 UI**로 제공한다.
* 자동화된 ETL, 백업, 크롤링, 모델 학습 등 실무에서 **데이터 엔지니어링의 핵심 도구**로 사용된다.

---

# 📘 2. Airflow 구성 요소 Architecture

Airflow는 여러 개의 서비스가 함께 돌아가면서 동작한다.

### ### 2.1 주요 컴포넌트

| 컴포넌트                        | 역할                        |
| --------------------------- | ------------------------- |
| **Webserver**               | UI 제공, DAG 조회/실행/모니터링     |
| **Scheduler**               | DAG 스케줄 관리, Task 실행 요청 생성 |
| **Worker**                  | 실제 Task(Python 함수 등)를 실행  |
| **Triggerer**               | 비동기 Task 처리               |
| **Metadata DB(PostgreSQL)** | DAG, Task 실행 이력 저장        |
| **Redis**                   | Celery Executor용 메시지 브로커  |
| **CLI**                     | 터미널에서 DAG 관리 명령 실행        |

---

# 📘 3. DAG의 개념 (Directed Acyclic Graph)

Airflow는 작업을 **"DAG"** 형태로 관리한다.

* **DAG (Directed Acyclic Graph)**
  : 방향성이 있고 순환이 없는 그래프 구조
  → 작업(task) 간의 **실행 순서**를 표현하는 방식

예:

```text
start → preprocess → load_to_db → finish
```

---

# 📘 4. Task & Operator

Task는 Airflow에서 실행하는 “작업 단위”.

Operator는 Task의 종류를 의미한다.

| Operator           | 설명                            |
| ------------------ | ----------------------------- |
| **EmptyOperator**  | 아무 작업도 하지 않는 Task (시작/종료 표시용) |
| **PythonOperator** | Python 함수를 실행하는 Task          |
| **BashOperator**   | Bash 명령 실행                    |
| **SQLOperator**    | SQL 실행                        |
| **Sensors**        | 특정 조건을 기다리는 Task              |

오늘 우리는:

* EmptyOperator (기초 DAG 실습)
* PythonOperator (함수 실행)

을 직접 사용했다.

---

# 📘 5. 스케줄링 개념

Airflow는 `schedule_interval` 로 Task 실행 시간을 지정한다.

| 값               | 의미                  |
| --------------- | ------------------- |
| `"@once"`       | DAG이 켜지면 딱 1번 실행    |
| `"@daily"`      | 매일 자정 실행            |
| `"0 */2 * * *"` | 2시간마다 실행 (cron 표현식) |
| `None`          | 수동 실행 전용            |

또한 `catchup=False` 를 두면 과거 날짜의 스케줄 실행을 하지 않는다.

---

# 📘 6. 오늘 실습으로 이해한 구조

오늘 네가 한 일은 Airflow의 전체 개념 중 핵심을 다뤘다:

### ✔ Docker 기반 Airflow 설치

→ Webserver, Scheduler, Worker, DB가 살아 있는 "오케스트레이션 서버" 구축

### ✔ DAG 파일을 DAGs 폴더에 넣으면 자동 인식

→ Airflow가 스스로 DAG을 읽어 UI에 보여줌

### ✔ EmptyOperator로 DAG 구조 파악

→ Task 간의 흐름(start → end) 학습

### ✔ PythonOperator로 Python 함수 자동 실행

→ Airflow가 코드를 대신 실행하는 원리 이해

### ✔ Import Error 해결

→ Airflow DAG은 스캔할 때 코드 문법 오류가 있으면 로딩 전체가 막힌다는 점 학습
→ DAG 파일은 반드시 유효한 파이썬 코드여야 함

---

# 📘 7. 오늘 만든 예시 DAG 코드 정리

## ✔ 1) EmptyOperator DAG (simple_test_dag)

```python
from airflow import DAG
from airflow.operators.empty import EmptyOperator
import pendulum

with DAG(
    dag_id="simple_test_dag",
    start_date=pendulum.datetime(2025, 11, 24, tz="Asia/Seoul"),
    schedule_interval=None,  
    catchup=False
) as dag:

    start = EmptyOperator(task_id="start")
    end = EmptyOperator(task_id="end")

    start >> end
```

---

## ✔ 2) PythonOperator DAG (hello_python_dag)

```python
from airflow import DAG
import pendulum
from airflow.operators.python import PythonOperator

def print_hello():
    print("Hello, Airflow!")

with DAG(
    dag_id="hello_python_dag",
    schedule_interval="@once",
    start_date=pendulum.datetime(2025, 11, 24, tz="Asia/Seoul"),
    catchup=False,
) as dag:

    hello_task = PythonOperator(
        task_id='print_hello_task',
        python_callable=print_hello,
    )

    hello_task
```

---

# 📘 8. Airflow가 DAG을 읽는 규칙 (중요)

* `~/ssafy_airflow/dags/` 아래의 `.py` 파일만 DAG으로 읽는다.
* 폴더 깊이(depth)는 **1단계까지만 허용**:

  * OK: `dags/folder/dag.py`
  * ❌ NO: `dags/folder/subfolder/dag.py`
* 코드에 문법 에러가 있으면 DAG 전체 로딩 실패 로그 발생

---

# 📘 9. 실무에서 Airflow가 필요한 이유

데이터 엔지니어는 다음 같은 작업을 한다:

* 매일 0시에 ETL 파이프라인 자동 실행
* 외부 API에서 데이터 수집 자동화
* Spark / SQL 작업 스케줄링
* 데이터 검증 자동 수행
* 모델 학습 자동화(MLOps)

Airflow는 이 모든 자동화의 “두뇌” 역할을 한다.

오늘 네가 만든 print 함수는 이 전체에서의 **Hello World**.

---

# 📘 10. 오늘 네가 실질적으로 배운 것들 요약

* Airflow 아키텍처가 어떻게 동작하는지
* DAG 정의법
* Task / Operator 개념
* PythonOperator의 역할
* 스케줄링 방식 (@once, catchup 등)
* DAG 폴더 구조
* Import Errors 해결하는 방법
* Docker를 통한 Airflow 실행 흐름
* UI에서 DAG 트리거 & 로그 확인

---

# 📘 11. 앞으로 확장 가능한 실습 아이디어

* BashOperator로 리눅스 명령 자동화
* SQLOperator로 DB 자동 쿼리 실행
* 크롤링 Task 넣어서 데이터 수집 자동화
* pandas로 CSV 읽고 가공하는 파이프라인 만들기
* DAG을 여러 개 연결하여 End-to-End 데이터 파이프라인 구성
