# 📘 **Spark · RDD · DataFrame · Structured Streaming · Kafka — 이론 총정리**

---

# # 🧩 **1. Apache Spark란?**

### 💡 **Spark는 대규모 데이터를 "분산 처리"하기 위한 클러스터 컴퓨팅 엔진**

## ✨ Spark의 핵심 특징

* **In-memory 연산**: Hadoop MapReduce는 디스크 기반 → Spark는 메모리 기반이라 100배까지 빠름
* **Lazy Evaluation(지연 실행)**: 변환(Transformation)은 즉시 실행되지 않고 DAG 형태로 쌓였다가 Action이 호출될 때 실행됨
* **Fault-tolerance**: RDD의 Lineage(연산 이력) 기반으로 데이터 복구 가능
* **다양한 API 제공**

  * **RDD API**
  * **DataFrame API**
  * **SQL API**
  * **Structured Streaming**

---

# # 🧩 **2. RDD (Resilient Distributed Dataset)**

### RDD는 Spark의 **가장 기본적인 분산 데이터 모델**

## 🔥 RDD의 특징

* **Immutable(불변)**: 한 번 생성되면 변경되지 않음 (새 객체를 생성)
* **Distributed(분산)**: 클러스터의 여러 노드에 나누어 저장됨
* **Fault-tolerant**: Lineage를 통해 자동 복구 가능
* **Lazy Evaluation** 적용

---

## 📌 Transformation vs Action

| 분류                 | 특징                | 예시                                  |
| ------------------ | ----------------- | ----------------------------------- |
| **Transformation** | 실행되지 않고 DAG에 기록됨  | map, filter, flatMap, mapPartitions |
| **Action**         | 실제 클러스터에서 연산이 실행됨 | collect, count, take, reduce        |

---

## 📌 주요 Transformation

### ① **map**

* 요소 1개 → 1개로 변환하는 함수

```python
rdd.map(lambda x: x * 2)
```

### ② **filter**

* 조건을 만족하는 요소만 통과

```python
rdd.filter(lambda x: x % 2 == 0)
```

### ③ **flatMap**

* 요소 1개 → 여러 개로 확장할 때 사용

```python
rdd.flatMap(lambda x: x.split(" "))
```

### ④ **mapPartitions**

* 파티션 단위로 데이터를 처리
* 맵보다 훨씬 빠름 (파티션 단위로 Python ↔ JVM 오버헤드 감소)

```python
rdd.mapPartitions(lambda iter: (x*2 for x in iter))
```

---

## 📌 주요 Action

### ① collect()

RDD를 모두 드라이버로 가져옴
→ 데이터가 클 경우 절대 사용하면 안 됨

### ② count()

요소 개수 반환

### ③ take(n)

앞에서 n개 데이터만 수집

### ④ reduce()

집계 연산 수행

---

# # 🧩 **3. RDD Sampling & randomSplit**

### ✔ sample(withReplacement, fraction)

```python
rdd.sample(False, 0.2)  # 비복원, 20%
```

### ✔ takeSample

```python
rdd.takeSample(False, 5)
```

### ✔ randomSplit

훈련·테스트 데이터 분할할 때 사용

```python
train, test = rdd.randomSplit([0.8, 0.2], seed=42)
```

---

# # 🧩 **4. DataFrame & Spark SQL**

### DataFrame은 RDD보다 더 고수준 API

→ 스키마 기반, Catalyst Optimizer 사용 → 훨씬 빠르고 효율적

---

## 📌 DataFrame 생성

```python
df = spark.createDataFrame(data, columns)
```

## 📌 select, filter, orderBy, groupBy

```python
df.select("name", "age")
df.filter(col("age") > 25)
df.orderBy(col("age").desc())
df.groupBy("department").agg(avg("age"))
```

---

## 📌 스키마 출력

```python
df.printSchema()
```

---

## 📌 스키마 정의(StructType)

```python
schema = StructType([
    StructField("Name", StringType()),
    StructField("Age", IntegerType())
])
```

---

## 📌 Type Casting (데이터 타입 변환)

### 문자열 숫자를 정수형으로 변환

```python
df.withColumn("Age", col("Age").cast(IntegerType()))
```

### 문자열 날짜를 Date 타입으로 변환

```python
df.withColumn("Order_Date", to_date(col("OrderDate"), "dd-MM-yyyy"))
```

---

# # 🧩 **5. Spark Structured Streaming**

### 💡 Spark Structured Streaming은 *실시간 스트리밍 처리 프레임워크*

---

## ⚙️ Structured Streaming의 핵심 개념

### ① Micro-batch Model

* 실시간 데이터지만 내부적으로는 **마이크로 배치 단위로 처리**

### ② Event Time vs Processing Time

| 종류                  | 의미                 |
| ------------------- | ------------------ |
| **Event Time**      | 데이터가 실제 발생한 시간     |
| **Processing Time** | Spark가 데이터를 처리한 시간 |

시험에 자주 나오는 내용

---

### ③ Watermark (지연 이벤트 허용)

지연된 데이터가 늦게 들어올 것을 허용하는 시간 설정

```python
.withWatermark("timestamp", "2 minutes")
```

---

### ④ Window Function

1분 단위로 묶어 집계

```python
groupBy(window(col("timestamp"), "1 minute"))
```

---

# # 🧩 **6. Kafka + Spark Structured Streaming**

Kafka → Spark로 데이터가 들어오는 전체 흐름

```
[Kafka Producer] → [Kafka Broker] → [Spark Structured Streaming] → [Console / DB / File]
```

---

## 📌 Kafka Producer

* JSON 메시지를 Kafka topic으로 발행(send)

```python
producer.send("click-events", json)
```

---

## 📌 Spark에서 Kafka 읽기

```python
df = spark.readStream \
    .format("kafka") \
    .option("subscribe", "click-events") \
    .load()
```

---

## 📌 Kafka value 파싱

Kafka value는 **바이너리** → 문자열 변환 필요

```python
value_df = df.selectExpr("CAST(value AS STRING)")
```

---

## 📌 JSON 파싱 (from_json)

```python
parsed_df = value_df.select(
    from_json(col("value"), schema).alias("data")
).select("data.*")
```

---

## 📌 Window + GroupBy 집계

```python
result_df = parsed_df \
    .withWatermark("timestamp", "2 minutes") \
    .groupBy(
        window(col("timestamp"), "1 minute"),
        col("event_type")
    ) \
    .agg(
        count("*").alias("event_count"),
        approx_count_distinct("user_id").alias("unique_users")
    )
```

※ 시험 포인트 → 스트리밍에서는
`countDistinct` 대신 **`approx_count_distinct`** 사용해야 함 (정확한 카운트 불가)

---

## 📌 결과 출력 (writeStream)

```python
result_df.writeStream \
    .outputMode("complete") \
    .format("console") \
    .start() \
    .awaitTermination()
```

---

# # 🧩 **7. Kafka 서버 구성 흐름**

### ① Zookeeper 실행

```
bin/zookeeper-server-start.sh config/zookeeper.properties
```

### ② Kafka Broker 실행

```
bin/kafka-server-start.sh config/server.properties
```

### ③ Producer 실행

```
python kafka_producer.py
```

### ④ Spark Streaming 실행

```
python streaming_job.py
```

---

# # 🧩 **8. 시험에 잘 나오는 개념 정리 (암기 필수)**

### 🔥 Spark 개념

* In-memory processing
* Lazy evaluation
* DAG (Directed Acyclic Graph)
* Transformation vs Action

### 🔥 RDD

* Immutable
* Fault-tolerance (Lineage)
* map / filter / flatMap / mapPartitions
* sample / randomSplit

### 🔥 DataFrame

* 스키마 기반 구조 데이터
* Catalyst Optimizer
* DataFrame API vs SQL API

### 🔥 Structured Streaming

* Micro-batch
* Event time / processing time
* Watermark
* Window function
* outputMode("append" / "complete" / "update")

### 🔥 Kafka + Spark

* Kafka value는 Binary
  → CAST(value AS STRING) 필수
* from_json()으로 파싱
* 스트리밍에서는 approx_count_distinct 사용

