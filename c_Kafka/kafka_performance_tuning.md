# 🧭 Kafka 성능 실험 정리 노트

## 🗓 오늘의 주제

> Kafka의 **Producer**와 **Consumer**를 최적화하기 위한 성능 실험

이번 실습의 목표는 Kafka의 전송 구조를 이해하고,
설정값 변화가 메시지 처리 속도에 어떤 영향을 주는지 직접 체감하는 것이었다.

---

# 🚚 Part 1. Producer 성능 실험

## 🎯 목적

Kafka 프로듀서의 설정(batch, compression, linger)을 조정하여
동일한 메시지를 전송할 때의 처리 속도를 비교하고
가장 효율적인 조합을 찾아낸다.

---

## ⚙️ 실험 설정

| 설정 항목                | 설명                           | 영향              | 비유           |
| -------------------- | ---------------------------- | --------------- | ------------ |
| **batch.size**       | 한 번에 묶어서 전송하는 데이터 크기         | 크면 효율↑, 지연↑     | 📦 상자 크기     |
| **compression.type** | 메시지 압축 방식 (none/gzip/snappy) | 압축률과 속도에 영향     | 🗜️ 포장 방식    |
| **linger.ms**        | 메시지를 모으기 위해 기다리는 시간(ms)      | 기다릴수록 효율↑, 즉시성↓ | ⏳ 트럭 출발 대기시간 |

---

## 🧩 실험 코드 요약

```python
for batch in [16384, 32768, 65536]:
    for compression in ["none", "gzip", "snappy"]:
        for linger in [0, 10, 50]:
            producer = KafkaProducer(
                bootstrap_servers="localhost:9092",
                batch_size=batch,
                compression_type=compression if compression != "none" else None,
                linger_ms=linger,
                acks='all'
            )
            start = time.time()
            for _ in range(100000):
                producer.send("test-topic", MESSAGE_PAYLOAD)
            producer.flush()
            print(batch, compression, linger, time.time() - start)
```

---

## 🧠 결과 해석

* **batch.size**
  → 메시지를 더 많이 모아 한 번에 보낼수록 전송 효율이 높아짐.
* **compression.type**
  → `snappy`는 속도 빠름, `gzip`은 압축률 좋지만 CPU 부담 큼.
* **linger.ms**
  → 약간 기다리면 한 번에 전송 가능(효율↑), 즉시성은 떨어짐.

---

## 📈 예시 결과 (가상 수치)

| batch.size | compression | linger.ms | Time(sec) |
| ---------- | ----------- | --------- | --------- |
| 16384      | none        | 0         | 12.4      |
| 16384      | gzip        | 10        | 9.3       |
| 65536      | snappy      | 50        | 4.9       |

> ✅ 결론
> 큰 batch + snappy 압축 + 적당한 linger.ms
> → 전송 효율과 속도의 균형이 가장 좋다.

---

## 💬 정리

Kafka Producer는 택배를 보내는 사람과 같다.

* `batch.size`: 한 박스에 담을 물건의 양
* `compression.type`: 포장 방식
* `linger.ms`: 트럭이 출발하기 전 기다리는 시간

즉, **조금 기다려서 트럭을 꽉 채워 보내면 훨씬 효율적이다.**

---

# 📥 Part 2. Consumer 성능 실험

## 🎯 목적

Kafka Consumer의 설정(poll, fetch)을 조정해
동일한 개수의 메시지를 소비할 때 속도를 비교하고
최적의 조합을 도출한다.

---

## ⚙️ 실험 설정

| 설정 항목                 | 설명                       | 영향                | 비유                  |
| --------------------- | ------------------------ | ----------------- | ------------------- |
| **max.poll.records**  | 한 번의 poll에서 가져올 최대 메시지 수 | 많을수록 효율↑, 메모리 사용↑ | 🛒 카트에 실을 물건 개수     |
| **fetch.min.bytes**   | 최소 데이터 크기(이만큼 모이면 전송)    | 크면 효율↑, 지연↑       | 🚚 트럭 최소 적재량        |
| **fetch.max.wait.ms** | 기다릴 수 있는 최대 시간           | 크면 효율↑, 지연↑       | ⏳ 트럭이 떠나기 전 최대 대기시간 |

---

## 🧩 실험 코드 요약

```python
for poll in [10, 100, 500]:
    for fetch_min in [1024, 10240, 51200]:
        for fetch_wait in [100, 500, 1000]:
            consumer = KafkaConsumer(
                "test-topic",
                bootstrap_servers="localhost:9092",
                auto_offset_reset="earliest",
                enable_auto_commit=False,
                max_poll_records=poll,
                fetch_min_bytes=fetch_min,
                fetch_max_wait_ms=fetch_wait
            )
            start = time.time()
            count = 0
            for msg in consumer:
                count += 1
                if count >= 100000:
                    break
            print(poll, fetch_min, fetch_wait, time.time() - start)
```

---

## 🧠 결과 해석

* **max.poll.records**
  → 한 번에 더 많이 가져오면 poll 호출 수 감소 → 효율↑
* **fetch.min.bytes**
  → 브로커가 데이터가 모일 때까지 기다렸다가 보내므로 효율↑, 지연↑
* **fetch.max.wait.ms**
  → 기다릴 시간 한도를 늘리면 효율↑, 실시간성↓

---

## 📈 예시 결과 (가상 수치)

| max.poll.records | fetch.min.bytes | fetch.max.wait.ms | Time(sec) |
| ---------------: | --------------: | ----------------: | --------: |
|               10 |             1KB |               100 |      13.2 |
|              100 |            10KB |               500 |       7.1 |
|              500 |            50KB |               500 |       5.5 |

> ✅ 결론
> 많은 양을 한 번에 가져오고(fetch.min.bytes ↑),
> 조금 기다릴(fetch.max.wait.ms ↑) 여유를 주면
> **전체 처리 효율이 향상된다.**

---

## 💬 정리

Consumer는 창고에서 물건을 꺼내오는 사람과 같다.

* `max.poll.records`: 한 번에 실을 상자 수
* `fetch.min.bytes`: 트럭이 실릴 최소한의 양
* `fetch.max.wait.ms`: 떠나기 전 기다리는 시간

즉, **너무 자주 조금씩 꺼내기보다, 적당히 기다려 한 번에 많이 가져오는 게 효율적이다.**

---

# 🔁 전체 흐름 요약

```
Producer → [Kafka Broker] → Consumer
```

* **Producer**: 데이터를 얼마나 모아서 보낼까?
* **Consumer**: 데이터를 얼마나 모아서 받을까?
* 결국 Kafka는 **“덜 자주, 더 많이” 전송하는 것이 핵심**이다.

---

# 🧩 종합 결론

| 관점           | 핵심 전략                                                               | 이유                 |
| ------------ | ------------------------------------------------------------------- | ------------------ |
| **Producer** | 큰 `batch.size`, 빠른 `snappy`, 적절한 `linger.ms`                        | 전송 효율 극대화          |
| **Consumer** | 높은 `fetch.min.bytes`, 적당한 `fetch.max.wait.ms`, 큰 `max.poll.records` | 소비 효율 극대화          |
| **공통**       | “자주 보내지 말고, 모아서 보내라”                                                | 지연보다 처리량이 중요할 때 유리 |

---

# 💡 한 줄로 요약

> Kafka 성능의 핵심은
> **“조금 기다려서, 더 많이, 한 번에 처리하는 것”**

