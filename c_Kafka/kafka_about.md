
# 🧠 Apache Kafka 학습 정리

## 📘 1. Kafka 기본 개념

### 🔹 Kafka란?

Apache Kafka는 **대용량 데이터를 실시간으로 처리하는 분산 메시징 시스템**이다.
Producer(생산자)가 메시지를 발행하면, Consumer(소비자)가 구독하여 메시지를 처리한다.

---

### 🔹 주요 구성 요소

| 구성요소          | 설명                                         |
| ------------- | ------------------------------------------ |
| **Broker**    | Kafka 서버. 메시지를 저장하고 관리하는 핵심 프로세스           |
| **Zookeeper** | 브로커의 상태, 클러스터 메타데이터를 관리 (Kafka 3.x 이후 선택적) |
| **Producer**  | 데이터를 Kafka 토픽으로 보내는 주체                     |
| **Consumer**  | Kafka에서 데이터를 읽어가는 주체                       |
| **Topic**     | 메시지가 저장되는 논리적 채널                           |
| **Partition** | 토픽의 실제 데이터 저장 단위 (병렬 처리 단위)                |
| **Offset**    | 파티션 내 메시지의 고유 번호 (읽기 위치를 나타냄)              |
| **Key**       | 특정 메시지를 어느 파티션에 보낼지를 결정하는 기준 값             |

---

## ⚙️ 2. Kafka 실행 및 환경 설정

### ✅ Zookeeper 실행

```bash
bin/zookeeper-server-start.sh config/zookeeper.properties > logs/zookeeper.log 2>&1 &
```

* 백그라운드 실행
* 로그: `logs/zookeeper.log`

### ✅ Kafka 브로커 실행

```bash
bin/kafka-server-start.sh config/server.properties > logs/kafka.log 2>&1 &
```

### ✅ 실행 상태 확인

```bash
ps -ef | grep kafka
ps -ef | grep zookeeper
```

---

## 🧩 3. 기본 명령어 실습

### ✅ 토픽 목록 확인

```bash
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### ✅ 토픽 생성

```bash
bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

### ✅ 토픽 상세 정보 확인

```bash
bin/kafka-topics.sh --describe --topic test-topic --bootstrap-server localhost:9092
```

---

## 💬 4. 메시지 송수신 테스트

### ✅ 프로듀서 실행 (메시지 전송)

```bash
bin/kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
```

메시지 입력 예:

```
Hello Kafka!
Another message
```

### ✅ 컨슈머 실행 (메시지 소비)

```bash
bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
```

---

## 🧠 5. 다중 토픽 실습

### ✅ 여러 개의 토픽 생성

```bash
bin/kafka-topics.sh --create --topic topic-1 --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
bin/kafka-topics.sh --create --topic topic-2 --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

### ✅ 메시지 전송 예시

```bash
bin/kafka-console-producer.sh --topic topic-1 --bootstrap-server localhost:9092
```

```
Message to topic-1
Another message for topic-1
```

```bash
bin/kafka-console-producer.sh --topic topic-2 --bootstrap-server localhost:9092
```

```
Message to topic-2
Another message for topic-2
```

### ✅ 컨슈머 확인

```bash
bin/kafka-console-consumer.sh --topic topic-1 --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic topic-2 --from-beginning --bootstrap-server localhost:9092
```

---

## 🔑 6. 파티션 & 키 실습

### ✅ 3개의 파티션을 가진 토픽 생성

```bash
bin/kafka-topics.sh --create --topic partition-test --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

### ✅ 특정 키로 메시지 전송

```bash
bin/kafka-console-producer.sh --topic partition-test --bootstrap-server localhost:9092 --property parse.key=true --property key.separator=:
```

입력 예시:

```
key1:Message from key1 - 1
key1:Message from key1 - 2
key2:Message from key2 - 1
```

### ✅ 키 없이 메시지 전송

```bash
bin/kafka-console-producer.sh --topic partition-test --bootstrap-server localhost:9092
```

입력 예시:

```
Message without key - 1
Message without key - 2
Message without key - 3
```

---

## 🔍 7. 파티션별 메시지 확인

| 명령어             | 설명            |
| --------------- | ------------- |
| `--partition 0` | 파티션 0번 메시지 확인 |
| `--partition 1` | 파티션 1번 메시지 확인 |
| `--partition 2` | 파티션 2번 메시지 확인 |

예시:

```bash
bin/kafka-console-consumer.sh --topic partition-test --partition 0 --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic partition-test --partition 1 --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic partition-test --partition 2 --from-beginning --bootstrap-server localhost:9092
```

**결과 분석:**

* `key1`, `key2` → 항상 동일한 파티션에 저장 (같은 해시 결과)
* 키 없는 메시지 → 라운드 로빈 방식으로 여러 파티션에 분배

---

## 🧹 8. 종료 및 로그 확인

### ✅ Kafka 종료

```bash
bin/kafka-server-stop.sh
```

### ✅ Zookeeper 종료

```bash
bin/zookeeper-server-stop.sh
```

### ✅ 로그 확인

```bash
cat logs/kafka.log | tail -n 50
```

---

## 🗂️ 9. 학습 핵심 요약

| 주제               | 핵심 개념                                    |
| ---------------- | ---------------------------------------- |
| **Kafka 구조**     | Broker(저장소), Topic(채널), Partition(병렬 단위) |
| **Zookeeper 역할** | 브로커 관리 및 메타데이터 유지                        |
| **Producer**     | 메시지를 특정 Topic에 전송                        |
| **Consumer**     | Topic의 메시지를 구독                           |
| **Key**          | 특정 파티션으로 메시지를 보내는 기준                     |
| **Offset**       | 파티션 내 메시지의 순번                            |
| **명령어 흐름**       | 실행 → 토픽 생성 → 전송 → 소비 → 종료                |
| **실습 포인트**       | 키가 있으면 한 파티션 고정, 없으면 분산                  |

---

## 📚 참고 흐름 요약 (실습 순서)

1️⃣ Zookeeper 실행
2️⃣ Kafka 브로커 실행
3️⃣ 토픽 생성 및 목록 확인
4️⃣ 프로듀서로 메시지 전송
5️⃣ 컨슈머로 메시지 수신
6️⃣ 여러 토픽 생성 및 테스트
7️⃣ 파티션 & 키 기반 메시지 분배 실험
8️⃣ Kafka 종료 및 로그 확인

---

## ✅ 마무리

> 이번 학습을 통해 Kafka의 구조적 이해와 메시지 흐름(생산 → 저장 → 소비)을 실습으로 직접 체험함.
> 특히 **파티션과 키의 개념**을 실험적으로 확인하며, 실제 데이터 스트리밍 시스템이 메시지를 어떻게 분배하고 관리하는지 파악함.
