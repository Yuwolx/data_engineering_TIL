
# 🚀 PySpark RDD 최적화 실습 정리

## ⭐ 학습 목표

오늘 학습의 핵심은 다음 3가지이다:

1. **map + filter**, **flatMap**, **mapPartitions**의 차이를 이해한다.
2. 1~100만 개의 숫자를 이용해 **RDD 연산 성능 차이**를 직접 실험한다.
3. Spark의 **파티션 개념**과 **데이터 처리 방식**을 경험한다.

---

# 🧱 1. 파티션(Partition)이란?

Spark RDD에서 파티션은 데이터를 쪼개어 **여러 노드에서 병렬 처리하기 위한 최소 단위**이다.

* 파티션이 많을수록 → 병렬 처리 증가
* 너무 많으면 → 스케줄링 오버헤드 증가
* 오늘 과제에서는 **파티션을 8개로 동일하게 유지**

```python
rdd = sc.parallelize(range(1, 1000001), numSlices=8)
```

즉, **100만 데이터를 8등분해서 처리**한 것.

---

# 🧮 2. 오늘의 실험 데이터: 1부터 1,000,000까지

```python
sc.parallelize(range(1, 1000001), 8)
```

고작 100만 개지만, 변환 방식에 따라 성능 차이가 꽤 크게 발생함.
이 실험으로 Spark의 **변환 방식 차이**를 정확히 체감할 수 있다.

---

# 🧪 3. 연산 시간 측정 함수

```python
def measure_time(fn):
    start = time.time()
    result = fn()
    end = time.time()
    return result, end - start
```

Spark는 **Lazy Execution(지연 실행)** 방식이므로,
실제로 연산이 수행되는 건 `collect()`, `count()` 같은 Action을 호출할 때이다.

이 함수는 Action이 포함된 lambda를 실행하여
**실제 연산 시간을 확인하기 위해 사용했다.**

---

# ⚙️ 4. map + filter 연산

### ✔ 개념

* **filter → 조건에 맞는 데이터만 유지**
* **map → 데이터를 변환**
  즉, 두 개의 transformation을 순차적으로 수행한다.

```python
rdd.filter(lambda x: x % 2 == 0)
   .map(lambda x: x * 2)
   .collect()
```

### ✔ 처리 방식

1. 먼저 짝수인지 검사하면서 100만 개 전체를 스캔
2. 그다음 결과를 다시 map에서 2배 변환
3. 파티션별로 처리: 총 8개의 파티션에서 수행
4. collect() 호출 시 최종 실행

### ✔ 특징

* 코드 가독성 좋음
* 하지만 **2개의 transformation 단계**가 필요 → 함수 호출 비용 증가 → 상대적으로 느림

---

# ⚡ 5. flatMap 연산

### ✔ 개념

flatMap은 원소 하나에서 **0개, 1개 또는 여러 개의 결과**를 반환할 수 있는 transformation.

이번 실습에서는:

```python
.flatMap(lambda x: [x*2] if x % 2 == 0 else [])
```

짝수면 `[값]`, 홀수면 `[]`를 반환
=> 아무것도 반환하지 않는 경우를 처리할 수 있다.

### ✔ 처리 방식

* filter + map을 **한 번에 처리**하는 구조
* 매 요소에 대해 “필터링 + 변환”을 동시에 수행

### ✔ 장점

* 함수 호출 횟수가 map+filter보다 적다
* 데이터 흐름이 단순해서 **map + filter보다 빠른 경우가 많다**

---

# 🚀 6. mapPartitions 연산 (가장 빠른 방식)

### ✔ 개념

**파티션 단위로 변환 작업을 수행**하는 함수.

즉, 데이터 하나씩 처리하는 것이 아니라:

* 파티션 전체(예: 12,500개씩 묶음)를 가져와
* 한 번의 Python 함수 호출로 처리함

```python
def transform_partition(iterator):
    return (x * 2 for x in iterator if x % 2 == 0)

rdd.mapPartitions(transform_partition)
```

### ✔ 처리 방식

1. Spark는 파티션 단위로 iterator를 넘겨주고
2. Python 함수는 한 번만 호출됨
3. iterator 내부에서 짝수만 2배로 변환
4. Python과 JVM 간의 오버헤드가 최소로 줄어듦

### ✔ 장점

* map/filter/flatMap보다 **훨씬 적은 함수 호출**
* 높은 성능
* 대규모 데이터 처리에 최적

### ✔ 단점

* 코드가 다소 복잡
* “함수 내부에서 파티션을 전부 읽음 → 메모리 부담 증가 가능”

---

# 📊 7. 최종 성능 비교 (이론적)

| 연산 방식             | 처리 방식              | 함수 호출 횟수 | 속도 경향 |
| ----------------- | ------------------ | -------- | ----- |
| **map + filter**  | 2번의 transformation | 많음       | 가장 느림 |
| **flatMap**       | 1번 transformation  | 중간       | 중간    |
| **mapPartitions** | 파티션 단위로 처리         | 매우 적음    | 가장 빠름 |

대규모 데이터일수록 이 차이는 커진다.

---

# 🧠 8. 왜 속도 차이가 날까?

Spark에서 transformation은 기본적으로 **Python 함수 호출 비용이 크다.**

정리하면:

* map/filter → 100만 번 함수 호출
* flatMap → 100만 번이지만 호출 횟수 1번
* mapPartitions → **8번** (파티션 개수만큼)

즉, Python 함수 호출 overhead를 줄이는 것이 핵심.

---

# 🎯 9. 오늘 배운 핵심 요약

* Spark RDD는 Lazy Execution 기반이므로 Action이 호출될 때 실행됨
* map + filter는 두 단계라서 느림
* flatMap은 한 번에 필터 + 변환 가능
* mapPartitions는 파티션 단위로 처리해 함수 호출 횟수가 극적으로 줄어듦
* 파티션 개수(8개)는 성능에 중요한 영향을 미침
* collect()로 Action이 호출되는 순간 전체 연산 실행됨


# 🎨 Spark RDD 처리 과정 ― 마크다운 그림으로 이해하기

---

## 1️⃣ 전체 구조: 원본 데이터 → 파티션 → 연산 → 결과

```
1 ~ 1,000,000 
         │
         ▼
+---------------------------+
|  Spark RDD (8 partitions) |
+---------------------------+
    │   │   │   │   │   │   │   │
    ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼
 [P1][P2][P3][P4][P5][P6][P7][P8]
    │
    ▼
 Transformation(map/filter...)
    │
    ▼
     Action(collect)
    │
    ▼
[Final Result]
```

Spark는 데이터를 이런 식으로 **파티션 단위로 쪼개서 병렬 처리**한다.

---

## 2️⃣ map + filter 수행 방식

**map과 filter가 따로따로 실행됨**

```
[Partition 1] → filter → map → 결과
[Partition 2] → filter → map → 결과
...
[Partition 8] → filter → map → 결과
```

구조적으로 보면:

```
(filter)
   │
   ▼
(map)
   │
   ▼
Output
```

### 함수 호출 횟수 (중요!)

```
filter → 1,000,000번 호출
map    → 500,000번 호출
총 약 150만번 호출 🔥
```

그래서 가장 느림.

---

## 3️⃣ flatMap 수행 방식

필터 + 변환을 **한 번에 처리**

```
flatMap: 
x → [2*x] 또는 []
```

그림으로 보면:

```
[Partition] 
     │
     ▼
(flatMap)  ──> [2*x] or []
     │
     ▼
Output
```

### 함수 호출 횟수

```
flatMap → 1,000,000번 호출
(단 한 번의 transformation)
```

map+filter를 합친 형태라 더 빠름.

---

## 4️⃣ mapPartitions 수행 방식 (최고속)

파티션 전체를 “묶음”으로 받아서 처리

```
mapPartitions(transform_partition):
    Input: iterator(12,500개 단위)
    Output: generator(필터+변환)
```

ASCII 그림:

```
[Partition 1: 125,000개]
        │
        ▼
(transform_partition)
        │
        ▼
[결과]

[Partition 2: 125,000개]
        │
        ▼
(transform_partition)
```

### 함수 호출 횟수

```
mapPartitions → 단 8번 호출 (파티션 수 만큼)
```

🔥 **압도적 성능 차이의 이유**
데이터 개수와 상관없이 “파티션 개수”만큼 함수가 호출됨 → 매우 빠름.

---

## 5️⃣ Lazy Execution 구조

Spark는 Action 호출 전까지 실행하지 않음.

```
rdd.filter(...)
   .map(...)
   .mapPartitions(...)

(아직 실행 ❌)

collect() 호출!
     │
     ▼
==== 실제 실행 시작 ====
```

그림으로 보면:

```
RDD
 │
 ├── filter (준비만 함)
 │
 ├── map (준비만 함)
 │
 └── mapPartitions (준비만 함)
 │
 ▼
 Action(collect) 실행 시 전체 DAG 한 번에 실행
```

---

## 6️⃣ 최종 비교 그림

### 성능 순위 그림

(왼쪽이 느림 → 오른쪽이 빠름)

```
map + filter  →  flatMap  →  mapPartitions
       (2단계)       (1단계)        (파티션 단위)
```

### 함수 호출 횟수 비교

```
map + filter : ~1,500,000번 호출
flatMap      : ~1,000,000번 호출
mapPartitions: ~8번 호출
```

차이가 완전히 눈에 보이지? 😎

---

# ✔ 전체 개념 총정리 ASCII 그림

```
+---------------------------------------------------------------+
|                           Spark RDD                           |
+---------------------------------------------------------------+
|  데이터 분할 → 파티션 단위 실행 → 지연 실행 → 액션 시 완전 실행 |
+---------------------------------------------------------------+

              map + filter           flatMap          mapPartitions
              -----------            --------         --------------
 데이터 → [filter] → [map]   데이터 → [flatMap]    데이터 → [partition 단위]
          (2 steps)                (1 step)          (8 partitions)

   함수 호출 많음 ❌         중간 정도 ⭕

                              함수 호출 극소량(8회) ⭕⭕⭕
```
