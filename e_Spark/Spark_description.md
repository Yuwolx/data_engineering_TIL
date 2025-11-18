# 📘 Apache Spark 개념 + 오늘 실습 전체 정리

## 1. 🔥 Spark란 무엇인가?

**Spark는 대용량 데이터를 빠르게 처리하기 위한 분산 처리 엔진**
→ 여러 대의 컴퓨터(클러스터)를 한 번에 사용해서 데이터 처리 속도를 극대화함.

### ✔ 왜 빠를까?

* 내장 메모리(RAM)를 적극적으로 사용함 → **Disk I/O를 최소화**
* DAG(Directed Acyclic Graph) 기반 최적화된 실행 계획
* 분산 컴퓨팅(노드 여러 개 사용)

### ✔ 언제 Spark를 쓸까?

* 데이터가 너무 많아서 Pandas로는 버티기 어려울 때
* 실시간/대규모 배치 처리
* MLlib(머신러닝), SQL 처리, 스트리밍 처리까지 해야 할 때

---

## 2. 🧩 Spark 구성 요소

### Spark Core

→ RDD, DAG 실행 엔진, 스케줄러 등 Spark의 뼈대

### Spark SQL

→ DataFrame, Dataset 기반의 SQL 엔진
→ 우리가 실무에서 가장 많이 쓰는 모듈

### Spark Streaming

→ 실시간 데이터 스트림 처리 (Kafka와 자주 연결)

### MLlib

→ 분산 머신러닝 라이브러리

### GraphX

→ 대규모 그래프 처리

---

## 3. 🧱 RDD란? (오늘 실습의 핵심)

**RDD (Resilient Distributed Dataset)**
Spark에서 가장 기본적인 데이터 구조.
“분산된 리스트”라고 보면 됨.

### 🧩 특징

* 변경 불가능(Immutable)
* 클러스터 전체에 자동 분산 저장
* Transformations / Actions 기반으로 동작

### Transformations (변환)

→ `.map()`, `.filter()`, `.flatMap()` 등
→ 실행 계획만 만들고, 실제 실행되지 않음 (Lazy Execution)

### Actions (실행)

→ `.collect()`, `.count()`, `.take()`
→ 이 때 실제 계산이 Spark 클러스터에서 발생

---

## 4. 🚀 오늘 한 실습 정리

### ✔ 1. SparkSession / SparkContext 생성

```python
spark = SparkSession.builder.appName("Transformations").getOrCreate()
sc = spark.sparkContext

print("Spark version:", sc.version)
```

### ✔ 2. 숫자 데이터 생성 및 변환

#### 1~20 숫자 생성

```python
numbers = sc.parallelize(range(1, 21))
```

#### 데이터 확인

```python
numbers.collect()
```

#### 숫자 2배 변환

```python
doubled = numbers.map(lambda x: x * 2)
doubled.collect()
```

#### 10보다 큰 숫자 필터링

```python
greater_than_10 = numbers.filter(lambda x: x > 10)
greater_than_10.collect()
```

#### 숫자 개수 확인

```python
numbers.count()
greater_than_10.count()
```

---

## 5. 🔤 알파벳 데이터 변환

```python
alphabets = sc.parallelize(["A","B","C","D","E","F","G","H","I","J"])
```

#### 전체 출력

```python
alphabets.collect()
```

#### 두 번 반복

```python
repeated = alphabets.map(lambda x: x * 2)
```

#### "E" 이후 문자만 출력

```python
after_E = alphabets.filter(lambda x: x > "E")
```

#### 소문자로 변환

```python
lower = alphabets.map(lambda x: x.lower())
```

---

## 6. 🎲 랜덤 숫자 변환

```python
random_numbers = sc.parallelize([3,10,5,7,1])
```

#### 제곱

```python
squared = random_numbers.map(lambda x: x * x)
```

#### 10보다 큰 제곱 출력

```python
squared.filter(lambda x: x > 10)
```

---

## 7. 📄 텍스트 파일 로드 & 탐색

### 텍스트 파일 불러오기

```python
text_data = sc.textFile("../data/test_1.txt")
```

### 전체 내용 확인

```python
text_data.collect()
```

### 라인 수 확인

```python
text_data.count()
```

### “data” 포함된 줄만 필터링

```python
contains_data = text_data.filter(lambda x: "data" in x.lower())
```

### 대문자 변환

```python
upper_case = text_data.map(lambda x: x.upper())
```

### 소문자 변환

```python
lower_case = text_data.map(lambda x: x.lower())
```

---

## 8. 🧱 파티션(Partition) 개념

### ✔ 파티션이란?

Spark RDD가 나눠져 저장되는 **데이터 조각**
→ 많을수록 병렬 처리 성능 ↑
→ 너무 많으면 스케줄 오버헤드 ↑

### 파티션 개수 확인

```python
text_data.getNumPartitions()
```

### 파티션 재설정

```python
text_data.repartition(4)
```

---

# 🎯 오늘 배운 핵심 요약

* Spark는 **대규모 데이터 분산 처리**가 목적
* Pandas보다 훨씬 큰 데이터를 처리할 수 있음
* RDD는 Spark의 기본 데이터 구조
* Transformations는 즉시 실행되지 않고, Actions 호출 시 실행됨
* 텍스트 데이터 로드, 변환(map), 필터링(filter), 파티션 재설정까지 실습함
