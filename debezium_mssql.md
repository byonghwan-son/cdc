# debezium SQL SERVER

* 최초 연결시 스냅샷 생성
* 초기 스냅샷을 기초로 CDC에 대한 활성화된 Sql Server DB에 Insert, Update, Delete 또는 작업에 대한 행수준 변경사항을 지속적으로 캡쳐
* Connector는 각 데이터 변경 작업에 대한 이벤트를 생성하고 이를 Kafka Topic으로 스트리밍함.

## 개요

* SQL Server 2016 서비스 팩 1(SP1) 이상
* 테이블과 DB에 각각 CDC 설정을 해야 함.
* 데이터베이스 로그(LSN-Log Sequence Number)에 이벤트의 위치를 주기적으로 기록함.
* 커넥터가 통신 장애, 네트워크 문제 또는 충돌 이유로 인해 중지된 경우 재시작 후 마지막 읽은 지점에서 다시 읽기 시작.

### SQL 서버 커넥터의 작동 방식

* 커넥터의 스냅샷 수행
* 이벤트 스트리밍
* Kafka 토픽 이름 결정
* 메타 데이터 사용 방법

#### 스냅샷

* 현재 상태에 대한 기준선을 설정
* 두가지 유형의 정보를 캡쳐
  * 테이블 데이터
    * INSERT, UPDATE, DELETE 작업 정보
  * 스키마 데이터
    * 테이블에 적용되는 구조적 변경 사항을 설명하는 DDL 문
    * **Internal schema history topic**
    * **Connector's schema change topic(구성된 경우)**

<u>Debezium SQL Server 커넥터가 초기 스냅샷을 수행하는 데 사용하는 기본 워크플로</u>

* `snapshot.mode`: "initial"
  * 구조와 값을 가져 온다.

1. 데이터 베이스에 대한 연결 설정
2. 캡쳐할 테이블을 결정. 기본적으로 시스템 테이블을 제외한 모든 테이블이 대상.
   1. `table.include.list`: "포함할 <테이블 명, 테이블 명, ...>"
   2. `table.exclude.list`: "제외할 <테이블 명, 테이블 명, ...>"
3. 스냅샷 도중 데이터 구조적 변경이 발생하지 않도록 테이블 락 적용.
   1. `snapshot.isolation.mode` 에 따라 잠금 수준이 정해짐.
4. 서버의 트랜잭션 로그에서 최대 LSN(Log Sequence Number) 위치 읽기.
5. 캡쳐용으로 지정된 모든 테이블의 구조를 캡쳐. 내부 DB 스키마 기록 항목에 이 정보를 유지. 변경 이벤트가 발생할 때 비교할 기준이 되어 준다. <br>

   ```sql
    기본적으로 커넥터는 캡쳐용으로 구성되지 않은 테이블을 포함하여
    캡쳐 모드에 있는 데이터베이스의 모든 테이블 스키마(구조)를 캡쳐한다.
   ```

6. 3단계에서 얻는 잠금을 해제. 타 DB 클라이언트는 잠금 해제된 테이블 사용 가능.
7. 4단계에서 획득한 LSN 으로 캡쳐할 데이터를 스캔. 스캔 중에 커넥터는 다음 작업을 실행.
   1. 스냅샷을 하기 전에 테이블이 생성되었는지 확인. 시작된 후 생성된 테이블은 무시. 스냅샷 완료 후 커넥터가 스트리밍으로 전환되면 스냅샷이 시작된 후 모든 테이블에 대한 변경 이벤트를 내보냄.
   2. read 테이블에서 캡쳐된 각 행에 대한 이벤트를 생성. Read 이벤트에는 동일한 LSN 위치가 포함되어 있음. (4단계에서 얻는 LSN)
   3. read 테이블의 Kafka Topic에 각 이벤트를 전송.
8. 커넥터 오프셋에 스냅샷의 성공적인 완료를 기록

추가 정보

* `schema.history.internal.store.only.captured.tables.ddl` : 스키마 정보를 캡처할 테이블을 지정하는 속성을 설정. (true/false)
* `schema.history.internal.store.only.captured.databases.ddl` : 스키마 변경 사항을 캡처할 논리적 데이터베이스를 지정하는 속성을 설정. (true/false)

<U>초기 스냅샷에서 캡쳐되지 않은 테이블에서 데이터 캡쳐(스키마 변경 없음)</U>

테이블 스키마가 History Topic에 없는 경우 커넥터는 테이블 캡쳐를 실하고 누락된 스키마 오류가 발생.
테이블에서 데이터 갭쳐가 가능하지만 테이블 스키마를 추가하려면 추가 단계를 수행해야 함.

전제 조건

* 커넥터가 초기 스냅샷 중에 캡쳐하지 않은 스키마가 있는 테이블에서 데이터를 캡쳐하려고 한다.
* 커넥터가 읽는 테이블 항목과 LSN 사이에 간극이 발생할 경우

절차

1. 커넥터를 중지
2. schema.history.internal.kafka.topic 프로퍼티에서 지정하는 내부 데이터베이스 스키마 항목을 제거
3. offset.storage.topic에 구성된 kafka connect에서 offset 삭제
4. 커넥터 구성에 다음 변경 사항을 적용하기
   1. schema.history.internal.captured.tables.ddl을 false로 설정. 스냅샷이 모든 테이블에 대한 스키마를 캡쳐하고 나중에 커넥터가 모든 테이블에 대한 스키마 기록을 재구성할 수 있도록 보장.
   2. table.include.list에 커넥터가 캡쳐할 테이블을 추가
   3. snapshot.mode의 값 설정
      1. initial <br> 스키마와 데이터 모두. 데이터 베이스의 전체 스냅샷이 생성됨. <br>
         `schema.history.internal.captured.tables.ddl` : false
      2. schema_only <br> 스키마만 캡쳐. 빠르게 커넥터를 다시 시작할 때 사용.
5. 커넥터를 다시 시작. snapshot.mode에서 지정한 스냅샷 유형을 완성함.
6. (옵션) 커넥터가 schema_only 스냅샷으로 수행한 경우 스냅샷 완료 후에 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작하여 추가한 테이블에서 데이터를 캡쳐. 커넥터는 테이블에서 실시간 변경 사항을 계속 스트리밍하면서 스냅샷을 실행.
   * 캡쳐되는 데이터 변경 사항
     * 커넥터가 이전에 캡처한 테이블의 경우 증분 스냅샷은 커넥터가 작동 중지된 동안, 즉 커넥터가 중지된 시간과 현재 다시 시작 사이의 간격에서 발생하는 변경 사항을 캡처
     * 새로 추가된 테이블의 경우 증분 스냅샷은 기존 테이블 행을 모두 캡처

<U>초기 스냅샷에서 캡쳐되지 않은 테이블에서 데이터 캡쳐(스키마 변경)</U>

* 스키마가 적용된 테이블의 레코드는 변경전의 레코드 구조와 다름. 이를 위해 테이블에서 데이터를 캡쳐할 때마다 스키마 정보를 읽어와서 올바른 스키마를 적용하는지 확인. Schema History Topic에 스키마가 없으면 커넥터가 오류를 발생시킴.
* 초기 스냅샷에서 캡쳐되지 않은 테이블에서 데이터를 캡쳐하려고 하고 테이블의 스키마가 수정된 경우, 아직 사용할 수 없는 스키마를 History Topic에 추가해야 함. 

전제 조건

* 커넥터가 초기 스냅샷 중에 캡쳐하지 않은 스키마가 있는 테이블에서 데이터를 캡쳐하려고 함.
* 캡처할 레코드가 균일한 구조를 가지지 않도록 테이블 스키마 변경이 적용됨.
  * 변경 전 레코드와 변경 후 레코드의 구조가 다름
  
절차

*초기 스냅샷이 모든 테이블의 스키마를 캡쳐했음.(**`store.only.captured.tables.ddl` : false**)*

1. 캡쳐할 테이블을 지정하려면 `table.include.list`의 속성을 편집
2. 커넥터를 다시 시작
3. 새로 추가된 테이블에서 기존 데이터를 캡쳐하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작

*초기 스냅샷이 모든 테이블의 스키마를 캡쳐하지 못했음.(**store.only.captured.tables.ddl : true**)*

초기 스냅샷이 캡쳐하려는 테이블의 스키마를 저장하지 않은 경우

절차 1: 스키마 스냅샷에 이어진 증분 스냅샷

1. 커넥터를 중지
2. `schema.history.internal.kafka.topic`속성에서 지정하는 내부 데이터베이스 Schema History Topic을 제거
3. Kafka Connect의 `offset.storage.topic`에 구성된 오프셋을 지움. 오프셋 제거 방법은 [카프카커넥터](kafka_connect.md)를 참조
4. 다음 단계에 설명된 대로 커넥터 구성의 속성값을 설정
   1. `snapshot.mode` 속성값을 `schema_only`로 변경
   2. 캡쳐하려는 테이블을 추가하려면 `table.include.list`를 편집
5. 커넥터를 다시 시작
6. Debezium이 새 테이블과 기존 테이블의 스키마를 캡쳐할 때까지 기다림. 커넥터가 중지된 후 테이블에서 발생한 데이터 변경 사항은 캡쳐되지 않음(7번 항목에서 처리).
7. 데이터가 손실되지 않도록 하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작함.

절차 2 : 초기 스냅샷에 이어 선택적 증분 스냅샷

이 절차에서 커넥터는 데이터베이스의 전체 초기 스냅샷을 수행. 모든 초기 스냅샷과 마찬가지로 대규모 테이블이 많은 데이터베이스에서는 초기 스냅샷을 실행하는 데 시간이 많이 걸릴 수 있음. 스냅샷이 완료된 후 선택적으로 증분 스냅샷을 트리거하여 커넥터가 오프라인인 동안 발생하는 모든 변경 사항을 캡처할 수 있음.

1. 커넥터 중지
2. `schema.history.internal.kafka.topic`속성에서 지정하는 내부 데이터베이스 Schema History Topic을 제거
3. Kafka Connect의 `offset.storage.topic`에 구성된 오프셋을 지움. 오프셋 제거 방법은 [카프카커넥터](kafka_connect.md)를 참조
4. 캡쳐하려는 테이블을 추가하려면 `table.include.list`를 편집.
5. 다음 단계에 설명된 대로 커넥터 구성의 속성 값을 설정
   1. 속성값 `snapshot.mode`를 `initial`로 설정
   2. (선택사항) `schema.history.internal.store.only.captured.tables.ddl`을 `false`로 설정
6. 커넥터를 다시 시작. 커넥터는 전체 데이터베이스 스냅샷을 생성. 스냅샷이 완료되면 커넥터가 스트리밍으로 전환.
7. (선택사항) 커넥터가 오프라인인 동안 변경된 데이터를 캡쳐하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작함.

#### 임시 스냅샷(adhoc snapshot)

기본적으로 커넥터는 처음 시작된 후에만 초기 스냅샷 작업을 실행함. 이 초기 스냅샷 이후에 정상적인 상황에서는 커넥터가 스냅샷 프로세스를 반복하지 않음. 향후 커넥터가 캡쳐하는 변경 데이터 이벤트는 스트리밍 프로세스를 통해서만 제공됨.

일부 상황에서는 커넥터가 가져온 초기 스냅샷 데이터가 오래되거나 손실되거나 불완전해질 수 있음. 테이블 데이터를 다시 캡쳐하는 메커니즘을 제공하기 위해 ***임시 스냅샷***을 수행하는 옵션이 있음.

전제 조건

* 커넥터 구성이 다른 테이블 세트를 캡쳐하도록 수정되었음.
* Kafka Topic이 삭제되었으므로 다시 빌드해야 함.
* 구성 오류 또는 기타 문제로 인해 데이터 손상이 발생함.

임시 스냅샷으로 이전에 스냅샷을 캡쳐한 테이블에 대에 스냅샷을 <u>다시 실행</u>할 수 있음. 임시 스냅샷에는 [신호 테이블](https://debezium.io/documentation/reference/stable/configuration/signalling.html#sending-signals-to-a-debezium-connector)을 사용해야 함. Debezium **Signaling Table**에 신호 요청을 보내 임시 스냅샷을 시작함.

기존 테이블에 임시 스냅샷을 시작하면 해당 테이블을 위해 이미 존재하는 Topic에 콘텐츠를 추가. 혹시 Topic이 제거 된 상태에서 [Automatic Topic Creation](https://debezium.io/documentation/reference/stable/configuration/topic-auto-create-config.html#customizing-debezium-automatically-created-topics)이 활성화되어 있다면 Debezium은 자동으로 Topic을 생성함.

임시 스냅샷 Signal에는 스냅샷에 포함할 테이블을 명시. 스냅샷은 데이터베이스의 전체 내용을 캡쳐하거나 테이블의 하위 집합만 캡쳐할 수 있음. 테이블 내용의 하위 집합을 캡쳐할 수 있음.

Signal 테이블에 `execute-snapshot` 메시지를 보내 캡쳐할 테이블을 지정함. `execute-snapshot` Signal의 type을 `incremental` 혹은 `blocking`으로 설정하고 스냅샷에 포함할 테이블의 이름을 작성함.

[임시 `execute-snapshot` signal 행의 필드 설명]
|Field|Default|Value|
|-|-|-|
|type|incremental<br>증분|실행하려는 스냅샷 유형을 지정<br>incremental, blocking|
|data-<br>collections|N/A<br>해당없음|스냅샷을 생성할 테이블의 정규화된 이름과 일치하는 정규식이 포함된 배열<br>`signal.data.collection` 구성 옵션과 동일|
|additional-<br>conditions|N/A<br>해당없음|스냅샷에 포함할 레코드(행)의 하위 집합을 결정하기 위해 커넥터가 평가하는 추가 조건 집합을 지정하는 선택적 배열<br>각 추가 조건은 임시 스냅샷이 캡쳐하는 데이터를 필터링하기 위한 기준을 지정하는 개체<br>각 추가 조건에 대해 다음 매개변수를 설정<br><br>*data-collection*<br>필터가 적용된 정규화된 이름. 각 테이블에 서로 다른 필터를 적용할 수 있음.<br><br>*filter*<br>스냅샷에 포함하기 위해 데이터베이스 레코드에 있어야 할 열 값을 지정<br>예) "color='blue'"<br><br>필터 매개변수에 할당하는 값은 blocking 스냅샷에 대한 `snapshot.select.statement.overrides` 속성을 설정할 때 SELECT 문의 WHERE 절에 지정할 수 있는 값과 동일한 유형.|
|surrogate-<br>key|N/A<br>해당없음|스냅샷 프로세스 중에 커넥터가 테이블의 기본 키로 사용하는 열 이름을 지정하는 선택적 문자열|

<u>임시 증분(incremental) 스냅샷 트리거</u>

`execute-snapshot` signal type을 Signal Table에 추가하여 임시 증분 스냅샷을 시작. 커넥터는 처리중인 메시지가 있다면 메시지를 처리하고 스냅샷 작업을 시작. 스냅샷 프로세스는 첫번째 및 마지막 키 값을 읽고 해당 값을 각 테이블의 시작과 끝 지점으로 사용. Debezium은 테이블의 항목 수와 구성된 청크 크기에 따라 테이블을 청크로 나누고 각 청크를 한 번에 하나씩 스냅샷 처리함.

<u>임시 블록(blocking) 스냅샷 트리거</u>

`execute-snapshot` signal type을 Signal Table에 추가하여 임시 블록(Blocking) 스냅샷을 시작. 커넥터는 처리중인 메시지가 있다면 메시지를 처리하고 스냅샷 작업을 시작. 커넥터는 일시적으로 스트리밍을 중지한 다음 초그 스냅샷 중에 사용하는 것과 동일한 프로세스에 따라 지정된 테이블의 스냅샷을 시작함. 스냅샷이 완료되면 커넥터가 스트리밍을 재개함.

#### 증분 스냅샷(Incremental Snapshot)

Debezium에는 스냅샷 관리에 유연성을 제공하기 위해 증분 스냅샷이라는 스냅샷 메커니즘이 포함되어 있음. [증분 스냅샷은 Debezium 커넥터에 신호를 보내기](https://debezium.io/documentation/reference/stable/configuration/signalling.html#sending-signals-to-a-debezium-connector) 위해 Debezium 자체의 매커니즘을 사용.

Debezium은 증분 스냅샷에서 구성 가능한 일련의 청크로 각 테이블을 단계적으로 캡쳐. 스냅샷으로 캡처할 테이블과 [각 청크의 크기](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-property-incremental-snapshot-chunk-size)를 지정할 수 있음. 청크 크기는 데이터 베이스에서 각 가져오기 작업 중에 스냅샷이 수집하는 행 수를 결정함. 기본 청크 크기는 1024개의 행.

증분 스냅샷이 진행됨에 따라 워터마크(해시값)를 사용하여 진행 상황을 추적하고 캡쳐하는 각 테이블 행의 기록을 유지. 데이터 캡쳐에 대한 이러한 단계별 접근 방식은 다음과 같은 이점을 제공함.

* 스냅샷이 완료될 때까지 스트리밍을 연기하는 대신 스트리밍 데이터 캡쳐와 증분 스냅샷을 병렬로 실행할 수 있음. 커넥터는 스냅샷 프로세스 전반에 결쳐 변경 로그에서 거의 실시간 이벤트를 계속 캡쳐하며 두 작업 모두 다른 작업을 차단하지 않음.
* <u>증분 스냅샷의 진행이 중단된 경우 데이터 손실이 없이 재개할 수 있음.</u> 프로세스가 재개된 후에 처음부터 테이블을 다시 캡쳐하는 것이 아니라 중지된 지점에서 스냅샷이 시작됨.
* 언제든지 필요에 따라 증분 스냅샷을 실행하고 데이터베이스 업데이트에 적응하기 위해 필요에 따라 프로세스를 반복할 수 있음. 예를 들어 커넥터 구성을 수정하여 해당 `table.include.list` 속성에 테이블을 추가한 후 스냅샷을 다시 실행할 수 있음.

<u>증분 스냅샷 프로세스(작업 순서 요약)</u>

증분 스냅샷을 실행하면 기본 키를 기준으로 각 테이블을 정렬한 다음 [구성된 청크 크기](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-property-incremental-snapshot-chunk-size)에 따라 테이블을 청크로 분할함. 청크 단위로 작업한 다음 청크의 각 테이블 행을 캡쳐함. 스냅샷은 캡쳐하는 각 행에 대해 `READ`(스냅샷) 이벤트를 생성. 해당 이벤트는 청크의 스냅샷이 시작될 당시의 행의 값을 나타냄.

스냅샷의 진행중에도 다른 프로세스가 계속해서 데이터베이스에 액세스하여 잠재적으로 테이블 레코드를 수정할 가능성이 있음. 이러한 변경 사항을 반영하기 위해 `INSERT`, `UPDATE`, `DELETE` 작업이 평소와 같이 트랜잭션 로그에 커밋됨. 마찬가지로 진행 중인 Debezium 스트리밍 프로세스도 이러한 변경 이벤트를 계속 감지해서 해당 변경 이벤트 레코드를 Kafka에 내보냄.

<u>Debezium이 동일한 기본 키를 가진 레코드 간의 충돌을 해결하는 방법</u>

순서 없이 도착하는 증분 스냅샷 이벤트가 올바른 논리적 순서로 처리되도록 하기 위해 충돌 해결을 위한 버퍼링 체계를 사용. 스냅샷 이벤트와 스트리밍 이벤트 간의 충돌이 해결된 후에만 Kafka에 이벤트 레코드를 전송.

<u>스냅샷 윈도우(Snapshot Window)</u>

스냅샷 윈도우는 늦게 도착하는 `READ`이벤트와 동일한 테이블 행을 수정하는 스트리밍 이벤트 간의 충돌 해결에 도움이 됨. 증분 스냅샷이 지정된 테이블 청크에 대한 데이터를 캡쳐하는 간격을 구분함. 스냅샷 청크의 창이 열리기 전에 Debezium은 기본적인 동작을 수행하고 트랜잭션 로그의 이벤트를 대상 Kafka Topic으로 직접 다운스트림으로 내보냄. 이 가운데 특정 청크에 대한 스냅샷이 열리는 순간부터 닫힐 때까지 Debezium은 동일한 기본 키를 가진 이벤트 간의 충돌을 해결하기 위해 중복 제거 단계를 수행.

각 데이터 수집에서 Debezium은 두 가지 유형의 이벤트를 내보내고 두 이벤트에 대한 레코드를 단일 대상 Kafka Topic에 저장. 테이블을 직접 캡쳐하는 스냅샷 레코드는 READ 작업으로. 사용자가 데이터 컬렉션을 기록하는 업데이트는 트랜잭션 로그에 각 커밋을 반영하고 변경함. `UPDATE`, `DELETE` 작업을 내보냄.

스냅샷 윈도우가 오픈되고 Debezium은 스냅샷 청크 처리를 시작하면서 <u>스냅샷 레코드를 메모리 버퍼에 전달</u>. 스냅샷 처리동안 `READ` 메모리 버퍼에 있는 이벤트의 기본 키는 수신되는 스트리밍 이벤트의 기본 키와 비교됨. 서로 일치하는 항목이 없으면 스트리밍된 이벤트 레코드가 Kafka로 직접 전송됨. 일치하는 항목이 감지되면 버퍼링된 `READ` 이벤트를 삭제하고 스트리밍된 레코드를 대상 Topic에 기록. 청크에 대한 스냅샷 윈도우가 닫힌 후에 버퍼에 남아있는 `READ` 이벤트를 Kafka 토픽으로 전송.

커넥터는 각 스냅샷 청크에 대해 스냅샷 윈도우를 생성하는 위의 프로세스를 반복.

<u>증분 스냅샷 트리거</u>

현재 증분 스냅샷을 시작하는 유일한 방법은 원본 데이터베이스의 Signaling Table(신호 테이블)에 [임시 스냅샷 신호](https://debezium.io/documentation/reference/stable/configuration/signalling.html#sending-signals-to-a-debezium-connector)를 저장하는 것.

SQL `INSERT`쿼리로 Signaling Table에 전달.

Debezium은 Signaling Table의 변경 사항을 감지한 후 Signal을 읽고 요청된 스냅샷 작업을 실행.

Signaling Table에 전달하는 쿼리는 스냅샷에 포함할 테이블과 선택적으로 스냅샷 작업 종류(incremental, blocking)를 지정. 현재 스냅샷 작업에 유효한 옵션은 기본값인 incremental 임.

증분 스냅샷에 포함할 테이블을 지정하려면 `data-collections`에 테이블을 나열한 배열이나 정규 표현식을 이용한 문자열 배열을 사용.  
예를 들면  

```json
{ "data-collections" : ["public.MyFirstTable", "public.MySecondTable"] }
```

증분 스냅샷 신호를 위한 `data-collections` 배열에는 기본값이 없음. 따라서 비워두면 필요한 작업이 없음을 감지하고 스냅샷을 수행하지 않음.

> [!NOTE]  
> 만약 테이블명에 데이터 베이스, 스키마와 같이 사용될 경우 (`.`)가 들어가게 되는데 이 경우는 쌍따옴표를 이용해서 묶어 줘야 한다.  
> 예) `public` 스키마에 `My.Table`라는 테이블 이라면 다음과 같이 작성을 해야 한다. `"public"."My.Table"`

전제 조건

* [Signaling이 활성화](https://debezium.io/documentation/reference/stable/configuration/signalling.html#debezium-signaling-enabling-source-signaling-channel) 되어 있음.
  * Signaling 데이터 컬렉션이 원본 데이터베이스에 존재
  * Signaling 데이터 컬렉션은 `signal.data.collection` 속성에 지정됨.

원본 Signaling 채널을 사용한 증분 스냅샷 트리거

1. Signaling 테이블에 임시 증분 스냅샷 요청을 추가하려면 Insert SQL 쿼리를 실행

```sql
INSERT INTO <signalTable> (id, type, data) 
VALUES ('<id>', '<snapshotType>', 
'{"data-collections" : ["<tableName>","<tableName>"],
   "type" : "<snapshotType>",
   "additional-conditions" : [{"data-collection": "<tableName>", "filter": "<additional-condition>"}]
 }');
```

예를 들어

```sql
INSERT INTO myschema.debezium_signal (id, type, data)    ①
values ('ad-hoc-1',                                      ②
    'execute-snapshot',                                  ③
    '{"data-collections": ["schema1.table1", "schema2.table2"],   ④
    "type":"incremental"},                                        ⑤
    "additional-conditions":[{"data-collection": "schema1.table1" ,"filter":"color='blue'"}]}');   ⑥
```

id, type, data 파라미터 각각의 의미는 [signaling 테이블 필드 설명 참조](https://debezium.io/documentation/reference/stable/configuration/signalling.html#debezium-signaling-description-of-required-structure-of-a-signaling-data-collection).

아래의 테이블에 예제의 각 파라미터를 설명하고 있음.

|항목|값|설명|
|-|-|-|
|1|myschema.debezium_signal|원본 데이터베이스에 있는 Signaling Table을 지정|
|2|ad-hoc-1|id 매개변수는 신호 요청에 대한 ID 식별자로 할당되는 임의의 문자열을 지정. 이 값을 사용하여 신호 테이블의 항목에 대한 로깅 메시지를 식별. Debezium은 이 문자열을 사용하지 않으며 오히려 스냅샷 중에 Debezium은 워터마킹(해쉬값) 신호로 자체 ID 문자열을 생성|
|3|execute-snapshot|Signal이 트리거 하려는 작업을 지정|
|4|data-collections|스냅샷에 포함할 테이블 이름과 일치하는 테이블 이름 또는 정규 표현식의 배열을 지정하는 신호 데이터 필드의 필수 구성요소. 배열에는 `signal.data.collection` 구성 속성에서 커넥터의 신호 테이블 이름을 지정하는 데 사용하는 것과 동일한 형식을 사용하여 정규화된 이름으로 테이블과 일치하는 정규식을 나열.|
|5|incremental|실행할 스냅샷 작업 종류를 지정하는 신호 데이터 필드의 선택적 유형 구성 요소. 현재 유일하게 유효한 옵션은 기본값인 `incremental`. 값을 지정하지 않으면 커넥터가 증분 스냅샷을 실행.|
|6|additional-conditions|스냅샷에 포함할 레코드의 하위 집합을 결정하기 위해 조건(where절) 집합을 지정하는 선택적 배열. 각 추가 조건은 데이터 수집 및 필터 속성이 포함된 개체. 각 데이터 컬렉션에 대해 서로 다른 필터를 지정 가능. <br> * data-collection 속성은 필터가 적용될 데이터 컬렉션의 정규화된 이름. 추가 조건 매개변수에 대한 자세한 내용은 [additional-conditions로 임시 증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots-additional-conditions)을 참조하세요.|

<u>additional-conditions로 임시 증분 스냅샷</u>

스냅샷에 테이블 콘텐츠의 하위 집합만 포함하려면 스냅샷 신호에 `additional-conditions` 매개변수를 추가하여 신호 요청을 수정

전형적인 스냅샷을 위한 쿼리는 아래의 형식을 따름.

```sql
SELECT * FROM <tableName> ....
```

다음은 `additional-conditions` 파라미터를 추가함으로써 SQL 쿼리에 `WHERE`절을 추가할 수 있음.

```sql
SELECT * FROM <data-collection> WHERE <filter> ....
```

다음 예에서는 신호 테이블에 추가 조건이 포함된 임시 증분 스냅샷 요청을 보내는 SQL 쿼리.

```sql
INSERT INTO <signalTable> (id, type, data) 
VALUES ('<id>', 
'<snapshotType>', 
'{"data-collections" : ["<tableName>","<tableName>"],
  "type" : "<snapshotType>",
  "additional-conditions" : [{"data-collection": "<tableName>", "filter": "<additional-condition>"}]
 }');
```

예를 들어 다음 열이 포함된 `products` 테이블이 있다고 가정.

* id (기본키)
* color
* quantity

`products` 테이블의 증분 스냅샷에 `color=blue` 데이터 항목만 포함하려면 다음 SQL문을 사용하여 스냅샷 트리거로 설정

```sql
INSERT INTO myschema.debezium_signal (id, type, data) 
VALUES('ad-hoc-1', 
'execute-snapshot', 
'{"data-collections" : ["schema1.products"],
  "type" : "incremental", 
  "additional-conditions" : [{"data-collection": "schema1.products", "filter": "color=blue"}]
}');
```

`additional-conditions` 매개변수를 사용하면 둘 이상의 열을 기반으로 하는 조건을 전달 가능. 예를 들어, 이전 예의 `products` 테이블을 사용하여 `color=blue and quantity>10`인 항목의 데이터만 포함하는 증분 스냅샷을 트리거하는 쿼리 등록 가능.

```sql
INSERT INTO myschema.debezium_signal (id, type, data)
VALUES('ad-hoc-1', 
'execute-snapshot', 
'{"data-collections" : ["schema1.products"],
  "type":"incremental", 
  "additional-conditions" : [{"data-collection": "schema1.products", "filter": "color=blue AND quantity>10"}]
}');
```

다음 예는 커넥터가 캡쳐한 증분 스냅샷 이벤트에 대한 JSON을 보여줌. (예: 증분 스냅샷 이벤트 메세지)

```json
{
    "before":null,
    "after": {
        "pk":"1",
        "value":"New data"
    },
    "source": {
        ...
        "snapshot":"incremental"    ① 
    },
    "op":"r",                       ②
    "ts_ms":"1620393591654",
    "transaction":null
}
```

|항목|필드명|설명|
|-|-|-|
|1|snapshot|실행할 스냅샷 작업 유형을 지정. 현재 유일하게 유효한 옵션은 기본값인 `incremental`.<br>유형 값을 지정하는 것은 선택 사항.<br>값을 지정하지 않으면 커넥터가 증분 스냅샷을 실행.|
|2|op|이벤트 유형을 지정. 스냅샷 이벤트 값은 `r` (`READ` 작업)|

<u>Kafka 신호 채널을 사용하여 증분 스냅샷 트리거</u>

[카프카 topic](https://debezium.io/documentation/reference/stable/configuration/signalling.html#debezium-signaling-enabling-kafka-signaling-channel)에 메시지를 보내 커넥터가 임시 증분 스냅샷을 실행하도록 요청 가능.

Kafka 메시지의 키는 커넥터 구성 옵션의 `topic.prefix` 값과 반드시 일치.

메시지의 값은 `type`및 `data`필드가 있는 JSON 개체

신호 유형은 `execute-snapshot`이며 `data` 필드에는 다음 필드 항목이 들어가야 함.

|필드|기본값|설명|
|-|-|-|
|type|incremental|실행할 스냅샷의 유형. 현재 Debezium은 `incremental` 유형만 지원.|
|data-collections|N/A|스냅샷에 포함할 테이블의 정규화된 이름과 일치하는 정규 표현식의 배열.<br>`signal.data.collection` 구성 옵션 에 필요한 것과 동일한 형식을 사용하여 이름을 지정.|
|additional-<br>conditions|N/A|스냅샷에 포함할 레코드(행)의 하위 집합을 결정하기 위해 커넥터가 평가하는 추가 조건 집합을 지정하는 선택적 배열<br>각 추가 조건은 임시 스냅샷이 캡쳐하는 데이터를 필터링하기 위한 기준을 지정하는 개체<br>각 추가 조건에 대해 다음 매개변수를 설정<br><br>*data-collection*<br>필터가 적용된 정규화된 이름. 각 테이블에 서로 다른 필터를 적용할 수 있음.<br><br>*filter*<br>스냅샷에 포함하기 위해 데이터베이스 레코드에 있어야 할 열 값을 지정<br>예) "color='blue'"<br><br>필터 매개변수에 할당하는 값은 blocking 스냅샷에 대한 `snapshot.select.statement.overrides` 속성을 설정할 때와 같이 SELECT 문의 WHERE 절에 지정할 수 있는 값과 동일한 유형|

`execute-snapshot` Kafka message의 예

```json
Key = `test_connector`
Value = `{"type":"execute-snapshot","data": {"data-collections": ["schema1.table1", "schema1.table2"], "type": "INCREMENTAL"}}`
```

<u>추가 조건을 사용한 임시 증분 스냅샷</u>

Debezium은 `additional-conditions` 필드를 사용하여 테이블 내용의 하위 집합을 선택

일반적으로 Debezium은 스냅샷을 실행할 때 다음과 같은 SQL 쿼리를 실행

```sql
SELECT * FROM <tableName> …​.
```

스냅샷 요청에 `additional-conditions` 속성이 포함된 경우 해당 속성의 `data-collection` 및 `filter` 매개변수가 SQL 쿼리에 추가됨. 예를 들면,

```sql
SELECT * FROM <data-collection> WHERE <filter> …​.
```

예를 들어, 열 `id(기본 키)`, `color`및 `brand` 필드를 가진 `products`테이블에서 `color='blue'` 스냅샷에 콘텐츠만 포함하도록 하려면  
스냅샷을 요청할 때 `additional-conditions` 속성을 추가하여 콘텐츠를 필터링 가능.

```json
Key = `test_connector`

Value = `{"type":"execute-snapshot","data": {"data-collections": ["schema1.products"], "type": "INCREMENTAL", "additional-conditions": [{"data-collection": "schema1.products" ,"filter":"color='blue'"}]}}`
```

이 `additional-conditions` 속성을 사용하여 여러 열에 조건을 전달할 수 있음. 예를 들어, 이전 예와 동일한 테이블을 사용하여, 
`products` 테이블의 `productscolor='blue'`및 `brand='MyBrand'`에 대한 콘텐츠만 스냅샷에 포함시키려면 다음 요청을 전송.

```json
Key = `test_connector`

Value = `{"type":"execute-snapshot","data": {"data-collections": ["schema1.products"], "type": "INCREMENTAL", "additional-conditions": [{"data-collection": "schema1.products" ,"filter":"color='blue' AND brand='MyBrand'"}]}}`
```

<u>증분 스냅샷 중지</u>

원본 데이터베이스의 테이블에 신호를 보내 증분 스냅샷 중지 가능. 스냅샷 중지 INSERT SQL 쿼리를 신호 테이블에 제출.

Debezium은 신호 테이블의 변경 사항을 감지하고 신호를 읽고 진행 중인 증분 스냅샷 작업을 중지.

제출한 쿼리는 `incremental`의 스냅샷 작업을 지정하고 선택적으로 현재 실행 중인 스냅샷의 테이블을 제거하도록 지정.

전제조건

* 신호가 활성화 되어 있어야 함.
  * 신호 데이터 컬렉션이 원본 데이터베이스에 존재
  * `signal.data.collection` 속성에 신호 데이터 컬렉션 지정

소스 신호 채널을 사용하여 증분 스냅샷 중지

1. 신호 테이블에 다음의 SQL 쿼리를 보내서 임시 증분 스냅샷을 중지  

```sql
INSERT INTO <signalTable> (id, type, data) values ('<id>', 'stop-snapshot', '{"data-collections": ["<tableName>","<tableName>"], "type":"incremental"}');
```

예를 들면,

```sql
NSERT INTO myschema.debezium_signal (id, type, data)              ①
values ('ad-hoc-1',                                               ②
    'stop-snapshot',                                              ③
    '{"data-collections": ["schema1.table1", "schema2.table2"],   ④
    "type":"incremental"}');                                      ⑤
```

id, type, data 파라미터 각각의 의미는 [signaling 테이블 필드 설명 참조](https://debezium.io/documentation/reference/stable/configuration/signalling.html#debezium-signaling-description-of-required-structure-of-a-signaling-data-collection).

아래의 테이블에 예제의 각 파라미터를 설명하고 있음.

|항목|값|설명|
|-|-|-|
|1|myschema.debezium_signal|원본 데이터베이스에 있는 Signaling Table을 지정|
|2|ad-hoc-1|id 매개변수는 신호 요청에 대한 ID 식별자로 할당되는 임의의 문자열을 지정. 이 값을 사용하여 신호 테이블의 항목에 대한 로깅 메시지를 식별. Debezium은 이 문자열을 사용하지 않으며 오히려 스냅샷 중에 Debezium은 워터마킹(해쉬값) 신호로 자체 ID 문자열을 생성|
|3|stop-snapshot|Signal이 트리거 하려는 작업을 지정|
|4|data-collections|스냅샷에 포함할 테이블 이름과 일치하는 테이블 이름 또는 정규 표현식의 배열을 지정하는 신호 데이터 필드의 필수 구성요소. 배열에는 `signal.data.collection` 구성 속성에서 커넥터의 신호 테이블 이름을 지정하는 데 사용하는 것과 동일한 형식을 사용하여 정규화된 이름으로 테이블과 일치하는 정규식을 나열. <u>`data` 필드가 생략되면 진행 중인 전에 증분 스냅샷을 중지.</u>|
|5|incremental|중지할 스냅샷 작업 종류를 지정하는 신호 데이터 필드의 선택적 유형 구성 요소. 현재 유일하게 유효한 옵션은 기본값인 `incremental`. <u>값을 지정하지 않으면 커넥터가 증분 스냅샷을 중지하지 못함.</u>|

2. Kafka 신호 채널을 사용하여 증분 스냅삿 중지

[카프카 topic](https://debezium.io/documentation/reference/stable/configuration/signalling.html#debezium-signaling-enabling-kafka-signaling-channel)에 메시지를 보내 커넥터가 임시 증분 스냅샷을 중지하도록 요청 가능.

Kafka 메시지의 키는 커넥터 구성 옵션의 `topic.prefix` 값과 반드시 일치.

메시지의 값은 `type`및 `data`필드가 있는 JSON 개체

신호 유형은 `stop-snapshot`이며 `data` 필드에는 다음 필드 항목이 들어가야 함.

|필드|기본값|설명|
|-|-|-|
|type|incremental|실행할 스냅샷의 유형. 현재 Debezium은 `incremental` 유형만 지원.|
|data-collections|N/A|스냅샷에 포함할 테이블의 정규화된 이름과 일치하는 정규 표현식의 배열.<br>`signal.data.collection` 구성 옵션 에 필요한 것과 동일한 형식을 사용하여 이름을 지정.|

다음 예는 일반적인 stop-snapshotKafka 메시지

```json
Key = `test_connector`
Value = `{"type":"stop-snapshot","data": {"data-collections": ["schema1.table1", "schema1.table2"], "type": "INCREMENTAL"}}`
```

<u>Blocking 스냅샷</u>

스냅샷 관리에 더 많은 유연성을 제공하기 위해 Debezium에는 `Blocking 스냅샷` 이라고 알려진 임시 스냅샷 메커니즘을 포함. 차단 스냅샷은 [Debezium 커넥터에 신호를 보내는데 Debezium 메커니즘을 사용](https://debezium.io/documentation/reference/stable/configuration/signalling.html).

Blocking 스냅샷은 런타임에 트리거할 수 있다는 점을 제외하면 initial 스냅샷과 동일하게 동작.

다음 상황에서는 initial 스냅샷 프로세스를 사용하는 대신 Blocking 스냅샷을 실행할 수 있습니다.

* 새 테이블을 추가하고 커넥터가 실행되는 동안 스냅샷을 완료하려 함.
* 큰 테이블을 추가하고 증분 스냅샷보다 더 짧은 시간에 스냅샷을 완료하려 함.

<u>Blocking 스냅샷 프로세스</u>

Blocking 스냅샷을 실행하면 Debezium은 스트리밍을 중지한 다음 초기 스냅샷 중에 사용하는 것과 동일한 프로세스에 따라 지정된 테이블의 스냅샷을 시작.  
스냅샷이 완료되면 스트리밍을 재개

<u>스냅샷 구성</u>

Signaling의 `data` 구성 요소에는 다음 속성 설정.

* data-collections: 스냅샷이 되어야 하는 테이블을 지정합니다.
* 추가 조건: 테이블마다 다른 필터를 지정할 수 있습니다.
  * 속성 data-collection은 필터가 적용될 테이블의 정규화된 이름
  * 해당 `filter` 속성 은 `snapshot.select.statement.overrides`과 같은 값을 가짐

```json
{"type": "blocking", 
 "data-collections": ["schema1.table1", "schema1.table2"], 
 "additional-conditions": [
   {"data-collection": "schema1.table1", "filter": "SELECT * FROM [schema1].[table1] WHERE column1 = 0 ORDER BY column2 DESC"}, 
   {"data-collection": "schema1.table2", "filter": "SELECT * FROM [schema1].[table2] WHERE column2 > 0"}
 ]
}
```

<u>중복 가능성</u>

스냅샷을 트리거하기 위해 신호를 보내는 시간과 스트리밍이 중지되고 스냅샷이 시작되는 시간 사이에 지연 발생 가능.  
이러한 지연으로 인해 스냅샷이 완료된 후 커넥터는 스냅샷에서 캡처한 레코드를 복제하는 일부 이벤트 레코드를 생성할 수 있음.

#### 변경 데이터 테이블 읽기

커넥터가 처음 시작되면 캡처된 테이블 구조의 구조적 스냅샷을 만들고 이 정보를 내부 데이터베이스 Schema history topic에 유지(`schema.history.internal.kafka.topic`).  
그런 다음 커넥터는 각 원본 테이블에 대한 변경 테이블을 식별하고 다음의 4단계를 시작.

1. 각 변경 테이블에 대해 커넥터는 마지막으로 저장된 최대 LSN과 현재 최대 LSN 사이에 생성된 모든 변경 내용 읽기.
2. 커넥터는 커밋 LSN(`payload.source.commit_lsn`) 및 변경 LSN(`payload.source.chnage_lsn`) 값을 기준으로 읽는 변경 사항을 오름차순 정렬. 이 정렬 순서는 변경 사항이 데이터베이스에서 발생한 것과 동일한 순서로 Debezium에서 재생되도록 보장.
3. 커넥터는 커밋 및 변경 LSN을 Kafka Connect에 오프셋(`connect-offsets`)으로 전달.
4. 커넥터는 최대 LSN을 저장하고 1단계부터 프로세스를 다시 시작.

```json
# show_topic_messages json mydb.MyDB.dbo.Customer
"payload": {
    "before": null,
    "after": {
      "CustomerName": "홍길동",
      "Age": 21,
      "CustomerAddress": "해운대구",
      "Salary": "HoSA",
      "IDX": 2
    },
    "source": {
      "version": "2.4.0.Final",
      "connector": "sqlserver",
      "name": "mydb",
      "ts_ms": 1698114165367,
      "snapshot": "false",
      "db": "MyDB",
      "sequence": null,
      "schema": "dbo",
      "table": "Customer",
      "change_lsn": "00000039:00000b80:0003",   # 이전 lsn, 최초값 : null
      "commit_lsn": "00000039:00000b80:0004",   # 커밋 후 lsn
      "event_serial_no": 1
    },
    "op": "c",
    "ts_ms": 1698114168434,
    "transaction": null
  }

  # show_topic_messages json schema-changes.mssql.mydb
  {
  "source": {
    "server": "mydb",
    "database": "MyDB"
  },
  "position": {
    "commit_lsn": "00000039:000008f0:001c",
    "snapshot": true,
    "snapshot_completed": false
  },
  "ts_ms": 1698114047182,
  "databaseName": "MyDB",
  "schemaName": "dbo",
  "tableChanges": [
    {
      "type": "CREATE",
      "id": "\"MyDB\".\"dbo\".\"Customer\"",
      "table": {
        "defaultCharsetName": null,
        "primaryKeyColumnNames": [
          "IDX"
        ],
        "columns": [
          {
            "name": "CustomerName",
            "jdbcType": 12,
            "typeName": "varchar",
            "typeExpression": "varchar",
            "charsetName": null,
            "length": 10,
            "position": 1,
            "optional": false,
            "autoIncremented": false,
            "generated": false,
            "comment": null,
            "hasDefaultValue": false,
            "enumValues": []
          },
          {
            "name": "Age",
            "jdbcType": 4,
            "typeName": "int",
            "typeExpression": "int",
            "charsetName": null,
            "length": 10,
            "scale": 0,
            "position": 2,
            "optional": false,
            "autoIncremented": false,
            "generated": false,
            "comment": null,
            "hasDefaultValue": false,
            "enumValues": []
          },
          {
            "name": "CustomerAddress",
            "jdbcType": 12,
            "typeName": "varchar",
            "typeExpression": "varchar",
            "charsetName": null,
            "length": 200,
            "position": 3,
            "optional": false,
            "autoIncremented": false,
            "generated": false,
            "comment": null,
            "hasDefaultValue": false,
            "enumValues": []
          },
          {
            "name": "Salary",
            "jdbcType": 3,
            "typeName": "decimal",
            "typeExpression": "decimal",
            "charsetName": null,
            "length": 10,
            "scale": 2,
            "position": 4,
            "optional": false,
            "autoIncremented": false,
            "generated": false,
            "comment": null,
            "hasDefaultValue": false,
            "enumValues": []
          },
          {
            "name": "IDX",
            "jdbcType": -5,
            "typeName": "bigint identity",
            "typeExpression": "bigint identity",
            "charsetName": null,
            "length": 19,
            "scale": 0,
            "position": 5,
            "optional": false,
            "autoIncremented": true,
            "generated": false,
            "comment": null,
            "hasDefaultValue": false,
            "enumValues": []
          }
        ],
        "attributes": []
      },
      "comment": null
    }
  ]
}
```

다시 시작한 후 커넥터는 읽은 마지막 오프셋(LSN 커밋 및 변경)부터 처리를 재개.

```json
# show_topic_messages json connect-offsets
[
  "mydb_connector",
  {
    "server": "mydb",
    "database": "MyDB"
  }
]
{
  "commit_lsn": "00000039:000008f0:001c",
  "snapshot": true,
  "snapshot_completed": true
}
[
  "mydb_connector",
  {
    "server": "mydb",
    "database": "MyDB"
  }
]
{
  "transaction_id": null,
  "event_serial_no": 1,
  "commit_lsn": "00000039:00000b80:0004",
  "change_lsn": "00000039:00000b80:0003"
}
[
  "mydb_connector",
  {
    "server": "mydb",
    "database": "MyDB"
  }
]
{
  "transaction_id": null,
  "event_serial_no": 1,
  "commit_lsn": "00000039:00002a80:0003",
  "change_lsn": "00000039:00002a80:0002"
}
[
  "mydb_connector",
  {
    "server": "mydb",
    "database": "MyDB"
  }
]
{
  "transaction_id": null,
  "event_serial_no": 1,
  "commit_lsn": "00000042:000020b0:0004",
  "change_lsn": "00000042:000020b0:0003"
}
```

커넥터는 원본 테이블에 대해 CDC가 활성화 또는 비활성화되었는지 여부를 감지하고 읽기 동작 조정 가능.

#### 데이터베이스에 기록된 최대 LSN이 없습니다.

다음과 같은 이유로 데이터베이스에 최대 LSN이 기록되지 않는 상황이 있을 수 있음.

1. SQL Server 에이전트가 실행되고 있지 않음.
2. 아직 변경 테이블에 변경 사항이 기록되지 않음.
3. 데이터베이스 활동이 낮고 cdc 정리 작업이 주기적으로 cdc 테이블에서 항목을 삭제.

이러한 가능성 중에서 SQL Server 에이전트를 실행하는 것이 전제 조건이므로 1번이 실제 문제임.(2번과 3번은 정상).

이 문제를 완화하고 1번과 다른 것을 구별하기 위해 다음 쿼리를 통해 SQL Server 에이전트의 상태를 확인합니다 

```sql
SELECT CASE WHEN dss.[status]=4 THEN 1 ELSE 0 END AS isRunning FROM [#db].sys.dm_server_services dss WHERE dss.[servicename] LIKE N’SQL Server Agent (%';
```

SQL Server 에이전트가 실행되고 있지 않으면 로그에 `No maximum LSN records in the Database; SQL Server Agent is running`이라는 오류가 기록됨.

#### 제한사항

SQL Server에서 변경 캡처 인스턴스를 생성하려는 기본 개체는 반드시 테이블. 결과적으로 인덱싱된 뷰(구체화된 뷰라고도 함)에서 변경 내용을 캡처하는 것은 SQL Server 및 Debezium SQL Server 커넥터에서 지원하지 않음.

#### Topic Names (토픽 이름)

기본적으로 SQL Server 커넥터는 테이블에서 발생하는 모든 `INSERT`, `UPDATE` 및 `DELETE` 작업에 대한 이벤트를 해당 테이블과 관련된 단일 Apache Kafka Topic에 기록.  
커넥터는 다음 규칙을 사용하여 변경 이벤트 Topic의 이름을 지정.

`<topicPrefix>.<schemaName>.<tableName>`

기본 이름의 구성 요소에 대한 정의

*주제 접두어*  
topic.prefix 구성 속성에 명시된 서버의 논리적 이름.

*스키마 이름*  
변경 이벤트가 발생한 데이터베이스 스키마의 이름.

*테이블 이름*  
변경 이벤트가 발생한 데이터베이스 테이블의 이름.

예를 들어 `fulfillment`가 논리 서버 이름이고 `dbo`가 스키마 이름이고, 데이터베이스에 이름이 `products`, `products_on_hand`, `customers`및 `orders`인 테이블이 포함된 경우 커넥터는 변경 이벤트 레코드를 다음 Kafka Topic으로 스트리밍합니다.

* `fulfillment.testDB.dbo.products`
* `fulfillment.testDB.dbo.products_on_hand`
* `fulfillment.testDB.dbo.customers`
* `fulfillment.testDB.dbo.orders`

커넥터는 유사한 명명 규칙을 적용하여 Internal database schema history topic, [schema change topic](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#about-the-debezium-sqlserver-connector-schema-change-topic) 및 [transaction metadata topic](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-transaction-metadata)에 레이블을 지정.

기본 topic 이름이 요구 사항을 충족하지 않는 경우 사용자 지정 topic 이름으로 구성 가능.  
사용자 정의 topic 이름을 구성하려면 논리적 topic routing SMT에 정규식을 지정.  
논리적 topic routing SMT를 사용하여 topic 이름 지정을 사용자 정의하는 방법에 대한 자세한 내용은 [topic routing](https://debezium.io/documentation/reference/stable/transformations/topic-routing.html#topic-routing)을 참조하세요.

#### Schema history topic

데이터베이스 클라이언트가 데이터베이스를 쿼리할 때 클라이언트는 데이터베이스의 현재 스키마를 사용.  
그러나 데이터베이스 스키마는 언제든지 변경 가능. 즉, 커넥터는 각 `INSERT`, `UPDATE` 또는 `DELETE` 작업이 기록된 당시의 스키마를 식별할 수 있어야 함.  
또한 커넥터는 반드시 현재 스키마를 모든 이벤트에 적용할 수는 없음. 이벤트가 비교적 오래된 경우 현재 스키마가 적용되기 전에 기록되었을 가능성이 있음.

Debezium SQL Server 커넥터는 스키마 변경 후 발생하는 변경 이벤트의 올바른 처리를 보장하기 위해  
관련 데이터 테이블의 구조를 미러링하는 SQL Server 변경 테이블의 구조를 기반으로 새 스키마의 스냅샷을 저장.  
커넥터는 데이터베이스 Schema history Kafka topic(`schema.history.internal.kafka.topic`)에 스키마 변경으로 인한 작업의 LSN과 함께  
테이블 스키마 정보를 저장.  
커넥터는 저장된 스키마 정보를 사용하여 각 `INSERT`, `UPDATE` 또는 `DELETE`작업 시 테이블 구조를 올바르게 미러링하는 변경 이벤트를 생성.

충돌 또는 정상적인 중지 후에 커넥터가 다시 시작되면 마지막으로 읽은 위치부터 SQL Server CDC 테이블의 항목 읽기를 다시 시작.  
커넥터는 데이터베이스 Schema history topic에서 읽는 스키마 정보를 기반으로 커넥터가 다시 시작되는 위치에 존재했던 테이블 구조를 적용.

캡처 모드에 있는 Db2 테이블의 스키마를 업데이트하는 경우 해당 변경 테이블의 스키마도 업데이트하는 것이 중요.  
데이터베이스 스키마를 업데이트하려면 높은 권한을 가진 SQL Server 데이터베이스 관리자가 필요.  
Debezium 환경에서 SQL Server 데이터베이스 스키마를 업데이트하는 방법에 대한 자세한 내용은 [데이터베이스 스키마 진화](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-schema-evolution)를 참조하세요 .

데이터베이스 Schema history topic은 내부 커넥터 전용.  
선택적으로 커넥터는 [Consumer 애플리케이션을 위한 다른 topic으로 스키마 변경 이벤트](#Schema-change-topic)를 내보낼 수도 있음.

추가 리소스
* [Debezium 이벤트 레코드를 수신하는 주제의 기본 이름](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-topic-names).

#### Schema change topic

CDC가 활성화된 각 테이블에 대해 Debezium SQL Server 커넥터는 데이터베이스의 테이블에 적용되는 스키마 변경 이벤트 기록을 저장.  
커넥터는 이름이 *`<topicPrefix>`* Kafka topic인 schema change event를 기록. `topicPrefix`는 `topic.prefix` 구성 속성에 명시된 논리 서버.

커넥터가 schema change topic으로 보내는 메시지에는 payload가 포함되어 있으며, 선택적으로 변경 이벤트 메시지의 스키마도 포함됨.

스키마 변경 이벤트 요소

**name**  
　　스키마 변경 이벤트 메시지 이름

**type**  
　　변경 이벤트 메시지 유형

**version**  
　　스키마 버전. 스키마가 변경될 때마다 증가되는 정수

**fields**  
　　변경 이벤트 메시지에 포함된 필드

<u>예: SQL Server 커넥터 스키마 변경 항목의 스키마</u>

JSON 형식의 일반적인 스키마 구조

```json
{
  "schema": {
    "type": "struct",
    "fields": [
      {
        "type": "string",
        "optional": false,
        "field": "databaseName"
      }
    ],
    "optional": false,
    "name": "io.debezium.connector.sqlserver.SchemaChangeKey",
    "version": 1
  },
  "payload": {
    "databaseName": "MyDB"  ①
  }
}
```

1. 스키마 변경 이벤트 메시지의 페이로드에 포함되는 요소

**dabaseName**  
　　명령문이 적용되는 데이터베이스 이름. `databaseName` 값은 메세지의 키 값.

**tableChanges**  
　　스키마 변경 후 전체 테이블 스키마의 구조화된 표현.  
　　필드 `tableChanges`에는 테이블의 각 열에 대한 상세 내역을 가진 배열을 포함.  
　　구조화된 표현은 데이터를 JSON 또는 Avro 형식으로 표시하므로 소비자는 먼저 DDL 파서를 통해 메시지를 처리하지 않고도 메시지를 쉽게 읽을 수 있음.

```json
"tableChanges": [
      {
        "type": "CREATE",
        "id": "\"MyDB\".\"dbo\".\"Customer\"",
        "table": {
          "defaultCharsetName": null,
          "primaryKeyColumnNames": [
            "IDX"
          ],
          "columns": [
            {
              "name": "CustomerName",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 10,
              "scale": null,
              "position": 1,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "Age",
              "jdbcType": 4,
              "nativeType": null,
              "typeName": "int",
              "typeExpression": "int",
              "charsetName": null,
              "length": 10,
              "scale": 0,
              "position": 2,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "CustomerAddress",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 200,
              "scale": null,
              "position": 3,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "Salary",
              "jdbcType": 3,
              "nativeType": null,
              "typeName": "decimal",
              "typeExpression": "decimal",
              "charsetName": null,
              "length": 10,
              "scale": 2,
              "position": 4,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "IDX",
              "jdbcType": -5,
              "nativeType": null,
              "typeName": "bigint identity",
              "typeExpression": "bigint identity",
              "charsetName": null,
              "length": 19,
              "scale": 0,
              "position": 5,
              "optional": false,
              "autoIncremented": true,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            }
          ],
          "comment": null
        }
      }
    ]
```

> [!NOTE]
> 커넥터가 테이블을 캡처하도록 구성되면 스키마 변경 항목뿐만 아니라 내부 데이터베이스 스키마 기록 항목에도 테이블의 스키마 변경 기록을 저장.  
> 내부 데이터베이스 schema history topic 은 커넥터 전용이며 애플리케이션을 사용하여 직접 사용하기 위한 것이 아님.  
> 스키마 변경에 대한 알림이 필요한 애플리케이션은 schema change topic의 해당 정보만 사용하는지 확인 필요.

> [!WARNING]
> 커넥터가 스키마 변경 주제에 내보내는 메시지 형식은 잠복기 상태이며 예고 없이 변경될 수 있음.

Debezium은 다음 이벤트가 발생할 때 schema change topic에 메시지를 보냄.

* 테이블에 대해 CDC를 활성화.
* 테이블에 대해 CDC를 비활성화.
* 스키마 발전 절차 에 따라 CDC가 활성화된 테이블의 구조를 변경

<u>예: SQL Server 커넥터 스키마 변경 항목으로 내보내는 메시지</u>

다음 예는 schema change topic의 메시지이며 테이블 스키마의 논리적 표현이 포함되어 있음.

```json
{
  "schema": {
    ...
  },
  "payload": {
    "source": {
      "version": "2.4.0.Final",
      "connector": "sqlserver",
      "name": "mydb",
      "ts_ms": 1698304664178,
      "snapshot": "true",
      "db": "MyDB",
      "sequence": null,
      "schema": "dbo",
      "table": "Customer",
      "change_lsn": null,
      "commit_lsn": "00000047:00005200:0001",
      "event_serial_no": null
    },
    "ts_ms": 1698304664179, ①
    "databaseName": "MyDB", ②
    "schemaName": "dbo",
    "ddl": null, ③
    "tableChanges": [ ④
      {
        "type": "CREATE", ⑤
        "id": "\"MyDB\".\"dbo\".\"Customer\"", ⑥
        "table": { ⑦
          "defaultCharsetName": null,
          "primaryKeyColumnNames": [ ⑧
            "IDX"
          ],
          "columns": [ ⑨
            {
              "name": "CustomerName",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 10,
              "scale": null,
              "position": 1,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "Age",
              "jdbcType": 4,
              "nativeType": null,
              "typeName": "int",
              "typeExpression": "int",
              "charsetName": null,
              "length": 10,
              "scale": 0,
              "position": 2,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "CustomerAddress",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 200,
              "scale": null,
              "position": 3,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "Salary",
              "jdbcType": 3,
              "nativeType": null,
              "typeName": "decimal",
              "typeExpression": "decimal",
              "charsetName": null,
              "length": 10,
              "scale": 2,
              "position": 4,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "IDX",
              "jdbcType": -5,
              "nativeType": null,
              "typeName": "bigint identity",
              "typeExpression": "bigint identity",
              "charsetName": null,
              "length": 19,
              "scale": 0,
              "position": 5,
              "optional": false,
              "autoIncremented": true,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            }
          ],
          "attributes": [ ⑩
            {
              "customAttribute": "attributeValue"
            }
          ]
        }
      }
    ]
  }
}
```

schema change topic으로 생성된 메시지의 필드에 대한 설명
|번호|필드명|설명|
|-|-|-|
|1|ts_ms|커넥터가 이벤트를 처리한 시간을 표시하는 선택적 필드. 시간은 Kafka Connect 작업을 실행하는 JVM의 시스템 시계가 기반. 원본 객체에서 ts_ms는 데이터베이스가 변경된 시간을 나타냄. `payload.source.ts_ms` 값과 `payload.ts_ms` 값을 비교하면 원본 데이터베이스 업데이트와 Debezium 사이의 지연 확인 가능|
|2|databaseName<br>schemaName|변경 사항이 포함된 데이터베이스와 스키마를 식별|
|3|ddl|SQL Server 커넥터에는 항상 null. 다른 커넥터의 경우 이 필드에는 스키마 변경을 담당하는 DDL을 포함. 이 DDL은 SQL Server 커넥터에 사용 불가.|
|4|tableChanges|DDL 명령으로 생성된 schema 변경을 포함하는 하나 이상의 항목 배열|
|5|type|스키마 변화의 종류. 다음 값 중 하나. <br><br>* CREATE- 테이블 생성. <br>* ALTER- 테이블 수정 <br>* DROP- 테이블 삭제.|
|6|id|생성, 변경 또는 삭제된 테이블의 전체 식별자|
|7|table|변경 사항이 적용된 후의 테이블 메타데이터|
|8|primaryKeyColumnNames|테이블의 기본 키를 구성하는 열 목록|
|9|columns|변경된 테이블의 각 열에 대한 메타데이터|
|10|attributes|각 테이블 변경에 대한 사용자 정의 속성 메타데이터|

커넥터가 schema change topic으로 보내는 메시지에서 Key는 스키마 변경이 포함된 데이터베이스의 이름.
다음 예에서는 payload필드에 키가 포함되어 있음.

```json
{
  "schema": {
    "type": "struct",
    "fields": [
      {
        "type": "string",
        "optional": false,
        "field": "databaseName"
      }
    ],
    "optional": false,
    "name": "io.debezium.connector.sqlserver.SchemaChangeKey",
    "version": 1
  },
  "payload": {
    "databaseName": "MyDB"
  }
}
```

스키마 변경 순서

1. 해당 테이블의 cdc 중지
2. 테이블 필드 속성 변경
3. 해당 테이블 cdc 시작

```json
{
  "schema": {
    ...
  },
  "payload": {
    "source": {
      "version": "2.4.0.Final",
      "connector": "sqlserver",
      "name": "mydb",
      "ts_ms": 1698305296337,
      "snapshot": "false",
      "db": "MyDB",
      "sequence": null,
      "schema": "dbo",
      "table": "Customer",
      "change_lsn": "00000047:00005f88:0002",
      "commit_lsn": "00000047:00005f88:0005",
      "event_serial_no": 1
    },
    "ts_ms": 1698311514501,
    "databaseName": "MyDB",
    "schemaName": "dbo",
    "ddl": "N/A",
    "tableChanges": [
      {
        "type": "ALTER",
        "id": "\"MyDB\".\"dbo\".\"Customer\"",
        "table": {
          "defaultCharsetName": null,
          "primaryKeyColumnNames": [
            "IDX"
          ],
          "columns": [
            {
              "name": "CustomerName",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 20,
              "scale": null,
              "position": 1,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "Age",
              "jdbcType": 4,
              "nativeType": null,
              "typeName": "int",
              "typeExpression": "int",
              "charsetName": null,
              "length": 10,
              "scale": 0,
              "position": 2,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "CustomerAddress",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 200,
              "scale": null,
              "position": 3,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "Salary",
              "jdbcType": 3,
              "nativeType": null,
              "typeName": "decimal",
              "typeExpression": "decimal",
              "charsetName": null,
              "length": 10,
              "scale": 2,
              "position": 4,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "IDX",
              "jdbcType": -5,
              "nativeType": null,
              "typeName": "bigint identity",
              "typeExpression": "bigint identity",
              "charsetName": null,
              "length": 19,
              "scale": 0,
              "position": 5,
              "optional": false,
              "autoIncremented": true,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            }
          ],
          "comment": null
        }
      }
    ]
  }
}
```

