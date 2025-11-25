# 🧠 Kafka 실습 과제 종합 정리

## 📅 학습 목표

Kafka의 전체 동작 흐름을 이해하고,
**Producer → Topic → Consumer** 구조와
**스트림 변환(Transformation) 파이프라인**을 직접 구현한다.

---

## ⚙️ 1️⃣ Kafka 환경 설정 및 브로커 실행

```bash
# Zookeeper 실행
bin/zookeeper-server-start.sh config/zookeeper.properties > logs/zookeeper.log 2>&1 &

# Kafka 브로커 실행
bin/kafka-server-start.sh config/server.properties > logs/kafka.log 2>&1 &
```

**확인 명령어**

```bash
ps -ef | grep zookeeper
ps -ef | grep kafka
```

> 💡 두 프로세스가 정상적으로 떠 있다면 Kafka 서버가 실행 중입니다.

---

## 🧩 2️⃣ 토픽 생성 및 메시지 송수신 테스트

### ✅ 토픽 생성

```bash
bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

### ✅ 토픽 목록 확인

```bash
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### ✅ 메시지 전송 (Producer)

```bash
bin/kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
> hello
> kafka
> world
```

### ✅ 메시지 수신 (Consumer)

```bash
bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
```

> 📸 캡처: 메시지 송신 및 수신 결과 화면

---

## 🔀 3️⃣ Key 기반 파티션 분배 실습

### ✅ 동일 키로 메시지 전송

```bash
bin/kafka-console-producer.sh --topic partition-test \
--bootstrap-server localhost:9092 \
--property parse.key=true --property key.separator=:
> key1:message-1
> key1:message-2
> key2:message-3
```

Kafka는 내부적으로

```
partition = hash(key) % num_partitions
```

공식을 사용하여 파티션을 결정한다.

### ✅ 파티션별 메시지 확인

```bash
bin/kafka-console-consumer.sh --topic partition-test --partition 0 --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic partition-test --partition 1 --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic partition-test --partition 2 --from-beginning --bootstrap-server localhost:9092
```

---

## 🧮 4️⃣ Python: Kafka Producer & Consumer 실습

### ✅ Producer (토픽 관리 및 메시지 전송)

```python
from kafka import KafkaProducer
from kafka.admin import KafkaAdminClient, NewTopic
import time

admin = KafkaAdminClient(bootstrap_servers='localhost:9092', client_id='test-admin')
topic_name = "test-topic"

# 기존 토픽 삭제 후 재생성
if topic_name in admin.list_topics():
    admin.delete_topics([topic_name])
    time.sleep(2)
admin.create_topics([NewTopic(name=topic_name, num_partitions=3, replication_factor=1)])
admin.close()

# 메시지 전송
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    key_serializer=lambda k: k.encode('utf-8'),
    value_serializer=lambda v: v.encode('utf-8')
)

for i in range(10):
    key = f"key{i%3}"
    value = f"message-{i}"
    producer.send(topic_name, key=key, value=value)
    print(f"Sent: {value} with key: {key}")

producer.flush()
producer.close()
```

### ✅ Consumer (메시지 소비)

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "test-topic",
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    enable_auto_commit=True,
    group_id='test-consumer-group'
)

for msg in consumer:
    print(f"Received: {msg.value.decode()}, Key: {msg.key.decode() if msg.key else None}, Partition: {msg.partition}")
```

---

## 🔁 5️⃣ Kafka 스트림 파이프라인 구축 (오늘 핵심 과제)

### 📘 전체 목표

> `input-topic`에 메시지를 넣고
> Consumer가 이를 변환(예: 대문자)해서 `output-topic`으로 내보내고
> 최종 Consumer가 변환된 메시지를 읽는다.

---

## 🧩 (1) Input Producer

**→ input-topic과 output-topic 생성 + 테스트 메시지 전송**

```python
from kafka import KafkaProducer
from kafka.admin import KafkaAdminClient, NewTopic
import time

admin = KafkaAdminClient(bootstrap_servers='localhost:9092', client_id='input-producer-admin')
input_topic, output_topic = "input-topic", "output-topic"

# 기존 토픽 삭제 후 재생성
for t in [input_topic, output_topic]:
    if t in admin.list_topics():
        admin.delete_topics([t])
        time.sleep(2)
admin.create_topics([
    NewTopic(name=input_topic, num_partitions=1, replication_factor=1),
    NewTopic(name=output_topic, num_partitions=1, replication_factor=1)
])
admin.close()

# 메시지 전송
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: v.encode('utf-8')
)
for msg in ["hello", "world", "kafka", "streaming", "data"]:
    producer.send(input_topic, value=msg)
    print(f"Sent: {msg}")

producer.flush()
producer.close()
```

---

## ⚙️ (2) Transformer (Consumer + Producer)

**→ input-topic에서 메시지 소비 → 대문자 변환 → output-topic으로 전송**

```python
from kafka import KafkaConsumer, KafkaProducer

consumer = KafkaConsumer(
    "input-topic",
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    enable_auto_commit=True,
    group_id='transformer-group'
)

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: v.encode('utf-8')
)

for message in consumer:
    transformed = message.value.decode('utf-8').upper()
    producer.send("output-topic", value=transformed)
    print(f"Transformed and Sent: {transformed}")
```

---

## 👀 (3) Output Consumer

**→ output-topic에서 변환된 메시지 소비 및 출력**

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "output-topic",
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    enable_auto_commit=True,
    group_id='output-consumer-group'
)

for message in consumer:
    print(f"Consumed from output-topic: {message.value.decode('utf-8')}")
```

---

# 🧭 **Kafka 실시간 파이프라인 다이어그램**

```
         ┌────────────────────────────────────────────────────────────┐
         │                   ① Input Producer                         │
         │────────────────────────────────────────────────────────────│
         │ - input-topic 및 output-topic 생성                          │
         │ - 테스트 메시지 전송 ("hello", "world", ...)               │
         └───────────────┬────────────────────────────────────────────┘
                         │
                         ▼
         ┌────────────────────────────────────────────────────────────┐
         │                     Kafka Broker                           │
         │────────────────────────────────────────────────────────────│
         │   [input-topic]  → 원본 메시지 저장                        │
         │   [output-topic] → 변환된 메시지 저장                      │
         │   (Kafka는 메시지를 파티션 단위로 순서 보존하며 관리)     │
         └───────────────┬────────────────────────────────────────────┘
                         │
                         ▼
         ┌────────────────────────────────────────────────────────────┐
         │                 ② Transformer (중간 처리기)                │
         │────────────────────────────────────────────────────────────│
         │ - input-topic을 Consumer로 구독                            │
         │ - 메시지 수신 후 변환 (ex: .upper() → 대문자 변환)        │
         │ - output-topic으로 Producer 전송                           │
         └───────────────┬────────────────────────────────────────────┘
                         │
                         ▼
         ┌────────────────────────────────────────────────────────────┐
         │                ③ Output Consumer                           │
         │────────────────────────────────────────────────────────────│
         │ - output-topic 구독                                        │
         │ - 변환된 메시지 출력 (HELLO, WORLD, ...)                   │
         └────────────────────────────────────────────────────────────┘
```

---

# 🔁 **Kafka 내부 데이터 흐름**

```
Producer ----> [Kafka Broker: input-topic] ----> Transformer ----> [Kafka Broker: output-topic] ----> Consumer
```

| 동작                   | 메커니즘                | 설명                       |
| -------------------- | ------------------- | ------------------------ |
| Producer → Broker    | `send()`            | 메시지를 Kafka 토픽으로 푸시(push) |
| Transformer → Broker | `poll()` → `send()` | 메시지를 읽고(poll) 변환 후 다시 전송 |
| Consumer ← Broker    | `poll()`            | Kafka에서 새 메시지를 가져와 출력    |

---

# 💬 **핵심 개념 요약**

| 개념                    | 설명                                          |
| --------------------- | ------------------------------------------- |
| **Topic**             | Kafka의 메시지 저장소 (input-topic / output-topic) |
| **Producer**          | 메시지를 Kafka로 전송하는 역할                         |
| **Consumer**          | Kafka에서 메시지를 읽는 역할 (poll 방식)                |
| **Broker**            | 메시지를 저장 및 관리하는 Kafka 서버                     |
| **Partition**         | 토픽을 병렬 분할하여 처리 성능 향상                        |
| **Key**               | 메시지의 파티션 분배 기준                              |
| **Offset**            | 메시지 순서를 보장하는 고유 ID                          |
| **Stream Processing** | 실시간 데이터 변환 및 파이프라인 처리 구조                    |

---

# ✅ **실행 순서 요약**

1️⃣ **Kafka Broker / Zookeeper 실행**
2️⃣ **input_producer 실행 → input-topic에 데이터 적재**
3️⃣ **transformer 실행 → input-topic 메시지 변환 후 output-topic으로 송신**
4️⃣ **output_consumer 실행 → 변환된 데이터 확인**

---

# 🧾 **최종 출력 예시**

```
Consumed from output-topic: HELLO
Consumed from output-topic: WORLD
Consumed from output-topic: KAFKA
Consumed from output-topic: STREAMING
Consumed from output-topic: DATA
```

---

# 🎯 **결론**

이번 실습을 통해

> ✅ Kafka의 토픽 구조
> ✅ Producer / Consumer 기본 동작
> ✅ Key 기반 파티션 분배
> ✅ 스트림 변환 파이프라인 구현
> 을 전부 직접 실습하며 **Kafka 데이터 흐름의 핵심 메커니즘**을 완전히 이해했습니다.
