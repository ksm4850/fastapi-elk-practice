## CDC(Change Data Capture)란

[위키백과](https://ko.wikipedia.org/wiki/%EB%B3%80%EA%B2%BD_%EB%8D%B0%EC%9D%B4%ED%84%B0_%EC%BA%A1%EC%B2%98)

데이터베이스에서 변경 데이터 캡처(change data capture, CDC)는 변경된 데이터를 사용하여 동작을 취할 수 있도록 데이터를 결정하고 추적하기 위해 사용되는 여러 소프트웨어 디자인 패턴들의 모임이다.

CDC는 기업 데이터 소스에 이루어지는 변경사항의 식별, 포착, 전송에 기반한 데이터 통합의 접근을 말한다.

CDC는 데이터 웨어하우스 환경에서 주로 발생하는데, 그 이유는 시간에 걸쳐 데이터 상태를 포착하고 보존하는 일이 데이터 웨어하우스의 핵심 기능 가운데 하나이기 때문이다. 그러나 CDC는 모든 데이터베이스, 데이터 저장소 시스템에서 활용이 가능하다.

## PostgreSQL의 WAL이란

[참고](https://tmaxtibero.blog/postgreql-wal/)

- PostgreSQL은 모든 데이터 변경 작업(INSERT, UPDATE, DELETE)을 **WAL(Write-Ahead Log)** 이라는 로그에 먼저 기록합니다.
- WAL은 장애 발생 시 복구를 위해 존재합니다. (데이터베이스가 실제 데이터 반영 전에 로그로 먼저 남김)
- **이걸 Debezium이 활용해서 CDC를 함**

PostgreSQL WAL 설정

```
-c wal_level=logical
-c max_replication_slots=4
-c max_wal_senders=4
```

`-c wal_level=logical`

- WAL 레벨을 `logical`로 설정
- 기본은 `replica`인데, `logical`은 Debezium같은 CDC 도구에서 필요한 포맷으로 WAL로그를 생성함
- `logical`설정을 해야 논리적 replication slot 및 출판(publication) 기능 사용 가능
- [참고](https://kimdubi.github.io/postgresql/pg_wal_level/)

`-c max_replication_slots=4`

- 동시에 유지할 수 있는 replication slot의 갯수 설정

`-c max_wal_senders=4`

- WAL로그를 외부로 송신할 수 있는 프로세스의 수

## 흐름

```
[사용자 데이터 변경] → [PostgreSQL WAL 기록] → [Debezium이 WAL 구독] → [Kafka로 전송] → [다른 시스템으로 전파]
```

## postgres connector plugin 옵션
[플러그인다운](https://www.confluent.io/hub/debezium/debezium-connector-postgresql)
```json
# postgres connector plugin 옵션
{
  "name": "pg-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "1234",
    "database.dbname": "postgres",
    "database.server.name": "pgserver",
    "table.include.list": "public.test_table",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_publication",
    "topic.prefix": "pg",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.add.fields": "op,ts_ms",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false"
  }
}
```

<details>
<summary>각 옵션별 설명</summary>
<div markdown="1">
`"connector.class": "io.debezium.connector.postgresql.PostgresConnector"`

- 사용하려는 커넥터 클래스

`"tasks.max": "1"`

- 최대 병렬 작업(Task) 개수

- 1로 설정하면 한 작업만 실행 (보통 단일 데이터베이스 커넥터는 1로 설정)

```
"database.hostname": "postgres"

`"database.port": "5432"

`"database.user": "postgres"

`"database.password": "1234"

`"database.dbname": "postgres"
```

- db 정보

`"database.server.name": "pgserver"`

- Kafka 토픽 이름 생성 시 접두어 역할

- 예) "pgserver.public.test_table" 와 같은 토픽 이름 생성

`"table.include.list": "public.test_table"`

- CDC를 적용할 테이블 목록 (스키마명.테이블명 형식)
- 지정한 테이블만 변경 감지(여러개 추가시 ,(콤마) 추가)

`"plugin.name": "pgoutput"`

- PostgreSQL의 Logical Decoding 플러그인 이름
- PostgreSQL 10 이상은 기본적으로 pgoutput 사용

`"slot.name": "debezium_slot"`

- PostgreSQL에서 사용되는 Logical Replication Slot 이름
- CDC 상태 유지용

`"publication.name": "debezium_publication"`

- PostgreSQL의 Publication 이름
- Logical Replication을 위해 생성한 Publication 이름과 동일해야 함

`"topic.prefix": "pg"`

- Kafka 토픽 이름 앞에 붙는 접두어
- 예) "pg.public.test_table" 토픽 생성

`"transforms": "unwrap"`

- 메시지 변환(transform) 설정 이름
- 아래에서 unwrap 변환을 사용함을 의미

`"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"`

- Debezium 메시지에서 실제 변경된 데이터만 추출하는 변환 클래스
- 기본 메시지 구조에서 envelope를 벗겨내 실제 레코드 상태로 변환

`"transforms.unwrap.add.fields": "op,ts_ms"`

- 언랩 변환 시 추가로 포함할 필드 지정
- op: 작업 유형 (c: create, u: update, d: delete)
- ts_ms: 변경 발생 타임스탬프 (밀리초 단위)

`"key.converter": "org.apache.kafka.connect.json.JsonConverter"`

- Kafka 메시지 키 변환기 지정 (여기선 JSON 포맷 사용)

`"value.converter": "org.apache.kafka.connect.json.JsonConverter"`

- Kafka 메시지 값 변환기 지정 (여기선 JSON 포맷 사용)

`"key.converter.schemas.enable": "false"`

- Kafka 키에 스키마 포함 여부 (false면 포함 안함)

`"value.converter.schemas.enable": "false"`

- Kafka 값에 스키마 포함 여부 (false면 포함 안함)
</div>
</details>
<br>

## ElasticSearch sink connector plugin 옵션
[플러그인다운](https://www.confluent.io/hub/confluentinc/kafka-connect-elasticsearch)
```json
{
  "name": "es-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "topics": "pgserver.public.test_table",
    "connection.url": "http://elasticsearch:9200",
    "key.ignore": "true",
    "schema.ignore": "true",
    "behavior.on.null.values": "delete"
  }
}
```

<details>
<summary>각 옵션별 설명</summary>
<div markdown="1">

`"connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector"`

- 사용할 커넥터 클래스

`"topics": "pgserver.public.test_table"`

- Kafka로부터 데이터를 가져올 토픽 이름
- 이 토픽의 메시지를 Elasticsearch에서 씀

`"connection.url": "http://elasticsearch:9200"`

- 연결할 Elasticsearch 서버 url

`"key.ignore": "true"`

- Kafka 메시지의 key를 무시하고 value만 사용. Elasticsearch에 문서를 저장할 때 ID로 key를 쓰지 않겠다는 의미.

`"schema.ignore": "true"`

- Kafka 메시지의 스키마 정보를 무시하고, 단순한 JSON으로 처리. (Avro, Schema Registry 등을 쓰지 않을 때 유용)

`"behavior.on.null.values": "delete"`

- Kafka 메시지의 value가 null일 경우, Elasticsearch에서 해당 document를 삭제.
- 즉, Debezium에서 tombstone 메시지나 delete 이벤트가 왔을 때 이를 삭제로 처리.

</div>
</details>
<br>
