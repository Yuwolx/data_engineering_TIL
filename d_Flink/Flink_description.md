# 🌀 Apache Flink 완전 정리

## 📘 1. Flink란?

> **Apache Flink**는 실시간(Streaming) 및 배치(Batch) 데이터 처리를 위한 **분산 데이터 처리 엔진**입니다.
> “데이터가 들어오는 즉시 계산할 수 있는 Spark보다 빠른 실시간 분석 엔진”이라고 생각하면 됩니다.

---

### 💡 Flink의 핵심 개념

| 개념                 | 설명                                                            |
| ------------------ | ------------------------------------------------------------- |
| **Stream**         | 끝이 없는 데이터 흐름 (예: Kafka 메시지, 실시간 센서 데이터 등)                     |
| **Batch**          | 고정된 크기의 데이터 묶음 (예: CSV 파일, 로그 파일 등)                           |
| **Transformation** | 데이터를 변환하거나 필터링하는 단계 (`map`, `flat_map`, `filter`, `reduce` 등) |
| **Keyed Stream**   | 특정 키를 기준으로 데이터를 그룹화 (`key_by`)                                |
| **Window**         | 시간 단위로 스트림을 나누는 개념 (`time_window`, `count_window`)            |
| **Sink**           | 결과를 저장하거나 외부로 내보내는 단계 (CSV, DB, Kafka 등)                      |

---

### ⚙️ Flink vs. Spark vs. Kafka

| 항목    | Flink                          | Spark               | Kafka          |
| ----- | ------------------------------ | ------------------- | -------------- |
| 주요 목적 | 실시간 스트리밍 + 배치 처리               | 배치 중심 + 스트리밍 지원     | 메시지 큐 (데이터 전달) |
| 처리 단위 | 이벤트 기반                         | 마이크로 배치             | 메시지 단위         |
| 지연 시간 | 매우 짧음(ms 단위)                   | 상대적으로 김(sec 단위)     | 매우 짧음          |
| 사용 예  | 실시간 금융 거래 분석, IoT 데이터, 로그 모니터링 | 대규모 배치 연산, ML 파이프라인 | 데이터 브로커 역할     |

---

## 🧩 2. PyFlink 환경 설정

### ✅ 설치 & 기본 세팅

```bash
# Flink 설치
wget https://archive.apache.org/dist/flink/flink-1.19.3/flink-1.19.3-bin-scala_2.12.tgz
tar -xvzf flink-1.19.3-bin-scala_2.12.tgz
mv flink-1.19.3 flink

# Flink 클러스터 실행
cd flink/bin
./start-cluster.sh

# UI 접속
http://localhost:8081
```

> `localhost:8081` → Flink 대시보드 (JobManager)

---

### 🧱 기본 구조 이해

Flink의 코드는 보통 다음 3단계로 구성됩니다.

1. **환경(Environment)** 생성
2. **데이터(DataStream/Table)** 불러오기
3. **변환 및 출력(Transformation & Sink)**

---

## 🔹 3. PyFlink 기초 실습 요약

---

### ✅ ① 금융 뉴스 WordCount 실습

#### 💭 목적

뉴스 기사 텍스트(`news_text`)를 읽어, 단어별 빈도수를 계산.

#### 🧩 핵심 코드

```python
env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(2)

df = pd.read_csv("../data/data.csv")
news_texts = df["news_text"].dropna().tolist()
text_stream = env.from_collection(news_texts, type_info=Types.STRING())

word_count = (
    text_stream
    .map(lambda text: [(w.lower(), 1) for w in text.split()], 
         output_type=Types.LIST(Types.TUPLE([Types.STRING(), Types.INT()])))
    .flat_map(lambda words: words, 
         output_type=Types.TUPLE([Types.STRING(), Types.INT()]))
    .key_by(lambda x: x[0])
    .reduce(lambda a, b: (a[0], a[1] + b[1]))
)

word_count.print()
env.execute("Finance News WordCount")
```

#### 🧠 포인트

* `.map()` → 문장 → 단어 리스트
* `.flat_map()` → 단어 하나씩 펼치기
* `.key_by()` + `.reduce()` → 단어별 합산
* **결과:** 단어별 등장 횟수 출력

---

### ✅ ② 거래 유형별 총 거래 금액 집계 (Streaming)

#### 💭 목적

CSV 데이터를 읽어 거래 유형별(`transaction_type`) 총 거래 금액 합산.

#### 🧩 코드 핵심

```python
transaction_stream = env.from_collection(
    transactions, type_info=Types.TUPLE([Types.STRING(), Types.FLOAT()])
)

total_amount = (
    transaction_stream
    .key_by(lambda x: x[0])
    .reduce(lambda a, b: (a[0], a[1] + b[1]))
)
total_amount.print()
env.execute("Streaming Transaction Processing")
```

#### 🧠 개념 요약

| 연산       | 설명               |
| -------- | ---------------- |
| `key_by` | 거래 유형별로 데이터 그룹화  |
| `reduce` | 같은 key의 값을 누적 합산 |

---

### ✅ ③ 거래 유형별 금액 집계 (PyFlink + Pandas 전처리)

#### 💭 목적

Pandas로 CSV를 읽고 PyFlink에서 거래유형별(`transaction_type`) 합계 집계.

#### 💻 코드 요약

```python
df = pd.read_csv("../data/financial_transactions.csv")
transactions = df[["transaction_type", "amount"]].dropna().values.tolist()
transaction_stream = env.from_collection(transactions, type_info=Types.TUPLE([Types.STRING(), Types.FLOAT()]))

summary = transaction_stream.key_by(lambda x: x[0]).sum(1)
summary.print()
env.execute("PyFlink Transaction Sum")
```

#### 🧠 포인트

* `sum(1)` 은 튜플의 두 번째 인덱스(`amount`)를 합산.
* `key_by` + `sum` = 그룹별 합계 (SQL의 `GROUP BY + SUM`과 동일)

---

### ✅ ④ Flink SQL을 활용한 배치 처리 (Batch Mode)

#### 💭 목적

`sector`(부문)별로 총 거래액, 평균가, 총 거래량, 거래 건수 집계.

#### 🧩 핵심 코드

```python
env_settings = EnvironmentSettings.in_batch_mode()
table_env = TableEnvironment.create(env_settings)

# CSV 입력 테이블
table_env.execute_sql(f"""
CREATE TABLE finance (
    stock_code STRING,
    sector STRING,
    price DOUBLE,
    volume INT,
    transaction_date STRING
) WITH (
    'connector' = 'filesystem',
    'path' = '{input_path}',
    'format' = 'csv'
)
""")

# SQL 집계
result = table_env.execute_sql("""
SELECT
    sector,
    SUM(price * volume) AS total_value,
    AVG(price) AS avg_price,
    SUM(volume) AS total_volume,
    COUNT(*) AS transaction_count
FROM finance
GROUP BY sector
""")

for row in result.collect():
    print(row)
```

#### 📊 결과 예시

```
=== 섹터별 금융 데이터 요약 ===
섹터           총 거래액           평균 가격        총 거래량       거래 건수
--------------------------------------------------------------------
semiconductor  88,352,397,157.52   501,889.14      170,360         340
internet       91,203,281,346.46   543,158.49      164,929         317
biotech        91,891,967,747.97   537,173.32      175,895         343
```

#### 🧠 포인트 요약

| 키워드                                                | 의미                 |
| -------------------------------------------------- | ------------------ |
| `TableEnvironment`                                 | SQL 기반 Flink 실행 환경 |
| `CREATE TABLE ... WITH ('connector'='filesystem')` | CSV 파일을 테이블처럼 사용   |
| `SUM / AVG / COUNT`                                | SQL 집계 함수          |
| `collect()`                                        | 결과를 파이썬에서 출력       |

---

## 🧠 4. Flink 핵심 개념 요약

| 구분                             | 설명                          | 예시                                        |
| ------------------------------ | --------------------------- | ----------------------------------------- |
| **StreamExecutionEnvironment** | 실시간 스트림 실행 환경               | `.execute("Job Name")`                    |
| **TableEnvironment**           | SQL 기반 배치/스트림 처리 환경         | `.execute_sql("SELECT ...")`              |
| **key_by()**                   | 그룹 기준 지정                    | `key_by(lambda x: x[0])`                  |
| **reduce() / sum()**           | 누적 연산                       | `.reduce(lambda a, b: (a[0], a[1]+b[1]))` |
| **map / flat_map**             | 데이터 변환                      | `.map(lambda x: ...)`                     |
| **execute_sql()**              | SQL 실행 (DDL, DML, SELECT 등) |                                           |
| **collect()**                  | SQL 결과 출력                   | `for row in result.collect(): ...`        |

---

## 🚀 5. 한눈에 보는 Flink 흐름 다이어그램

```
        +-------------------------------+
        |        데이터 입력 (CSV)       |
        +---------------+---------------+
                        |
                        v
        +---------------+---------------+
        |   PyFlink 변환 단계 (map, key_by, sum) |
        +---------------+---------------+
                        |
                        v
        +---------------+---------------+
        |   SQL TableEnv 집계 (GROUP BY) |
        +---------------+---------------+
                        |
                        v
        +---------------+---------------+
        |     결과 출력 / 파일 저장     |
        +-------------------------------+
```

---

## 📚 마무리 요약

* **PyFlink = Flink + Python 인터페이스**
  → Pandas처럼 직관적이면서도, 분산 환경에서 스트리밍/배치 모두 처리 가능.

* Flink의 강점
  ✅ 실시간 처리 (Streaming)
  ✅ SQL 집계 (Batch)
  ✅ Kafka, HDFS 등 다양한 소스 연동

* **핵심 문법 3종 세트**

  1. `key_by()` → 그룹화
  2. `sum()` / `reduce()` → 누적 집계
  3. `execute()` / `execute_sql()` → 실행

---

## 🏁 학습 정리 포인트

| 단계  | 실습 내용               | 배운 개념                    |
| --- | ------------------- | ------------------------ |
| 1단계 | WordCount           | map, flat_map, reduce    |
| 2단계 | 거래 유형별 금액 합산        | key_by, sum              |
| 3단계 | Pandas + Flink 스트리밍 | from_collection, Type 지정 |
| 4단계 | Flink SQL 배치 처리     | TableEnvironment, SQL 집계 |

---

### 🔖 한 줄 정리

> **Flink는 “데이터가 들어오는 즉시 계산되는 분산 실시간 엔진”이다.**
> PyFlink를 사용하면 Python만으로 SQL, 스트리밍, 배치 분석을 한 번에 다룰 수 있다.

