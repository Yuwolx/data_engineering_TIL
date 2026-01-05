
## 1. Elasticsearch는 뭐 하는 애인가?

* **검색/분석 엔진 + 문서 데이터베이스**
* 데이터를 **JSON 문서** 형태로 저장하고, 매우 빠르게 검색할 수 있게 해준다.
* 관계형 DB처럼 `테이블/행`이 아니라 **`인덱스(index)` / `문서(document)`** 개념을 쓴다.([Medium][1])

---

## 2. 기본 용어 정리

### 2-1. 인덱스(Index)

* 여러 문서(Document)가 들어 있는 **데이터 묶음(=DB의 테이블 비슷)**
* 예:

  * `products` 인덱스: 상품 정보 모음
  * `users` 인덱스: 사용자 정보 모음

파이썬 클라이언트에서 인덱스 존재 여부/생성은 이렇게 한다:([elasticsearch-py.readthedocs.io][2])

```python
es.indices.exists(index="products")
es.indices.create(index="products")
```

---

### 2-2. 문서(Document)와 ID

* 인덱스 안에 저장되는 **JSON 한 개**가 문서
* 각 문서는 고유한 `_id`를 가진다.
* 예: `products` 인덱스, ID `1` 문서

```json
{
  "product_name": "Samsung Galaxy S25",
  "brand": "Samsung",
  "price": 1199.99
}
```

---

### 2-3. `_source`

* 문서를 조회할 때 서버가 돌려주는 **실제 데이터 내용** 필드
* `GET /products/_doc/1` 하면 응답 안에 `_source`가 들어있고, 그 안에 우리가 넣었던 JSON이 그대로 있다.([GeeksforGeeks][3])

---

## 3. 단일 문서 API 3형제 (Index / Get / Update / Delete)

Elasticsearch에는 기본적으로 **단일 문서용 API**가 있다.([Medium][1])

1. **Index API** : 문서 생성/덮어쓰기
2. **Get API** : 문서 조회
3. **Update API** : 문서 일부 수정
4. **Delete API** : 문서 삭제

파이썬 클라이언트에서는 각각 이런 메서드로 사용:

* `es.index()` – 생성/덮어쓰기
* `es.get()` – 조회
* `es.update()` – 부분 업데이트
* `es.delete()` – 삭제([elasticsearch-py.readthedocs.io][2])

---

## 4. 파이썬 클라이언트로 연결하기

```python
from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")
```

* 여기서 `"http://localhost:9200"`는 **내 로컬에서 실행 중인 Elasticsearch 서버** 주소
* 앞으로 하는 모든 작업은 `es` 객체 메서드로 실행

---

## 5. 인덱스 생성 흐름

### 5-1. 인덱스 존재 여부 확인

```python
if not es.indices.exists(index="products"):
    es.indices.create(index="products")
```

* 없으면 `create`로 생성
* 있으면 그냥 사용

---

## 6. 문서 삽입 (Create / Index)

### 6-1. Index API 특징

* `index()`는 **“해당 ID에 문서를 저장”**하는 역할

  * 해당 ID가 없으면 → 새로 생성
  * 이미 있으면 → **덮어쓰기(교체)**([Medium][1])

### 6-2. 파이썬 코드

```python
doc = {
    "product_name": "Samsung Galaxy S25",
    "brand": "Samsung",
    "release_date": "2025-02-07",
    "price": 1199.99
}

res = es.index(index="products", id=1, document=doc)
print(res["result"])  # created 또는 updated
```

---

## 7. 문서 조회 (Read: Get API)

### 7-1. Get API 역할

* 특정 인덱스 + 특정 ID 문서를 가져오는 API
* 문서가 있으면 `found: true`, 없으면 `found: false` 반환([GeeksforGeeks][3])

### 7-2. 파이썬 코드

```python
from elasticsearch import NotFoundError

try:
    res = es.get(index="products", id=1)
    print(res["_source"])
except NotFoundError:
    print("문서가 존재하지 않습니다.")
```

---

## 8. 문서 수정 (Update API – 부분 업데이트)

### 8-1. Update API의 핵심 개념

* Elasticsearch의 **Update는 실제로는 내부적으로 “읽고 → 수정하고 → 다시 색인”**하는 구조다.([mindmajix][4])
* 그러나 우리가 쓸 때는 단순히 **“바꾸고 싶은 필드만 `doc`에 넣어 보내면 됨”**

예를 들어 가격만 바꾸고 싶을 때:

```json
POST /products/_update/1
{
  "doc": {
    "price": 1099.99
  }
}
```

이렇게 하면 나머지 필드는 그대로 두고 `price`만 바뀐다.([Elastic][5])

### 8-2. 파이썬 코드

```python
update_fields = {"price": 1099.99}

res = es.update(
    index="products",
    id=1,
    body={"doc": update_fields}
)
print(res["result"])  # "updated" 등
```

---

## 9. 문서 삭제 (Delete API)

### 9-1. Delete API 역할

* 인덱스에서 해당 ID의 문서를 삭제
* 삭제된 경우 응답에서 `"result": "deleted"` 같은 값을 준다.([GeeksforGeeks][3])

### 9-2. 파이썬 코드

```python
try:
    res = es.delete(index="products", id=1)
    print(res["result"])  # deleted
except NotFoundError:
    print("이미 문서가 없음")
```

삭제 후 다시 `get()`을 호출하면 `NotFoundError`가 발생하게 되고,
이를 통해 **문서가 진짜 없는 상태**임을 확인할 수 있다.

---

## 10. Upsert란? (Update + Insert)

이제 오늘의 핵심 개념.

### 10-1. Upsert 개념 자체

* **Upsert = Update + Insert**

  * 문서가 **있으면** → Update
  * 문서가 **없으면** → Insert (새 문서 생성)

엘라스틱에서는 이 동작을 **Update API + `doc_as_upsert: true` 옵션**으로 구현한다.([Packt][6])

---

### 10-2. 원래 Update의 기본 동작

* 기본적으로 `_update`는 **“문서가 이미 존재한다”**는 전제
* 문서가 없으면 **“document missing” 에러**가 난다.([mindmajix][4])

---

### 10-3. `doc_as_upsert: true` 옵션

* 의미:

  > “`doc`에 있는 내용을, 문서가 없으면 새 문서 내용으로 사용해라.”

* 즉,

  * 문서 없음 → `doc` 그대로 새 문서로 **생성 (`result: created`)**
  * 문서 있음 → `doc` 기준으로 **부분 업데이트 (`result: updated` or `noop`)**([Elastic][7])

#### 예시 (REST 형태)

```json
POST /products/_update/1
{
  "doc": {
    "price": 999.99
  },
  "doc_as_upsert": true
}
```

#### 동작:

1. ID 1 문서가 **없다면**
   → `price: 999.99` 하나만 가진 새 문서를 만들어준다.
2. ID 1 문서가 **이미 있다면**
   → 그 문서의 `price` 필드만 수정한다.

---

### 10-4. 파이썬 클라이언트에서 Upsert

문서가 있어도/없어도 아래 한 줄로 처리 가능:([elasticsearch-py.readthedocs.io][2])

```python
upsert_fields = {"price": 999.99}

res = es.update(
    index="products",
    id=1,
    body={
        "doc": upsert_fields,
        "doc_as_upsert": True
    }
)
print(res["result"])  # created / updated / noop
```

---

## 11. 오늘 실습 흐름을 개념으로 다시 정리하기

너가 작성한 메인 코드의 흐름을 개념으로 풀어보면:

1. **Elasticsearch 클라이언트 생성**
   → `es = Elasticsearch("http://localhost:9200")`

2. **인덱스 생성**

   * `products`가 없으면 `create_index()`에서 생성

3. **문서 삽입 (index)**

   * `insert_document()`로 `ID=1`에 상품 정보 저장
   * 여기까지가 기본 **Create**

4. **문서 조회 (get)**

   * `get_document()`로 `_source` 확인 → 기본 **Read**

5. **문서 수정 (update)**

   * `update_document()`로 `price`만 변경 → 부분 업데이트

6. **다시 조회 (get)**

   * 수정된 `price`가 반영되었는지 확인

7. **Upsert 호출 (문서 있을 때)**

   * `upsert_document()` 호출
   * 이미 문서가 있으므로 → **Update 로 동작**

8. **삭제 (delete)**

   * `delete_document()`로 `ID=1` 삭제 → **Delete**

9. **삭제 확인 (get)**

   * `NotFoundError` 또는 “문서 없음” 메시지

10. **Upsert 다시 호출 (문서 없을 때)**

    * 이번엔 문서가 없으므로
    * `doc_as_upsert: true`로 **새 문서 생성**

11. **최종 조회 (get)**

    * Upsert로 새로 생성된 문서 확인

→ 이 과정을 통해 **CRUD + Upsert 전체 사이클**을 한 번 다 돈 것.

---

## 12. 응답에서 자주 보게 되는 필드들

Update / Delete / Upsert 응답에서 자주 보이는 필드들:

* `_index`: 작업한 인덱스 이름
* `_id`: 문서 ID
* `_version`: 변경될 때마다 1씩 증가하는 버전 번호([GeeksforGeeks][3])
* `result`: 작업 결과

  * `"created"` : 새 문서 생성
  * `"updated"` : 기존 문서 변경
  * `"deleted"` : 삭제됨
  * `"noop"` : 업데이트 요청이 왔지만 실제로 바뀐 내용이 없음 (변화 X)([Discuss the Elastic Stack][8])

이 필드들만 읽어도 “내가 지금 뭘 한 건지” 디버깅하기가 훨씬 쉬워져.

---

## 13. 오늘 범위 핵심만 한 줄씩 요약

* **인덱스(index)**: 문서를 모아두는 논리적 공간 (DB 테이블 느낌)
* **문서(document)**: JSON 한 개, `_id`로 구분
* **Index API / `es.index()`**: 해당 ID의 문서를 생성/덮어쓰기
* **Get API / `es.get()`**: 해당 ID의 문서 내용 확인
* **Update API / `es.update()`**: `{"doc": {...}}`로 일부 필드만 수정
* **Delete API / `es.delete()`**: 문서를 제거
* **Upsert / `doc_as_upsert: true`**:

  * 문서 있으면 → 업데이트
  * 문서 없으면 → 새로 생성
* 파이썬에서는 이 모든 걸 `Elasticsearch` 클라이언트 메서드로 처리
