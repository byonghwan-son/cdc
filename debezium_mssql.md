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
* [스키마 변경 순서](#데이터베이스-스키마-변경) 에 따라 CDC가 활성화된 테이블의 구조를 변경

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

#### <u>데이터 변경 이벤트</u>

Debezium SQL Server 커넥터는 각 행 수준 `INSERT`, `UPDATE` 및 `DELETE` 작업에 대한 데이터 변경 이벤트를 생성.  
각 이벤트에는 키와 값이 포함되어 있고 키와 값의 구조는 변경된 테이블에 따라 다름.

Debezium과 Kafka Connect는 지속적인 이벤트 메시지 스트림을 중심으로 설계.  
그러나 이러한 이벤트의 구조는 시간이 지남에 따라 변경될 수 있으며 이는 Consumer가 처리하기 어려울 수 있음.  
이 문제를 해결하기 위해 각 이벤트에는 해당 콘텐츠에 대한 스키마가 포함되어 있으며,  
스키마 레지스트리를 사용하는 경우 Consumer가 레지스트리에서 스키마를 가져오는 데 사용할 수 있는 스키마 ID가 포함되어 있음.  
이렇게 하면 각 이벤트에 자체 포함됩니다.

다음 뼈대 JSON은 변경 이벤트의 기본 네 부분을 보여줌.  
그러나 애플리케이션에서 사용하기로 선택한 Kafka Connect 변환기를 구성하는 방법에 따라  
변경 이벤트에서 이러한 네 부분의 표현이 결정됨.  
필드 schema는 필드를 생성하도록 변환기를 구성한 경우에만 변경 이벤트에 포함.  
마찬가지로, 이벤트 키와 이벤트 페이로드는 이를 생성하도록 변환기를 구성한 경우에만 변경 이벤트에 있음.  
JSON 변환기를 사용하고 4개의 기본 변경 이벤트 부분을 모두 생성하도록 구성하는 경우 변경 이벤트의 구조는 다음과 같음.

```json
{
 "schema": { ①
   ...
  },
 "payload": { ②
   ...
 },
 "schema": { ③
   ...
 },
 "payload": { ④
   ...
 },
}
```

변경 이벤트 기본 내용 개요

|항목|필드|설명|
|-|-|-|
|1|schema|첫 번째 `schema`필드는 이벤트 키의 일부. 이벤트 키 `payload`부분에 무엇이 있는지 설명하는 Kafka Connect 스키마를 지정. 즉, 첫 번째 `schema`필드는 변경된 테이블에 대한 기본 키의 구조를 설명. 즉, 테이블에 기본 키가 없는 경우 고유 키를 설명. <br><br> [`message.key.columns` 커넥터 구성 속성](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-property-message-key-columns)을 설정하여 기본 키를 재정의할 수 있음. 이 경우 첫 번째 스키마 필드는 해당 속성으로 식별되는 키의 구조를 설명|
|2|payload|첫 번째 `payload`필드는 이벤트 키의 일부. 이전 필드에서 설명한 구조를 가지며 `schema`변경된 행에 대한 키를 포함.|
|3|schema|두 번째 `schema`필드는 이벤트 값의 일부. 이벤트 값 `payload` 부분에 무엇이 있는지 설명하는 Kafka Connect 스키마를 지정. 즉, 두 번째는 `schema`변경된 행의 구조를 설명. 일반적으로 이 스키마에는 중첩된 스키마가 포함되어 있음.|
|4|payload|두 번째 `payload`필드는 이벤트 값의 일부. 이는 이전 필드에서 설명한 구조를 가지며 `schema` 변경된 행에 대한 실제 데이터를 포함.|

기본적으로 커넥터는 변경 이벤트 레코드를 이벤트의 원래 테이블과 이름이 동일한 topic으로 스트리밍함.  
자세한 내용은 [주제 이름을 참조](#topic-names-토픽-이름)

> [!WARNING]
> SQL Server 커넥터는 모든 Kafka Connect 스키마 이름이 Avro 스키마 이름 형식을 준수하는지 확인함. 이는 논리 서버 이름이 라틴 문자나 밑줄(즉, az, AZ 또는 _)로 시작해야 함을 의미. 논리 서버 이름의 나머지 문자와 데이터베이스 및 테이블 이름의 각 문자는 라틴 문자, 숫자 또는 밑줄(즉, az, AZ, 0-9 또는 \_)이어야 하며 유효하지 않은 문자가 있으면 밑줄 문자로 대체.
>
> 논리 서버 이름, 데이터베이스 이름 또는 테이블 이름에 잘못된 문자가 포함되어 있고 이름을 서로 구별하는 유일한 문자가 잘못되어 밑줄로 바뀌는 경우 예기치 않은 충돌이 발생할 수 있음.

##### 이벤트 키 변경

변경 이벤트의 키에는 변경된 테이블의 키와 변경된 행의 실제 키에 대한 스키마를 포함. 스키마와 해당 페이로드에는 커넥터가 이벤트를 생성할 당시 변경된 테이블의 기본 키(또는 고유 키 제약 조건)의 각 열에 대한 필드를 포함.

테이블에 대한 변경 이벤트 키의 예가 이어지는 다음 `customers` 테이블을 확인.

*예시 테이블*

```sql
CREATE TABLE MyDB.dbo.Customer (
  CustomerName varchar(20) NOT NULL,
  Age int NOT NULL,
  CustomerAddress varchar(200) NOT NULL,
  Salary decimal(10,2) NOT NULL,
  IDX bigint IDENTITY(1,1) NOT NULL PRIMARY KEY
);
```

##### 변경 이벤트 키 예시

`customers` 테이블 변경 사항을 캡처하는 모든 변경 이벤트에는 동일한 이벤트 키 스키마가 있음.  
`customers` 테이블에 이전 정의가 있는 한 `customers` 테이블에 대한 변경 사항을 캡처하는 모든 변경 이벤트는  
`JSON`에서 다음과 같은 키 구조를 가짐.

```json
{
  "schema": { ①
    "type": "struct",
    "fields": [ ②
      {
        "type": "int64",
        "optional": false,
        "field": "IDX"
      }
    ],
    "optional": false, ③
    "name": "mydb.MyDB.dbo.Customer.Key" ④
  },
  "payload": { ⑤
    "IDX": 2
  }
}
```

*변경 이벤트 키 설명*

|항목|필드|설명|
|-|-|-|
|1|schema||
|2|fields||
|3|optional||
|4|mydb.MyDB.dbo.Customer.Key||
|5|paload||

> [!NOTE]
> `column.exclude.list`및 `column.include.list` 커넥터 구성 속성을 사용하면 해당 테이블 열의 하위 집합만 캡처할 수 있지만  
> 기본 키 또는 고유 키의 모든 열은 항상 이벤트 키를 포함.

> [!WARNING]
> 테이블에 기본 키 또는 고유 키가 없으면 변경 이벤트의 키는 null.  
> 당연하게 기본 키 또는 고유 키 제약 조건이 없는 테이블의 행은 고유하게 식별할 수 없음.

##### 이벤트 값 변경

변경 이벤트의 값은 키보다 조금 더 복잡함. 키와 마찬가지로 값에도 `schema` 섹션과 `payload` 섹션이 있음.  
`schema` 섹션에는 중첩된 필드를 포함하여 `payload` 섹션의 `Envelop` 구조를 설명하는 스키마를 포함.  
데이터를 생성, 업데이트 또는 삭제하는 작업에 대한 변경 이벤트는 모두 `envelop` 구조의 `payload` 값를 갖습니다.

변경 이벤트 키의 예를 보여주기 위해 사용된 것과 동일한 샘플 테이블을 고려해보세요.

*예시 테이블*

```sql
CREATE TABLE MyDB.dbo.Customer (
  CustomerName varchar(20) NOT NULL,
  Age int NOT NULL,
  CustomerAddress varchar(200) NOT NULL,
  Salary decimal(10,2) NOT NULL,
  IDX bigint IDENTITY(1,1) NOT NULL PRIMARY KEY
);
```

이 테이블의 변경 이벤트의 값 부분은 각 이벤트 유형으로 설명됨.

###### 이벤트 만들기

다음 예에서는 `customers` 테이블에 데이터를 생성하는 작업에 대해 커넥터가 생성하는 변경 이벤트의 값 부분을 보여줌.

```json
{
  "schema": { ①
    "type": "struct",
    "fields": [
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "CustomerName"
          },
          {
            "type": "int32",
            "optional": false,
            "field": "Age"
          },
          {
            "type": "string",
            "optional": false,
            "field": "CustomerAddress"
          },
          {
            "type": "bytes",
            "optional": false,
            "name": "org.apache.kafka.connect.data.Decimal",
            "version": 1,
            "parameters": {
              "scale": "2",
              "connect.decimal.precision": "10"
            },
            "field": "Salary"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "IDX"
          }
        ],
        "optional": true,
        "name": "mydb.MyDB.dbo.Customer.Value", ②
        "field": "before"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "CustomerName"
          },
          {
            "type": "int32",
            "optional": false,
            "field": "Age"
          },
          {
            "type": "string",
            "optional": false,
            "field": "CustomerAddress"
          },
          {
            "type": "bytes",
            "optional": false,
            "name": "org.apache.kafka.connect.data.Decimal",
            "version": 1,
            "parameters": {
              "scale": "2",
              "connect.decimal.precision": "10"
            },
            "field": "Salary"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "IDX"
          }
        ],
        "optional": true,
        "name": "mydb.MyDB.dbo.Customer.Value",
        "field": "after"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "version"
          },
          {
            "type": "string",
            "optional": false,
            "field": "connector"
          },
          {
            "type": "string",
            "optional": false,
            "field": "name"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "ts_ms"
          },
          {
            "type": "string",
            "optional": true,
            "name": "io.debezium.data.Enum",
            "version": 1,
            "parameters": {
              "allowed": "true,last,false,incremental"
            },
            "default": "false",
            "field": "snapshot"
          },
          {
            "type": "string",
            "optional": false,
            "field": "db"
          },
          {
            "type": "string",
            "optional": true,
            "field": "sequence"
          },
          {
            "type": "string",
            "optional": false,
            "field": "schema"
          },
          {
            "type": "string",
            "optional": false,
            "field": "table"
          },
          {
            "type": "string",
            "optional": true,
            "field": "change_lsn"
          },
          {
            "type": "string",
            "optional": true,
            "field": "commit_lsn"
          },
          {
            "type": "int64",
            "optional": true,
            "field": "event_serial_no"
          }
        ],
        "optional": false,
        "name": "io.debezium.connector.sqlserver.Source", ③
        "field": "source"
      },
      {
        "type": "string",
        "optional": false,
        "field": "op"
      },
      {
        "type": "int64",
        "optional": true,
        "field": "ts_ms"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "id"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "total_order"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "data_collection_order"
          }
        ],
        "optional": true,
        "name": "event.block",
        "version": 1,
        "field": "transaction"
      }
    ],
    "optional": false,
    "name": "mydb.MyDB.dbo.Customer.Envelope", ④
    "version": 1
  },
  "payload": { ⑤ 
    "before": null, ⑥
    "after": { ⑦
      "CustomerName": "john doe",
      "Age": 33,
      "CustomerAddress": "New York",
      "Salary": "AvrwgA==",
      "IDX": 9
    },
    "source": { ⑧
      "version": "2.4.0.Final",
      "connector": "sqlserver",
      "name": "mydb",
      "ts_ms": 1698390150850,
      "snapshot": "false",
      "db": "MyDB",
      "sequence": null,
      "schema": "dbo",
      "table": "Customer",
      "change_lsn": "0000004c:000007d0:0003",
      "commit_lsn": "0000004c:000007d0:0004",
      "event_serial_no": 1
    },
    "op": "c", ⑨
    "ts_ms": 1698390156393, ⑩
    "transaction": null
  }
}
```

이벤트 생성 값 필드 에 대한 설명

|항목|필드|설명|
|-|-|-|
|1|schema|payload 값의 구조를 설명하는 스키마.변경 이벤트의 값 스키마는 커넥터가 특정 테이블에 대해 생성하는 모든 변경 이벤트에서 동일.|
|2|name|스키마 섹션에서 각 필드 이름은 payload 값에 있는 필드의 스키마를 지정.<br><br>`mydb.MyDB.dbo.Customer.Value`는 페이로드의 이전 및 이후 필드에 대한 스키마. 이 스키마는 customer 테이블에만 적용.<br><br>이전 및 이후 필드의 스키마 이름은 `logicalName.databaseschemaName.tableName.Value` 형식으로 되어 있어 스키마 이름이 데이터베이스에서 고유함.이는 Avro 변환기를 사용할 때 각 논리 소스의 각 테이블에 대한 결과 Avro 스키마가 자체적인 발전과 기록을 가짐을 의미.|
|3|name|`io.debezium.connector.sqlserver.Source`는 `payload`내의 `source` 필드의 스키마.이 스키마는 SQL Server 커넥터에만 적용. 커넥터는 생성하는 모든 이벤트에 이를 사용.|
|4|name|`mydb.MyDB.dbo.Customer.Envelope`는 `payload`의 전체 구조에 대한 스키마. 여기서 `mydb`은 커넥터 이름, `MyDB.dbo`는 데이터베이스 스키마 이름, `Customer`은 테이블입니다.|
|5|payload|변경 이벤트가 제공하는 실제 값.<br><br>이벤트의 JSON 표현은 이벤트가 설명하는 행보다 훨씬 더 큰 것처럼 보일 수 있음. 이는 JSON 표현에 메시지의 스키마와 페이로드 부분이 포함되어야 하기 때문. 그러나 [Avro 변환기](https://debezium.io/documentation/reference/stable/configuration/avro.html#avro-serialization)를 사용하면 커넥터가 Kafka Topic으로 스트리밍하는 메시지의 크기를 크게 줄일 수 있음.|
|6|before|이벤트가 발생하기 전의 행 상태를 지정하는 선택적 필드.<br>이 예에서와 같이 `op`필드가 생성을 의미하는 `c`인 경우 이 변경 이벤트는 새 콘텐츠에 대한 것이므로 `before`필드는 null.|
|7|after|이벤트가 발생한 후의 행 상태를 지정하는 선택적 필드.<br>이 예에서 `after` 필드에는 새 행의 `CustomerName`, `Age`, `CustomerAddress`, `Salary` 및 `IDX` 열 값을 포함.|
|8|source|이벤트의 소스 메타데이터를 설명하는 필수 필드. 이 필드에는 이벤트의 출처, 이벤트가 발생한 순서 및 이벤트가 동일한 트랜잭션의 일부인지 여부와 관련하여 이 이벤트를 다른 이벤트와 비교하는 데 사용할 수 있는 정보를 포함. 소스 메타데이터에는 다음을 포함.<br><br>* Debezium 버전<br>* 커넥터 유형 및 이름<br>* 데이터베이스 및 스키마 이름<br>* 데이터베이스가 변경된 시점의 타임 스탬프<br>* 이벤트가 스냅샷의 일부인 경우<br>* 새 행을 포함하는 테이블의 이름<br>* 서버 로그 오프셋|
|9|op|커넥터가 이벤트를 생성하게 만든 작업 유형을 설명하는 필수 문자열. 이 예에서 는 `c` 작업이 행을 생성했음을 표현. 유효한 값<br><br>* c = Create / Insert<br>* u = Update<br>* d = Delete<br>* r = 읽기 (스냅샷에만 적용)|
|10|ts_ms|커넥터가 이벤트를 처리한 시간을 표시하는 선택적 필드. Event Message Envelop에서 시간은 Kafka Connect 작업을 실행하는 JVM 안에서의 시스템 시간을 기반으로 함.<br><br>소스 객체에서 ts_ms는 데이터베이스에 변경 사항이 커밋된 시간. `payload.source.ts_ms` 값과 `payload.ts_ms` 값을 비교하면 소스 데이터베이스 업데이트와 Debezium 사이의 지연을 확인 가능. `(1698390156393 -1698390150850 = 5543)`|

###### 업데이트 이벤트

샘플 `Customer` 테이블의 업데이트에 대한 변경 이벤트 값은 해당 테이블에 대한 생성 이벤트와 동일한 스키마를 가짐. 마찬가지로 이벤트 값의 페이로드도 동일한 구조를 가짐.  
그러나 페이로드의 업데이트 이벤트에는 다른 값이 포함되어 있습니다. 다음은 `Customer` 테이블의 업데이트에 대해 커넥터가 생성하는 이벤트의 변경 이벤트 값의 예.

```json
{
  "schema": { ... },
  "payload": {
    "before": { ①
      "CustomerName": "손주원",
      "Age": 15,
      "CustomerAddress": "해운대구",
      "Salary": "AA==",
      "IDX": 8
    },
    "after": { ②
      "CustomerName": "손주원",
      "Age": 15,
      "CustomerAddress": "해운대구",
      "Salary": "A+g=",
      "IDX": 8
    },
    "source": { ③
      "version": "2.4.0.Final",
      "connector": "sqlserver",
      "name": "mydb",
      "ts_ms": 1698369375027,
      "snapshot": "false",
      "db": "MyDB",
      "sequence": null,
      "schema": "dbo",
      "table": "Customer",
      "change_lsn": "0000004a:000014e8:0002",
      "commit_lsn": "0000004a:000014e8:0003",
      "event_serial_no": 2
    },
    "op": "u", ④
    "ts_ms": 1698369376558, ⑤
    "transaction": null
  }
}
```

|항목|필드|설명|
|-|-|-|
|1|before|이벤트가 발생하기 전 행의 상태를 지정하는 선택적 필드. 업데이트 이벤트 값에서 이전 필드에는 각 테이블 열에 대한 필드와 데이터베이스 커밋 전에 해당 열에 있었던 값을 포함. 이 예에서 `Salary` 값은 `AA==`.<br>(Decimal type의 경우 소숫점 단위로 인해 일반 숫자가 아닌 인코딩된 형태로 표시)|
|2|after|이벤트가 발생한 후 행의 상태를 지정하는 선택적 필드. 이전 및 이후 구조를 비교하여 이 행에 대한 업데이트가 무엇인지 확인 가능. 이 예에서 `Salary` 값은 이제 `A+g=`.<br>(Decimal type의 경우 소숫점 단위로 인해 일반 숫자가 아닌 인코딩된 형태로 표시)|
|3|source|이벤트의 소스 메타데이터를 설명하는 필수 필드. 이 필드에는 이벤트의 출처, 이벤트가 발생한 순서 및 이벤트가 동일한 트랜잭션의 일부인지 여부와 관련하여 이 이벤트를 다른 이벤트와 비교하는 데 사용할 수 있는 정보를 포함. 소스 메타데이터에는 다음을 포함.<br><br>* Debezium 버전<br>* 커넥터 유형 및 이름<br>* 데이터베이스 및 스키마 이름<br>* 데이터베이스가 변경된 시점의 타임 스탬프<br>* 이벤트가 스냅샷의 일부인 경우<br>* 새 행을 포함하는 테이블의 이름<br>* 서버 로그 오프셋<br><br>`event_serial_no` 필드는 커밋이 동일하고 LSN이 변경된 이벤트를 구분함. 이 필드의 값이 1이 아닌 경우의 일반적인 상황은 아래와 같음.<br><br>* 업데이트는 SQL Server의 CDC 변경 테이블에 두 개의 이벤트를 생성하므로 업데이트 이벤트의 값은 2로 설정됨.([자세한 내용은 소스 설명서 참조](https://learn.microsoft.com/en-us/sql/relational-databases/system-tables/cdc-capture-instance-ct-transact-sql?view=sql-server-2017)). 첫 번째 이벤트에는 이전 값을 포함하고 두 번째 이벤트에는 새 값을 포함. 커넥터는 첫 번째 이벤트의 값을 사용하여 두 번째 이벤트를 생성. 커넥터는 첫 번째 이벤트를 삭제.<br>* 기본 키가 업데이트되면 SQL Server는 두 가지 이벤트를 보냄. 이전 기본 키 값이 있는 레코드를 제거하는 삭제 이벤트와 새 기본 키 값이 있는 레코드를 추가하는 생성 이벤트. 두 작업 모두 동일한 커밋을 공유하고 LSN을 변경하며 해당 이벤트 번호는 각각 1과 2임.|
|4|op|커넥터가 이벤트를 생성하게 만든 작업 유형을 설명하는 필수 문자열. 이 예에서 는 `u` 작업이 행을 변경했음을 표현. 유효한 값<br><br>* c = Create / Insert<br>* u = Update<br>* d = Delete<br>* r = 읽기 (스냅샷에만 적용)|
|5|ts_ms|커넥터가 이벤트를 처리한 시간을 표시하는 선택적 필드. Event Message Envelop에서 시간은 Kafka Connect 작업을 실행하는 JVM 안에서의 시스템 시간을 기반으로 함.<br><br>소스 객체에서 ts_ms는 데이터베이스에 변경 사항이 커밋된 시간. `payload.source.ts_ms` 값과 `payload.ts_ms` 값을 비교하면 소스 데이터베이스 업데이트와 Debezium 사이의 지연을 확인 가능. `(1698369376558 -1698369375027 = 1531)`|

> [!NOTE]
> 행의 기본/고유 키에 대한 열을 업데이트하면 행의 키 값이 변경됨.  
> 키가 변경되면 Debezium은 행에 대한 이전 키를 사용하는 삭제 이벤트와 삭제 표시 이벤트, 행에 대한 새 키를 사용하는 생성 이벤트 등 세 가지 이벤트를 출력함.

###### 이벤트 삭제

삭제 이벤트의 값은 동일한 테이블에 대한 생성 및 업데이트 이벤트와 동일한 스키마 부분을 갖습니다. 샘플 Customer 테이블에 대한 삭제 이벤트의 페이로드 부분은 다음과 같습니다.

```json
{
  "schema": { ... },
  "payload": {
    "before": {
      "CustomerName": "홍순남",
      "Age": 40,
      "CustomerAddress": "연제구",
      "Salary": "D0JA",
      "IDX": 6
    },
    "after": null,
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
    "op": "d",
    "ts_ms": 1698305297092,
    "transaction": null
  }
}
```

|항목|필드|설명|
|-|-|-|
|1|before|이벤트가 발생하기 전 행의 상태를 지정하는 선택적 필드. 삭제 이벤트 값에서 이전 필드에는 데이터베이스 커밋으로 삭제되기 전 행에 있던 값을 포함.|
|2|after|이벤트가 발생한 후 행의 상태를 지정하는 선택적 필드. 삭제 이벤트 값에서 after 필드는 null이며 이는 행이 더 이상 존재하지 않음을 나타냄.|
|3|source|이벤트의 소스 메타데이터를 설명하는 필수 필드. 삭제 이벤트 값에서 소스 필드 구조는 동일한 테이블에 대한 생성 및 업데이트 이벤트와 동일. 많은 소스 필드 값도 동일. 삭제 이벤트 값에서 ts_ms 및 pos 필드 값과 기타 값이 변경되었을 수 있음. 그러나 삭제 이벤트 값의 소스 필드는 동일한 메타데이터를 제공.<br><br>* Debezium 버전<br>* 커넥터 유형 및 이름<br>* 데이터베이스 및 스키마 이름<br>* 데이터베이스가 변경된 시점의 타임 스탬프<br>* 이벤트가 스냅샷의 일부인 경우<br>* 새 행을 포함하는 테이블의 이름<br>* 서버 로그 오프셋|
|4|op|커넥터가 이벤트를 생성하게 만든 작업 유형을 설명하는 필수 문자열. 이 예에서 는 `d` 작업이 행을 삭제했음을 표현. 유효한 값<br><br>* c = Create / Insert<br>* u = Update<br>* d = Delete<br>* r = 읽기 (스냅샷에만 적용)|
|5|ts_ms|커넥터가 이벤트를 처리한 시간을 표시하는 선택적 필드. Event Message Envelop에서 시간은 Kafka Connect 작업을 실행하는 JVM 안에서의 시스템 시간을 기반으로 함.<br><br>소스 객체에서 ts_ms는 데이터베이스에 변경 사항이 커밋된 시간. `payload.source.ts_ms` 값과 `payload.ts_ms` 값을 비교하면 소스 데이터베이스 업데이트와 Debezium 사이의 지연을 확인 가능. `(1698369376558 -1698369375027 = 1531)`|

SQL Server 커넥터 이벤트는 Kafka 로그 압축 과 함께 작동하도록 설계됨. 로그 압축을 사용하면 모든 키에 대해 최소한 최신 메시지가 유지되는 한 일부 오래된 메시지를 제거 가능. 이를 통해 Kafka는 토픽에 완전한 데이터 세트가 포함되어 있고 키 기반 상태를 다시 로드하는 데 사용될 수 있도록 하면서 저장 공간을 최소화 함.

##### 삭제표시 이벤트

행이 삭제되면 Kafka가 동일한 키를 가진 모든 이전 메시지를 제거할 수 있기 때문에 삭제 이벤트 값은 로그 압축과 함께 계속 작동함. 그러나 Kafka가 동일한 키를 가진 모든 메시지를 제거하려면 메시지 값이 `null`이어야 함. 이를 가능하게 하기 위해 Debezium의 SQL Server 커넥터가 삭제 이벤트를 생성한 후 커넥터는 동일한 키이지만 `null`값을 갖는 특수 삭제 표시 이벤트를 생성.

#### 트랜잭션 메타 데이터

Debezium은 트랜잭션 경계를 나타내고 데이터 변경 이벤트 메시지를 강화하는 이벤트를 생성

> [!NOTE] Debezium이 트랜잭션 메타데이터를 수신하는 시점에 대한 제한  
> Debezium은 커넥터를 배포한 후에 발생하는 트랜잭션에 대해서만 메타데이터를 등록하고 수신.  
> 커넥터를 배포하기 전에 발생하는 트랜잭션에 대한 메타데이터는 사용할 수 없음.

데이터베이스 트랜잭션은 BEGIN 및 END 키워드 사이에 포함된 명령문 블록으로 표시. Debezium은 모든 트랜잭션에서 BEGIN 및 END 구분 기호에 대한 트랜잭션 경계 이벤트를 생성. 거래 경계 이벤트에는 다음 필드를 포함.

***status***  
　`BEGIN` 또는 `END`

***id***  
　고유한 transaction 식별자(문자열)

***ts_ms***  
　데이터 소스에서 트랜잭션 경계 이벤트(`BEGIN` 또는 `END` 이벤트)가 발생한 시간.  
　데이터 소스가 Debezium에 이벤트 시간을 제공하지 않는 경우 필드는 대신 Debezium이 이벤트를 처리하는 시간을 표시.

***event_count(`END` 이벤트용)***  
　트랜잭션에서 발생하는 총 이벤트 수

***data_collections(`END` 이벤트용)***  
　데이터 컬렉션에서 발생한 변경 사항에 대해 커넥터가 내보내는 이벤트 수를 나타내는 `data_collection` 및 `event_count` 요소 쌍의 배열입니다.

> [!WARNING]
> Debezium에서는 트랜잭션이 언제 종료되었는지 확실하게 식별할 수 있는 방법이 없음.  
> 따라서 트랜잭션 END 마커는 다른 트랜잭션의 첫 번째 이벤트가 도착한 후에만 방출됨.  
> 이로 인해 트래픽이 적은 시스템의 경우 END 마커 전달의 지연 발생.

일반적인 트랜잭션 경계 메시지의 예

```json
{
  "status": "BEGIN",
  "id": "00000025:00000d08:0025",
  "ts_ms": 1486500577125,
  "event_count": null,
  "data_collections": null
}

{
  "status": "END",
  "id": "00000025:00000d08:0025",
  "ts_ms": 1486500577691,
  "event_count": 2,
  "data_collections": [
    {
      "data_collection": "testDB.dbo.testDB.tablea",
      "event_count": 1
    },
    {
      "data_collection": "testDB.dbo.testDB.tableb",
      "event_count": 1
    }
  ]
}
```

`topic.transaction` 옵션을 통해 재정의되지 않는 한 트랜잭션 이벤트는 `<topic.prefix>`.transaction이라는 topic에 기록됨.

##### 변경 데이터 이벤트 강화(change data event enrichment)

트랜잭션 메타데이터가 활성화되면 데이터 메시지 `Envelope`은 새 `transaction` 필드로 강화됩니다.  
이 필드는 필드 복합 형식으로 모든 이벤트에 대한 정보를 제공합니다.

***id***  
　고유한 transaction 식별자(문자열)

***total_order***  
　트랜잭션에 의해 발생한 모든 이벤트 중 해당 이벤트의 절대 위치

***data_collection_order***  
　트랜잭션에서 발생한 모든 이벤트 중 해당 이벤트의 `data collection` 위치

일반적인 메세지 모양

```json
{
  "before": null,
  "after": {
    "pk": "2",
    "aa": "1"
  },
  "source": {
...
  },
  "op": "c",
  "ts_ms": "1580390884335",
  "transaction": {
    "id": "00000025:00000d08:0025",
    "total_order": "1",
    "data_collection_order": "1"
  }
}
```

#### 데이터 타입 매핑

Debezium SQL Server 커넥터는 행이 존재하는 테이블과 유사하게 구성된 이벤트를 생성하여 테이블 행 데이터의 변경 사항을 나타냄. 각 이벤트에는 행의 열 값을 나타내는 필드가 포함되어 있음. 이벤트가 작업에 대한 열 값을 나타내는 방식은 열의 SQL 데이터 유형에 따라 다름. 이 경우 커넥터는 각 SQL Server 데이터 유형의 필드를 *리터럴* 유형과 *의미* 유형 모두에 매핑함.

커넥터는 SQL Server 데이터 유형을 *리터럴* 및 *의미* 유형 모두에 매핑 가능.

*리터럴 타입*  
　Kafka Connect 스키마 유형(`INT8, INT16, INT32, INT64, FLOAT32, FLOAT64, BOOLEAN, STRING, BYTES, ARRAY, MAP 및 STRUCT`)을 사용하여 값을 문자 그대로 표현하는 방법을 설명.

*Semantic(의미) 타입*  
　Kafka Connect 스키마가 필드에 대한 Kafka Connect 스키마 이름을 사용하여 필드의 *의미*(*meaning*)를 캡처하는 방법을 설명.

기본 데이터 유형 변환이 요구 사항을 충족하지 않는 경우 커넥터에 대한 [사용자 지정 변환기](https://debezium.io/documentation/reference/stable/development/converters.html#custom-converters)를 만들 수 있습니다.

##### 기본 타입

커넥터가 기본 SQL Server 데이터 유형을 매핑하는 방법

|SQL SERVER 데이터 타입|리터럴 타입(스키마 타입)|의미 타입(스키마 이름) 및 메모|
|-|-|-|
|BIT|BOOLEAN|해당사항 없음|
|TINYINT|INT16|해당사항 없음|
|SMALLINT|INT16|해당사항 없음|
|INT|INT32|해당사항 없음|
|BIGINT|INT64|해당사항 없음|
|REAL|FLOAT32|해당사항 없음|
|FLOAT[(N)]|FLOAT64|해당사항 없음|
|CHAR[(N)]|STRING|해당사항 없음|
|VARCHAR[(N)]|STRING|해당사항 없음|
|TEXT|STRING|해당사항 없음|
|NCHAR[(N)]|STRING|해당사항 없음|
|NVARCHAR[(N)]|STRING|해당사항 없음|
|NTEXT|STRING|해당사항 없음|
|XML|STRING|io.debezium.data.Xml<br><br>XML 문서의 문자열 표현 포함|
|DATETIMEOFFSET[(P)]|STRING|io.debezium.time.ZonedTimestamp<br><br>시간대 정보가 포함된 타임스탬프의 문자열 표현<br>(여기서 시간대는 GMT임)|

다른 데이터 타입 매핑은 다음 섹션에서 설명함.

열의 기본값이 있는 경우 해당 필드의 Kafka Connect 스키마에 전파. 변경 메시지에는 필드의 기본값이 포함되므로(명시적인 열 값이 제공되지 않은 경우) 스키마에서 기본값을 가져올 필요가 거의 없음. Confluent 스키마 레지스트리와 함께 [Avro를 직렬화 형식으로 사용](https://debezium.io/documentation/reference/stable/configuration/avro.html)할 때 기본값을 전달하면 호환성 규칙을 충족하는 데 도움이 됨.

##### 시간 타입

SQL Server의 `DATETIMEOFFSET` 데이터 타입(표준 시간대 정보 포함) 외에 다른 시간형 타입은 `time.precision.mode` 구성 속성의 값에 따라 달라짐. `time.precision.mode` 구성 속성이 `adaptive`(기본값)으로 설정된 경우, 커넥터는 이벤트가 데이터베이스의 값을 정확하게 나타내도록 열의 데이터 유형 정의를 기반으로 시간형 타입에 대한 리터럴 유형 및 의미 유형을 결정함.

|SQL Server 데이터 유형|리터럴 유형(스키마 유형)|의미 유형(스키마 이름) 및 메모 (에포크-epoch 1970.1.1)|
|-|-|-|
|DATE|INT32|io.debezium.time.Date<br><br>에포크 이후의 일수.|
|TIME(0), TIME(1), TIME(2),TIME(3)|INT32|io.debezium.time.Time<br><br>자정 이후의 밀리초 수를 나타내며 시간대 정보는 포함하지 않음.|
|TIME(4), TIME(5),TIME(6)|INT64|io.debezium.time.MicroTime<br><br>자정 이후의 마이크로초 수를 나타내며 시간대 정보는 포함하지 않음.|
|TIME(7)|INT64|io.debezium.time.NanoTime<br><br>자정 이후의 나노초 수를 나타내며 시간대 정보는 포함하지 않음.|
|DATETIME|INT64|io.debezium.time.Timestamp<br><br>에포크 이후의 밀리초 수를 나타내며 시간대 정보는 포함하지 않음.|
|SMALLDATETIME|INT64|io.debezium.time.Timestamp<br><br>에포크 이후의 밀리초 수를 나타내며 시간대 정보는 포함하지 않음.|
|DATETIME2(0), DATETIME2(1), DATETIME2(2),DATETIME2(3)|INT64|io.debezium.time.Timestamp<br><br>에포크 이후의 밀리초 수를 나타내며 시간대 정보는 포함하지 않음.|
|DATETIME2(4), DATETIME2(5),DATETIME2(6)|INT64|io.debezium.time.MicroTimestamp<br><br>epoch 이후의 마이크로초 수를 나타내며 시간대 정보는 포함하지 않음.|
|DATETIME2(7)|INT64|io.debezium.time.NanoTimestamp<br><br>에포크 이후의 나노초 수를 나타내며 시간대 정보는 포함하지 않음.|

`time.precision.mode` 구성 속성이 `connect`로 설정된 경우 커넥터는 사전 정의된 Kafka Connect 논리 타입을 사용. 이는 소비자가 내장된 Kafka Connect 논리 유형에 대해서만 알고 있고 가변 정밀도 시간 값을 처리할 수 없는 경우에 유용함. 반면에 SQL Server는 10분의 1마이크로초의 정밀도를 지원하므로 `connect`시간 정밀도 모드를 사용하는 커넥터에서 생성된 이벤트는 데이터베이스 열의 소수 *초 정밀도 값*이 3보다 큰 경우 **정밀도가 손실됨**.

|SQL Server 데이터 유형|리터럴 유형(스키마 유형)|의미 유형(스키마 이름) 및 메모|
|-|-|-|
|DATE|INT32|org.apache.kafka.connect.data.Date<br><br>epoch 이후의 일수|
|TIME([P])|INT64|org.apache.kafka.connect.data.Time<br><br>자정 이후의 시간(밀리초)을 나타내며 시간대 정보는 포함하지 않음. SQL Server에서는 P가 0-7 범위에 있도록 허용하여 최대 10분의 1마이크로초 정밀도를 저장. 하지만 이 모드에서는 P > 3일 때 정밀도를 손실.(`second` 아래 최대 3자리)|
|DATETIME|INT64|org.apache.kafka.connect.data.Timestamp<br><br>epoch 이후의 밀리초 수를 나타내며 시간대 정보를 포함하지 않음.|
|SMALLDATETIME|INT64|org.apache.kafka.connect.data.Timestamp<br><br>epoch 이후의 밀리초 수를 나타내며 시간대 정보는 포함하지 않음.|
|DATETIME2|INT64|org.apache.kafka.connect.data.Timestamp<br><br>epoch 이후의 밀리초 수를 나타내며 시간대 정보를 포함하지 않음. SQL Server에서는 P가 0-7 범위에 있도록 허용하여 최대 10분의 1마이크로초 정밀도를 저장. 하지만 이 모드에서는 P > 3일 때 정밀도를 손실.(`second` 아래 최대 3자리)|

###### timestamp values

`DATETIME`, `SMALLDATETIME` 및 `DATETIME2` 유형은 시간대 정보가 없는 타임스탬프를 표시. 이러한 열은 UTC를 기반으로 하는 동등한 Kafka Connect 값으로 변환됨. 예를 들어 `DATETIME2` 값 "2018-06-20 15:13:16.945104"는 "1529507596945104" 값을 가진 `io.debezium.time.MicroTimestamp`로 표시.

Kafka Connect 및 Debezium을 실행하는 JVM의 시간대는 이 변환에 영향을 미치지 않음.

##### Decimal Values

Debezium 커넥터는 [`decimal.handling.mode` connector 구성 속성](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-property-decimal-handling-mode)의 설정에 따라 소수를 처리.

*`decimal.handling.mode=precise`*

|SQL Server Type|Literal type<br>(schema type)|Semantic type(schema name)|
|-|-|-|
|NUMERI[(P[,S])]|BYTES|org.apache.kafka.connect.data.Decimal<br>scale 스키마 매개변수에는 소수점이 이동한 자릿수를 나타내는 정수를 포함.|
|DECIMAL[(P[,S])]|BYTES|org.apache.kafka.connect.data.Decimal<br>scale 스키마 매개변수에는 소수점이 이동한 자릿수를 나타내는 정수를 포함.|
|SMALLMONEY|BYTES|org.apache.kafka.connect.data.Decimal<br>scale 스키마 매개변수에는 소수점이 이동한 자릿수를 나타내는 정수를 포함.|
|MONEY|BYTES|org.apache.kafka.connect.data.Decimal<br>scale 스키마 매개변수에는 소수점이 이동한 자릿수를 나타내는 정수를 포함.|

*`decimal.handling.mode=double`*

|SQL Server Type|Literal type|Semantic type|
|-|-|-|
|NUMERIC[(M[,D])]|FLOAT64|n/a - 해당사항 없음|
|DECIMAL[(M[,D])]|FLOAT64|n/a - 해당사항 없음|
|SMALLMONEY[(M[,D])]|FLOAT64|n/a - 해당사항 없음|
|MONEY[(M[,D])]|FLOAT64|n/a - 해당사항 없음|

*`decimal.handling.mode=string`*

|SQL Server Type|Literal type|Semantic type|
|-|-|-|
|NUMERIC[(M[,D])]|STRING|n/a - 해당사항 없음|
|DECIMAL[(M[,D])]|STRING|n/a - 해당사항 없음|
|SMALLMONEY[(M[,D])]|STRING|n/a - 해당사항 없음|
|MONEY[(M[,D])]|STRING|n/a - 해당사항 없음|

## [Setting up SQL Server](CDC환경설정.md)

### SQL Server 캡처 작업 에이전트 구성이 서버 로드 및 대기 시간에 미치는 영향

데이터베이스 관리자가 원본 테이블에 대한 변경 데이터 캡처를 활성화하면 캡처 작업 에이전트가 실행되기 시작.  
에이전트는 트랜잭션 로그에서 새 변경 이벤트 레코드를 읽고 해당 이벤트 레코드를 변경 데이터 테이블에 복제.  
원본 테이블에 변경 내용이 커밋되는 시간과 해당 변경 테이블에 변경 내용이 나타나는 시간 사이에는 항상 짧은 대기 시간 간격이 존재.  
이 대기 시간 간격은 소스 테이블에서 변경 사항이 발생하는 시점과 Debezium이 Apache Kafka로 스트리밍할 수 있게 되는 시점 사이의 간격.

이상적으로는 데이터 변경에 신속하게 응답해야 하는 애플리케이션의 경우 원본 테이블과 변경 테이블 간의 긴밀한 동기화를 유지하는 것이 좋음.  
변경 이벤트를 최대한 빠르게 지속적으로 처리하기 위해 캡처 에이전트를 실행하면 처리량이 증가하고 대기 시간이 줄어들 수 있음.  
즉, 이벤트가 발생한 후 가능한 한 빨리, 거의 실시간으로 새 이벤트 레코드로 변경 테이블을 채울 수 있다고 상상할 수 있음.  
그러나 반드시 그런 것은 아님. 보다 즉각적인 동기화를 추구하면 성능 저하가 발생.  
캡처 작업 에이전트가 데이터베이스에 새 이벤트 레코드를 쿼리할 때마다 데이터베이스 호스트의 CPU 로드가 증가.  
서버에 대한 추가 로드는 전체 데이터베이스 성능에 부정적인 영향을 미칠 수 있으며  
특히 데이터베이스 사용이 가장 많은 시간 동안 트랜잭션 효율성을 감소시킬 수 있음.

데이터베이스가 서버가 더 이상 캡처 에이전트의 활동 수준을 지원할 수 없는 지점에 도달하는지 알 수 있도록  
데이터베이스 메트릭을 모니터링하는 것이 중요.  
성능 문제가 발견되면 허용 가능한 수준의 대기 시간으로 데이터베이스 호스트의 전체 CPU 로드 균형을 맞추는 데 도움이 되도록  
수정할 수 있는 SQL Server 캡처 에이전트 설정이 있음.

### SQL Server 캡처 작업 에이전트 구성 매개변수

SQL Server에서 캡처 작업 에이전트의 동작을 제어하는 매개 변수는 SQL Server 테이블 msdb.dbo.cdc_jobs에 정의.  
캡처 작업 에이전트를 실행하는 동안 성능 문제가 발생하는 경우 sys.sp_cdc_change_job 저장 프로시저를 실행하고  
새 값을 제공하여 CPU 로드를 줄이도록 캡처 작업 설정을 조정 해야 함.

> [!NOTE]  
> SQL Server 캡처 작업 에이전트 매개 변수를 구성하는 방법에 대한 구체적인 지침은 이 문서의 범위를 벗어납니다.

다음 매개변수는 Debezium SQL Server 커넥터와 함께 사용할 캡처 에이전트 동작을 수정하는 데 가장 중요.

* ***pollinginterval***
  * 캡처 에이전트가 로그 스캔 주기 사이에 대기하는 시간(초)을 지정.
  * 값이 높을수록 데이터베이스 호스트의 로드가 줄어들고 대기 시간이 늘어남.
  * 값 0은 검색 사이에 대기하지 않음을 지정.
  * 기본값은 5.

* ***maxtrans***
  * 각 로그 스캔 주기 동안 처리할 최대 트랜잭션 수를 지정.  
    캡처 작업은 지정된 수의 트랜잭션을 처리한 후 다음 검색이 시작되기 전에  
    pollinginterval이 지정하는 시간 동안 일시 중지.
  * 값이 낮을수록 데이터베이스 호스트의 로드가 줄어들고 대기 시간이 늘어남.
  * 기본값은 500.

* ***maxscans***
  * 데이터베이스 트랜잭션 로그의 전체 내용을 캡처하기 위해 캡처 작업이 시도할 수 있는 검색 주기 수에 대한 제한을 지정.  
    Continuous 매개변수가 1로 설정된 경우 작업은 검색을 다시 시작하기 전에 pollinginterval이 지정하는 시간 동안 일시 중지됩니다.
  * 값이 낮을수록 데이터베이스 호스트의 로드가 줄어들고 대기 시간이 늘어남.
  * 기본값은 10.

## 전개

### 카프카 커넥터 전개 방법 참조

### 커넥터 속성

Debezium SQL Server 커넥터에는 애플리케이션에 적합한 커넥터 동작을 달성하는 데 사용할 수 있는 다양한 구성 속성이 존재.  
많은 속성이 기본값을 가짐.

속성의 상세 내역

* [필수 커넥터 구성 속성](#필수-debezium-sql-server-커넥터-구성-속성)
* [고급 커넥터 구성 속성](#고급-sql-server-커넥터-구성-속성)
  * Debezium이 데이터베이스 스키마 기록 항목에서 읽는 이벤트를 처리하는 방법을 제어하는 ​[​데이터베이스 스키마 기록 커넥터 구성 속성](#debezium-sql-server-커넥터-데이터베이스-스키마-기록-구성-속성)
  * [Pass-through 데이터베이스 스키마 기록 속성](#생산자-및-소비자-클라이언트-구성을-위한-pass-through-데이터베이스-스키마-기록-속성)
* [데이터베이스 드라이버 의 동작을 제어하는 Pass-through 데이터베이스 드라이버 속성](#debezium-sql-server-커넥터-pass-through-데이터베이스-드라이버-구성-속성)

#### 필수 Debezium SQL Server 커넥터 구성 속성

기본값을 사용할 수 없는 경우 다음 구성 속성이 필요

|속성|기본 값|설명|
|-|-|-|
|name|기본값 없음|커넥터의 고유 이름. 동일한 이름으로 다시 등록을 시도하면 실패. (이 속성은 모든 Kafka Connect 커넥터에 필요.)|
|connector.class|기본값 없음|커넥터에 대한 Java 클래스의 이름. SQL Server 커넥터에는 항상 `io.debezium.connector.sqlserver.SqlServerConnector` 값을 사용.|
|tasks.max|1|커넥터가 데이터베이스 인스턴스에서 데이터를 캡처하는 데 사용할 수 있는 최대 작업 수를 지정. `Database.names` 목록에 둘 이상의 요소가 포함된 경우 이 속성의 값을 목록에 있는 요소 수보다 작거나 같은 수로 늘릴 수 있음.|
|database.hostname|기본값 없음|SQL Server 데이터베이스 서버의 IP 주소 또는 호스트 이름.|
|database.port|1433|SQL Server 데이터베이스 서버의 정수 포트 번호.|
|database.user|기본값 없음|SQL Server 데이터베이스 서버에 연결할 때 사용할 사용자 이름. Pass-through 속성을 사용하여 구성할 수 있는 Kerberos 인증을 사용하는 경우 생략할 수 있음.|
|database.password|기본값 없음|SQL Server 데이터베이스 서버에 연결할 때 사용할 비밀번호.|
|database.instance|기본값 없음|SQL Server 명명된 인스턴스의 인스턴스 이름을 지정.|
|database.names|기본값 없음|변경 사항을 스트리밍해서 가져올 SQL Server 데이터베이스 이름을 쉼표로 구분한 목록.|
|`topic.prefix`|기본값 없음|Debezium에서 캡처할 SQL Server 데이터베이스 서버에 대한 네임스페이스를 제공하는 Topic 접두사. 접두사는 이 커넥터에서 레코드를 수신하는 모든 Kafka Topic 이름의 접두사로 사용되므로 다른 모든 커넥터에서 고유해야 함. 데이터베이스 서버 논리적 이름에는 영숫자, 하이픈, 점, 밑줄만 사용해야 함.<br><br> * WARNING <br>이 속성의 값을 변경하지 마십시오. 이름 값을 변경하면 다시 시작한 후에 원래 Topic에 이벤트를 계속 생성하는 대신, 커넥터는 새로운 이름을 기반으로 하는 Topic에 후속 이벤트를 생성. 또한 커넥터는 해당 데이터베이스 schema history topic을 복구할 수 없음.|
|schema.include.list|기본값 없음|선택적. 변경 사항을 캡처하려는 스키마 이름과 일치하며 쉼표로 구분된 정규식 목록. schema.include.list에 포함되지 않은 모든 스키마 이름은 변경 사항 캡처에서 제외. 기본적으로 커넥터는 모든 비시스템 스키마에 대한 변경 사항을 캡처.<br>스키마 이름과 일치시키기 위해 Debezium은 고정된 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 스키마의 전체 이름 문자열과 일치. 스키마 이름에 존재할 수 있는 하위 문자열과 일치하지 않음(?).<br>구성에 이 속성을 포함하는 경우 Schema.exclude.list 속성은 설정하지 마세요.|
|schema.exclude.list|기본값 없음|선택적. 변경 사항을 캡처하지 않으려는 스키마 이름과 일치하며 쉼표로 구분된 정규식 목록. schema.exclude.list에 이름이 포함되지 않은 모든 스키마에는 시스템 스키마를 제외하고 변경 사항을 캡쳐.<br>스키마 이름과 일치시키기 위해 Debezium은 고정된 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 스키마의 전체 이름 문자열과 일치. 스키마 이름에 존재할 수 있는 하위 문자열과 일치하지 않음(?).<br>구성에 이 속성을 포함하는 경우 Schema.include.list 속성을 설정하지 마세요.|
|table.include.list|기본값 없음|선택적. Debezium에서 캡처하려는 테이블의 정규화된 테이블 식별자와 일치하며 쉼표로 구분된 정규식 목록. 기본적으로 커넥터는 지정된 스키마에 대한 모든 비시스템 테이블을 캡처. 이 속성이 설정되면 커넥터는 지정된 테이블의 변경 사항만 캡처. 각 식별자는 `schemaName.tableName`(`dbo.customers`) 형식.<br>테이블 이름과 일치시키기 위해 Debezium은 고정된 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 테이블의 전체 이름 문자열과 일치. 테이블 이름에 존재할 수 있는 하위 문자열과 일치하지 않습니다(?).<br>구성에 이 속성을 포함하는 경우 table.exclude.list 속성도 설정하지 마세요.|
|table.exclude.list|기본값 없음|선택적. 캡처에서 제외하려는 테이블의 정규화된 테이블 식별자와 일치하며 쉼표로 구분된 정규식 목록. Debezium은 table.exclude.list에 포함되지 않은 모든 테이블을 캡처. 각 식별자는 `SchemaName.tableName`(dbo.customers) 형식.<br>테이블 이름과 일치시키기 위해 Debezium은 고정된 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 테이블의 전체 이름 문자열과 일치. 테이블 이름에 존재할 수 있는 하위 문자열과 일치하지 않습니다(?).<br>구성에 이 속성을 포함하는 경우 table.include.list 속성도 설정하지 마세요.|
|column.include.list|빈 문자열|선택적. 변경 이벤트 메시지 값에 포함되어야 하는 열의 정규화된 이름과 일치하며 쉼표로 구분된 정규식 목록. 열의 정규화된 이름은 `SchemaName.tableName.columnName` 형식. 기본 키 열은 값에 포함되지 않더라도 항상 이벤트 키를 포함됨.<br>열 이름과 일치시키기 위해 Debezium은 고정 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 열의 전체 이름 문자열과 일치. 열 이름에 존재할 수 있는 하위 문자열과 일치하지 않습니다.(?)<br>구성에 이 속성을 포함하는 경우 column.exclude.list 속성도 설정하지 마세요.|
|column.exclude.list|빈 문자열|선택적. 변경 이벤트 메시지 값에서 제외해야 하는 열의 정규화된 이름과 일치하며 쉼표로 구분된 정규식 목록. 열의 정규화된 이름은 `SchemaName.tableName.columnName` 형식. 기본 키 열은 값에서 제외되는 경우에도 항상 이벤트의 키를 포함.<br>열 이름과 일치시키기 위해 Debezium은 고정 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 열의 전체 이름 문자열과 일치. 열 이름에 존재할 수 있는 하위 문자열과 일치하지 않습니다.(?)<br>구성에 이 속성을 포함하는 경우 column.include.list 속성도 설정하지 마세요.|
|skip.messages.without.change|false|포함된 열에 변경 사항이 없을 때 게시 메시지를 건너뛸지 여부를 지정. 이는 column.include.list 또는 column.exclude.list 속성에 변경 사항이 없는 경우 기본적으로 메시지를 필터링.|
|column.mask.hash.<br>hashAlgorithm.with.salt.salt;<br>column.mask.hash.<br>v2.hashAlgorithm.with.salt.salt|해당사항 없음|선택적. 문자 기반 열의 정규화된 이름과 일치하며 쉼표로 구분된 정규식 목록. 열의 정규화된 이름은 `<schemaName>.<tableName>.<columnName>` 형식입니다.<br>열 이름과 일치시키기 위해 Debezium은 _anchored 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 열의 전체 이름 문자열과 일치. 표현식이 열 이름에 있을 수 있는 하위 문자열과 일치하지 않습니다.(?) 결과 변경 이벤트 레코드에서 지정된 열의 값은 가명으로 대체.<br>가명은 지정된 hashAlgorithm과 솔트를 적용한 결과로 생성된 해시 값으로 구성. 사용되는 해시 함수에 따라 참조 무결성이 유지되고 열 값은 가명으로 대체. 지원되는 해시 함수는 Java 암호화 아키텍처 표준 알고리즘 이름 문서의 `MessageDigest` 섹션에서 설명함.<br><br>다음 예에서 CzQMA0cB5K는 무작위로 선택된 솔트.<br><br>`column.mask.hash.SHA-256.with.salt.CzQMA0cB5K = Inventory.orders.customerName, Inventory.shipment.customerName`<br><br>필요한 경우 가명은 열 길이에 맞게 자동으로 단축됨. 커넥터 구성에는 다양한 해시 알고리즘과 솔트를 지정하는 여러 속성을 포함.<br>사용된 해시 알고리즘, 선택한 솔트 및 실제 데이터 세트에 따라 결과 데이터 세트가 완전히 마스킹되지 않을 수 있음.<br>값이 다른 장소나 시스템에서 해시되는 경우 충실도를 보장하려면 해싱 전략 버전 2를 사용해야 함.|
|time.precision.mode|adaptive|시간, 날짜 및 타임스탬프는 다음을 포함하여 다양한 종류의 정밀도로 표시될 수 있음. `adaptive`(default)은 데이터베이스 열 유형에 따라 밀리초, 마이크로초 또는 나노초 정밀도 값을 사용하여 데이터베이스에서 정확하게 시간 및 타임스탬프 값을 캡처. 혹은 데이터베이스와 연결된 열의 정밀도에 관계없이 밀리초 정밀도를 사용하는 Kafka Connect의 시간, 날짜 및 타임스탬프에 대한 기본 제공 표현을 사용하여 항상 시간 및 타임스탬프 값을 나타냄. 자세한 내용은 [시간 타입](#시간-타입)을 참조.|
|decimal.handling.mode|precise|커넥터가 DECIMAL 및 NUMERIC 열의 값을 처리하는 방법을 지정.<br>정밀(기본값)은 변경 이벤트에 바이너리 형식으로 표시되는 java.math.BigDecimal 값을 사용하여 이를 정확하게 나타냄.<br>double은 double 값을 사용하여 이를 표현하므로 정밀도가 떨어질 수 있지만 사용하기가 더 쉬움.<br>string은 값을 형식화된 문자열로 인코딩합니다. 이는 사용하기 쉽지만 실제 유형에 대한 의미 정보는 손실됨.|
|include.schema.changes|true|커넥터가 데이터베이스 서버 ID와 동일한 이름을 가진 Kafka Topic에 데이터베이스 스키마의 변경 사항을 게시해야 하는지 여부를 지정하는 부울 값. 각 스키마 변경 사항은 데이터베이스 이름이 포함된 키와 스키마 업데이트를 설명하는 JSON 구조인 값으로 기록됨. 이는 커넥터가 내부적으로 데이터베이스 스키마 기록을 기록하는 방식과 무관함. 기본값은 true.|
|tombstones.on.delete|true|삭제 이벤트 다음에 삭제 표시 이벤트가 뒤따르는지 여부를 제어.<br><br>true - 삭제 작업은 삭제 이벤트와 후속 삭제 표시 이벤트로 표시<br><br>false - 삭제 이벤트만 발생.<br><br>소스 레코드가 삭제된 후 Tombstone 이벤트를 내보내면(기본 동작) Kafka는 Topic에 대해 로그 압축이 활성화된 경우 삭제된 행의 키와 관련된 모든 이벤트를 완전히 삭제할 수 있음.|
|column.truncate.to.`length`.chars|해당사항 없음|선택적. 문자 기반 열의 정규화된 이름과 일치하며 쉼표로 구분된 정규식 목록. 속성 이름의 길이로 지정된 문자 수를 초과하는 경우 열 집합의 데이터를 자르려면 이 속성을 설정. 길이를 양의 정수 값으로 설정하십시오(예: column.truncate.to.`20`.chars).<br>열의 정규화된 이름은 `<schemaName>.<tableName>.<columnName>` 형식을 따름. 열 이름과 일치시키기 위해 Debezium은 고정 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 열의 전체 이름 문자열과 일치. 표현식이 열 이름에 있을 수 있는 하위 문자열과 일치하지 않습니다.(?)<br>단일 구성에서 길이가 다른 여러 속성을 지정할 수 있습니다.|
|column.mask.with.`length`.chars|해당 사항 없음.<br>열의 정규화된 이름은<br> `schemaName.tableName.columnName`<br>형식|선택적. 문자 기반 열의 정규화된 이름과 일치하며 쉼표로 구분된 정규식 목록. 예를 들어 민감한 데이터가 포함된 경우 커넥터가 열 집합의 값을 마스킹하도록 하려면 이 속성을 설정하십시오. 지정된 열의 데이터를 속성 이름의 길이로 지정된 별표(*) 문자 수로 바꾸려면 길이를 양의 정수로 설정. 지정된 열의 데이터를 빈 문자열로 바꾸려면 길이를 0(영)으로 설정합니다.<br>열의 정규화된 이름은 `schemaName.tableName.columnName` 형식을 따름. 열 이름과 일치시키기 위해 Debezium은 고정 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 열의 전체 이름 문자열과 일치. 표현식이 열 이름에 있을 수 있는 하위 문자열과 일치하지 않습니다.<br>단일 구성에서 길이가 다른 여러 속성을 지정할 수 있음.|
|column.propagate.source.type|해당사항 없음|선택적. 커넥터가 열 메타데이터를 나타내는 추가 매개변수를 내보내도록 하려는 열의 정규화된 이름과 일치하며 쉼표로 구분된 정규식 목록. 이 속성이 설정되면 커넥터는 이벤트 레코드의 스키마에 다음 필드를 추가.<br><br>* __debezium.source.column.type<br>* __debezium.source.column.length<br>*__debezium.source.column.scale<br><br>이러한 매개변수는 각각 열의 원래 유형 이름과 길이(가변 너비 유형의 경우)를 전파. 이 추가 데이터를 내보내도록 커넥터를 활성화하면 싱크 데이터베이스에서 특정 숫자 또는 문자 기반 열의 크기를 적절하게 조정하는 데 도움이 될 수 있음.<br><br>열의 정규화된 이름은 `schemaName.tableName.columnName` 형식을 따름. 열 이름과 일치시키기 위해 Debezium은 고정 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 열의 전체 이름 문자열과 일치. 표현식이 열 이름에 있을 수 있는 하위 문자열과 일치하지 않음.|
|datatype.propagate.source.type|해당사항 없음|선택적. 데이터베이스의 열에 대해 정의된 데이터 유형의 완전한 이름을 지정하며 쉼표로 구분된 정규식 목록. 이 속성이 설정되면 데이터 유형이 일치하는 열에 대해 커넥터는 스키마에 다음 추가 필드를 포함하는 이벤트 레코드를 내보냄.<br><br>* __debezium.source.column.type<br>* __debezium.source.column.length<br>* __debezium.source.column.scale<br><br>이러한 매개변수는 각각 열의 원래 유형 이름과 길이(가변 너비 유형의 경우)를 전파. 이 추가 데이터를 내보내도록 커넥터를 활성화하면 싱크 데이터베이스에서 특정 숫자 또는 문자 기반 열의 크기를 적절하게 조정하는 데 도움이 될 수 있음.<br><br>열의 정규화된 이름은 `schemaName.tableName.typeName` 형식을 따름. 데이터 유형의 이름을 일치시키기 위해 Debezium은 고정된 정규식으로 지정한 정규식을 적용합니다. 즉, 지정된 표현식은 데이터 유형의 전체 이름 문자열과 일치됩니다. 표현식이 유형 이름에 존재할 수 있는 하위 문자열과 일치하지 않습니다.<br><br>SQL Server 관련 데이터 형식 이름 목록은 [SQL Server 데이터 형식 매핑](#데이터-타입-매핑)을 참조하세요.|
|message.key.columns|해당사항 없음|커넥터가 지정된 테이블의 Kafka Topic에 게시하는 변경 이벤트 레코드에 대한 사용자 정의 메시지 키를 형성하는 데 사용하는 열을 지정하는 표현식 목록.<br><br>기본적으로 Debezium은 테이블의 기본 키 열을 내보내는 레코드의 메시지 키로 사용. 기본값 대신 또는 기본 키가 없는 테이블에 대한 키를 지정하기 위해 하나 이상의 열을 기반으로 사용자 지정 메시지 키를 구성할 수 있음.<br><br>테이블에 대한 사용자 정의 메시지 키를 설정하려면 테이블을 나열한 다음 메시지 키로 사용할 열을 나열하십시오. 각 목록 항목은 다음 형식을 사용.<br><br><완전한 자격을 갖춘_테이블 이름>:<keyColumn>,<keyColumn><br><br>여러 열 이름을 기반으로 테이블 키를 만들려면 열 이름 사이에 쉼표를 삽입하세요.<br><br>정규화된 각 테이블 이름은 다음 형식의 정규식.<br><br><스키마 이름>.<테이블 이름><br><br>속성에는 여러 테이블에 대한 항목이 포함될 수 있음. 목록에서 테이블 항목을 구분하려면 세미콜론을 사용.<br><br>다음 예에서는 Inventory.customers 및 buy.orders 테이블에 대한 메시지 키를 설정.<br><br>Inventory.customers:pk1,pk2;<br>(.*).purchaseorders:pk3,pk4<br><br>Inventory.customer 테이블의 경우 pk1 및 pk2 열이 메시지 키로 지정됨. 모든 스키마의 buyorders 테이블에 대해 pk3 및 pk4 서버 열은 메시지 키임.<br><br>사용자 정의 메시지 키를 생성하는 데 사용하는 열 수에는 제한이 없음. 그러나 고유 키를 지정하는 데 필요한 최소 개수를 사용하는 것이 가장 좋음.|
|binary.handling.mode|bytes|다음을 포함하여 변경 이벤트에서 이진(binary, varbinary) 열이 표시되어야 하는 방법을 지정.<br>`bytes`는 이진 데이터를 바이트 배열로 표시(기본값).<br>`base64`는 이진 데이터를 base64로 인코딩된 문자열로 표시.<br>`base64-url-safe`는 이진 데이터를 base64-url-save-encoded 문자열로 표시<br>`hex`는 바이너리 데이터를 16진수 인코딩(base16) 문자열로 표시.|
|schema.name.adjustment.mode|none|커넥터에서 사용하는 메시지 변환기와의 호환성을 위해 스키마 이름을 조정하는 방법을 지정합니다.<br><br>가능한 설정:<br>* none : 조정을 적용하지 않음.<br>* avro : Avro 유형 이름에 사용할 수 없는 문자를 밑줄로 변경.<br>* avro_unicode : Avro 유형 이름에 사용할 수 없는 밑줄이나 문자를 _uxxxx와 같은 해당 유니코드로 변경.<br>참고: _는 Java의 백슬래시와 같은 이스케이프 시퀀스|
|field.name.adjustment.mode|없음|커넥터에서 사용하는 메시지 변환기와의 호환성을 위해 필드 이름을 조정하는 방법을 지정.<br><br>가능한 설정:<br>* none : 조정을 적용하지 않음.<br>* avro : Avro 유형 이름에 사용할 수 없는 문자를 밑줄로 변경<br>* avro_unicode : Avro 유형 이름에 사용할 수 없는 밑줄이나 문자를 _uxxxx와 같은 해당 유니코드로 변경.<br>참고: _는 Java의 백슬래시와 같은 이스케이프 시퀀스<br><br>자세한 내용은 [Avro 이름 지정](https://debezium.io/documentation/reference/stable/configuration/avro.html#avro-naming)을 참조하세요.|

#### 고급 SQL Server 커넥터 구성 속성

다음 고급 구성 속성에는 대부분의 상황에서 작동하는 좋은 기본값이 있으므로 커넥터 구성에서 지정할 필요가 거의 없음.  
혹시 필요하면 [웹사이트 참조](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-advanced-connector-configuration-properties)

|속성|기본 값|설명|
|-|-|-|
|converters|기본값 없음|커넥터가 사용할 수 있는 [사용자 정의 변환기](https://debezium.io/documentation/reference/stable/development/converters.html#custom-converters) 인스턴스의 기호 이름을 쉼표로 구분한 목록을 열거함. 예를 들어,<br>Isbn<br>커넥터가 사용자 정의 변환기를 사용할 수 있도록 하려면 변환기 속성을 설정해야 함.<br>커넥터에 대해 구성하는 각 변환기에 대해 변환기 인터페이스를 구현하는 클래스의 정규화된 이름을 지정하는 `.type` 속성도 추가. `.type` 속성은 다음 형식을 사용.<br><*converterSymbolicName*>.type<br>예를 들어,<br>`isbn.type: io.debezium.test.IsbnConverter`<br>구성된 변환기의 동작을 추가로 제어하려면 하나 이상의 구성 매개변수를 추가하여 변환기에 값을 전달할 수 있음. 추가 구성 매개변수를 변환기와 연관시키려면 매개변수 이름 앞에 변환기의 기호 이름을 붙이십시오. 예를 들어,<br>`isbn.schema.name: io.debezium.sqlserver.type.Isbn`|
|snapshot.mode|*initial*|캡처된 테이블의 구조 및 선택적으로 데이터의 초기 스냅샷을 찍는 모드. 스냅샷이 완료되면 커넥터는 다시 데이터베이스의 redo 로그에서 변경 이벤트를 계속해서 읽음. 다음 값이 지원됨.<br><br>* `initial` : 캡처된 테이블의 구조 및 데이터에 대한 스냅샷을 찍음. 캡처된 테이블의 데이터에 대한 완전한 형태로 항목을 채워야 하는 경우 유용.<br>* `initial_only` : initial와 같은 구조 및 데이터의 스냅샷을 찍지만 대신 스냅샷이 완료된 후 변경사항의 스트리밍으로 전환하지 않음.<br>* `schema_only` : 캡처된 테이블의 구조에 대해서만 스냅샷을 찍음. 지금부터 발생하는 변경 사항만 topic에 전파해야 하는 경우 유용.|
|snapshot.include.collection.list|`table.include.list`에<br>지정된 모든 테이블|스냅샷에 포함할 테이블의 정규화된 이름(`<dbName>.<schemaName>.<tableName>`)과 일치하는 선택적인 쉼표로 구분된 정규식 목록. 지정된 항목의 이름은 커넥터의 `table.include.list` 속성에 지정되어야 함. 이 속성은 커넥터의 `snapshot.mode` 속성이 `never` 이외의 값으로 설정된 경우에만 적용됨. 이 속성은 증분 스냅샷의 동작에 영향을 주지 않음. 테이블 이름과 일치시키기 위해 Debezium은 고정된 정규식으로 지정한 정규식을 적용. 즉, 지정된 표현식은 테이블의 전체 이름 문자열과 일치. 테이블 이름에 존재할 수 있는 하위 문자열과 일치하지 않음.|
|snapshot.isolation.mode|*repeatable_read*|사용되는 트랜잭션 격리 수준과 캡처용으로 지정된 테이블을 커넥터가 잠그는 기간을 제어하는 모드. 다음 값이 지원됩니다.<br><br>* read_uncommitted<br>* read_committed<br>* repeatable_read<br>* snapshot<br>* exclusive (배타적 모드는 repeatable_read 격리 수준을 사용하지만 읽을 모든 테이블에 대해 배타적 잠금을 사용.)<br><br>snapshot, read_committed 및 read_uncommitted 모드는 초기 스냅샷 중에 다른 트랜잭션이 테이블 행을 업데이트하는 것을 방지하지 않음. Exclusive 및 Repeatable_read 모드는 동시 업데이트를 방지함.<br>모드 선택은 데이터 일관성에도 영향을 미침. 배타적 및 스냅샷 모드만 전체 일관성을 보장. 즉, 초기 스냅샷 및 스트리밍 로그는 선형적인(이어지는) 기록을 구성함. 예를 들어, Repeatable_read 및 read_committed 모드의 경우 추가된 레코드가 초기 스냅샷에 한 번, 스트리밍 단계에 한 번, 두 번 나타날 수 있음. 그럼에도 불구하고 해당 일관성 수준은 데이터 미러링에 적합해야 함. read_uncommitted의 경우 데이터 일관성이 전혀 보장되지 않음(일부 데이터가 손실되거나 손상될 수 있음).|
|event.processing.failure.handling.mode|fail|이벤트 처리 중 커넥터가 예외에 반응하는 방법을 지정.<br> * `fail` :  Exception(문제가 있는 이벤트의 오프셋을 나타냄)이 전파되어 커넥터를 중지함.<br> * `warn` : 문제가 있는 이벤트를 건너뛰고 문제가 있는 이벤트의 오프셋이 기록됨.<br> * `skip` : 문제가 있는 이벤트를 건너뜀.|
|poll.interval.ms|500|새 변경 이벤트가 나타날 때까지 커넥터가 각 반복 중에 기다려야 하는 시간(밀리초)을 지정하는 양의 정수 값. 기본값은 500밀리초, 즉 0.5초.|
|max.queue.size|8192|blocking 큐가 보유할 수 있는 최대 레코드 수를 지정하는 양의 정수 값. Debezium은 데이터베이스에서 스트리밍된 이벤트를 읽을 때 이벤트를 Kafka에 쓰기 전에 blocking 대기열에 배치함. blocking 대기열은 커넥터가 Kafka에 쓸 수 있는 것보다 더 빠르게 메시지를 수집하거나 Kafka를 사용할 수 없게 되는 경우 데이터베이스에서 변경 이벤트를 읽기 위한 역압을 제공. 커넥터가 주기적으로 오프셋을 기록할 때 대기열에 보관된 이벤트는 무시. 항상 `max.queue.siz`e 값을 `max.batch.size` 값보다 크게 설정해야 함.|
|max.queue.size.in.bytes|0|blocking 큐의 최대 볼륨을 바이트 단위로 지정하는 긴 정수 값. 기본적으로 blocking 대기열에는 볼륨 제한이 지정되지 않음. 대기열이 consume할 수 있는 바이트 수를 지정하려면 이 속성을 양의 큰 값으로 설정.<br>`max.queue.size`도 설정된 경우 대기열 크기가 두 속성 중 하나에 지정된 제한에 도달하면 대기열 쓰기를 차단함. 예를 들어 `max.queue.size=1000` 및 `max.queue.size.in.bytes=5000`을 설정한 경우 대기열에 1000개의 레코드가 포함되거나 대기열의 레코드 볼륨이 5000바이트에 도달해서 초과되면 대기열에 대한 쓰기를 차단함.|
|max.batch.size|2048|커넥터의 각 반복 중에 처리되어야 하는 각 이벤트 배치의 최대 크기를 지정하는 양의 정수 값.|
|heartbeat.interval.ms|0|heartbeat 메시지를 전송하는 빈도를 제어. 이 특성에는 커넥터가 heartbeat topic에 메시지를 보내는 빈도를 정의하는 간격(밀리초)을 포함. 이 속성을 사용하여 커넥터가 데이터베이스로부터 변경 이벤트를 계속 수신하고 있는지 확인할 수 있음. 또한 캡처되지 않은 테이블의 레코드만 장기간 변경되는 경우 하트비트 메시지를 활용해야 함. 이러한 상황에서 커넥터는 데이터베이스에서 로그 읽기를 진행하지만 Kafka에 변경 메시지를 내보내지 않음. 이는 오프셋 업데이트가 Kafka에 커밋되지 않음을 의미함. 이로 인해 커넥터가 다시 시작된 후 더 많은 변경 이벤트가 다시 전송될 수 있음. 하트비트 메시지를 전혀 보내지 않으려면 이 매개변수를 0으로 설정.
기본적으로 비활성화되어 있습니다.|
|snapshot.delay.ms|기본값 없음|커넥터가 시작된 후 스냅샷을 찍기 전에 기다려야 하는 간격(밀리초).<br>클러스터에서 여러 커넥터를 시작할 때 스냅샷 중단을 방지하는 데 사용할 수 있으며 이로 인해 커넥터 균형이 재조정될 수 있음.|
|snapshot.fetch.size|2000|스냅샷을 생성하는 동안 각 테이블에서 한 번에 읽어야 하는 최대 행 수를 지정. 커넥터는 이 크기의 중복 배치로 테이블 내용을 읽음. 기본값은 2000.|
|query.fetch.size|기본값 없음|지정된 쿼리로 반복해서 가져올 각 데이터베이스의 행 수를 지정. 기본값은 JDBC 드라이버의 기본 가져오기 크기.|
|snapshot.lock.timeout.ms|10000|스냅샷을 수행할 때 테이블 잠금을 얻기 위해 기다리는 최대 시간(밀리초)을 지정하는 정수 값. 이 시간 간격 내에 테이블 잠금을 획득할 수 없으면 스냅샷을 실패함.(스냅샷 참조). `0`으로 설정하면 커넥터가 잠금을 얻을 수 없을 때 즉시 실패. 값 `-1`은 무한 대기.|
|snapshot.select.statement.overrides|기본값 없음|스냅샷에 포함할 테이블 행을 지정. 스냅샷에 테이블 행의 하위 집합만 포함하려면 이 속성을 사용. 이 속성은 스냅샷에만 영향을 줌. 커넥터가 로그에서 읽는 이벤트에는 적용되지 않음.이 속성에는 `<schemaName>.<tableName>` 형식의 정규화된 테이블 이름이 쉼표로 구분된 목록이 포함되어 있음. 예를 들어,<br><br>`"snapshot.select.statement.overrides": "inventory.products,customers.orders"`<br><br>목록의 각 테이블에 대해 스냅샷을 생성할 때 커넥터가 테이블에서 실행할 SELECT 문을 지정하는 추가 구성 속성을 추가함. 지정된 SELECT 문은 스냅샷에 포함할 테이블 행의 하위 집합을 결정. 이 SELECT 문 속성의 이름을 지정하려면 다음 형식을 사용함.<br><br>`snapshot.select.statement.overrides.<schemaName>.<tableName>`.<br>예를 들면 `snapshot.select.statement.overrides.customers.orders`<br><br>예:<br>일시 삭제 열인 delete_flag가 포함된 Customers.orders 테이블에서 일시 삭제되지 않은 레코드만 스냅샷에 포함시키려면 다음 속성을 추가.<br><br>`"snapshot.select.statement.overrides": "customer.orders"`,<br>`"snapshot.select.statement.overrides.customer.orders": "SELECT * FROM [customers].[orders] WHERE delete_flag = 0 ORDER BY id DESC"`<br><br>결과적으로 스냅샷에서 커넥터에는 `delete_flag = 0`인 레코드만 포함.|
|source.struct.version<br>(deprecated)|v2|CDC 이벤트의 소스 블록에 대한 스키마 버전. Debezium 0.10에는 몇 가지 중단 사항이 도입됨. 모든 커넥터에 걸쳐 노출된 구조를 통합하기 위해 소스 블록의 구조를 변경함. 이 옵션을 v1로 설정하면 이전 버전에서 사용된 구조를 생성할 수 있음. 이 설정은 권장되지 않으며 향후 Debezium 버전에서 제거될 예정.|
|provide.transaction.metadata|false|true로 설정되면 Debezium은 트랜잭션 경계가 있는 이벤트를 생성하고 트랜잭션 메타데이터로 데이터 이벤트 Envelop을 강화.|
|retriable.restart.connector.wait.ms|10000(10초)|재시도 가능한 오류가 발생한 후 커넥터를 다시 시작하기 전에 기다려야 하는 시간(밀리초).|
|skipped.operations|t|스트리밍 중에 건너뛸 작업 유형의 쉼표로 구분된 목록. 작업에는 `c`(삽입/생성), `u`(업데이트), `d`(삭제), `t`(자르기), `none`(작업을 건너뛰지 않음)을 포함. 기본적으로 자르기 작업은 건너뜀.(이 커넥터에서는 생성되지 않음).|
|signal.data.collection|기본값 없음|커넥터에 신호를 보내는 데 사용되는 데이터 컬렉션의 정규화된 이름.<br>컬렉션 이름을 지정하려면 다음 형식을 사용 : `<데이터베이스 이름>.<스키마 이름>.<테이블 이름>`|
|signal.enabled.channels|source|커넥터에 대해 활성화된 signal 채널 이름 목록. 기본적으로 다음 채널을 사용.<br>* source<br>* kafka<br>* file<br>* jmx : 선택적으로 [사용자 정의 신호 채널](https://debezium.io/documentation/reference/stable/configuration/signalling.html#debezium-signaling-enabling-custom-signaling-channel)을 구현할 수도 있음.|
|notification.enabled.channels|기본값 없음|커넥터에 대해 활성화된 알림 채널 이름 목록. 기본적으로 다음 채널을 사용.<br>* sink<br>* log<br>* jmx : 선택적으로 [사용자 정의 알림 채널](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#{link-notification}.adoc#debezium-notification-custom-channel)을 구현할 수도 있음.|
|incremental.snapshot.allow.<br>schema.changes|false|증분 스냅샷 중에 스키마 변경을 허용. 활성화되면 커넥터는 증분 스냅샷 중에 스키마 변경을 감지하고 현재 청크를 다시 선택하여 DDL 잠금을 방지.<br><br>기본 키에 대한 변경은 지원되지 않으며 증분 스냅샷 중에 수행되면 잘못된 결과가 발생할 수 있습니다. 또 다른 제한 사항은 스키마 변경이 열의 기본값에만 영향을 미치는 경우 트랜잭션 로그 스트림에서 DDL이 처리될 때까지 변경 사항이 감지되지 않는다는 것. 이는 스냅샷 이벤트 값에 영향을 미치지 않지만 스냅샷 이벤트의 스키마에는 오래된 기본값이 있을 수 있음.|
|incremental.snapshot.chunk.size|1024|증분 스냅샷 청크 중에 커넥터가 가져와서 메모리로 읽는 최대 행 수. 청크 크기를 늘리면 스냅샷이 더 큰 크기의 더 적은 수의 스냅샷 쿼리를 실행하므로 효율성이 더 높아짐. 그러나 청크 크기가 클수록 스냅샷 데이터를 버퍼링하는 데 더 많은 메모리가 필요. 특정 환경에 최상의 성능을 제공하는 값으로 청크 크기를 조정해야 함.|
|max.iteration.transactions|0|데이터베이스의 여러 테이블에서 변경 내용을 스트리밍할 때 메모리 공간을 줄이기 위해 반복마다 사용할 최대 트랜잭션 수를 지정. 0(기본값)으로 설정하면 커넥터는 현재 최대 LSN을 변경 사항을 가져올 범위로 사용. 0보다 큰 값으로 설정되면 커넥터는 이 설정에 지정된 n번째 LSN을 변경 사항을 가져올 범위로 사용.|
|incremental.snapshot.option.recompile|false|증분 스냅샷 중에 사용되는 모든 SELECT 문에 OPTION(RECOMPILE) 쿼리 옵션을 사용. 이는 발생할 수 있지만 쿼리 실행 빈도에 따라 원본 데이터베이스의 CPU 로드가 증가할 수 있는 매개 변수 스니핑 문제를 해결하는 데 도움이 될 수 있음.|
|topic.naming.strategy|io.debezium.schema.<br>SchemaTopicNamingStrategy|데이터 변경, 스키마 변경, 트랜잭션, heartbeat 이벤트 등에 대한 Topic 이름을 결정하는 데 사용해야 하는 `TopicNamingStrategy 클래스의 이름`은 기본적으로 `SchemaTopicNamingStrategy`로 설정됨.|
|topic.delimiter|`.`|주제 이름에 대한 구분 기호를 지정. 기본값은 `.`|
|topic.cache.size|10000|bounded concurrent 해시 맵에서 Topic 이름을 보유하는 데 사용되는 크기. 이 캐시는 특정 데이터 컬렉션에 해당하는 Topic 이름을 결정하는 데 도움이 됨.|
|topic.heartbeat.prefix|__debezium-heartbeat|커넥터가 heartbeat 메시지를 보내는 Topic의 이름을 제어. Topic 이름에는 다음 패턴이 있음.<br>`topic.heartbeat.prefix.topic.prefix`<br>예를 들어 topic 접두사가 `fulfillment`인 경우 기본 topic 이름은 `__debezium-heartbeat.fulfillment`.|
|topic.transaction|transaction|커넥터가 트랜잭션 메타데이터 메시지를 보내는 Topic의 이름을 제어. Topic 이름에는 다음 패턴이 있음.<br>`topic.prefix.topic.transaction`<br>예를 들어 주제 접두사가 `fulfillment`인 경우 기본 주제 이름은 `fulfillment.transaction`. 자세한 내용은 [트랜잭션 메타데이터](#트랜잭션-메타-데이터)를 참조.|
|snapshot.max.threads|1|초기 스냅샷을 수행할 때 커넥터가 사용하는 스레드 수를 지정. 병렬 초기 스냅샷을 활성화하려면 속성을 1보다 큰 값으로 설정. 병렬 초기 스냅샷에서 커넥터는 여러 테이블을 동시에 처리. 이 기능은 인큐베이팅 중.|
|custom.metric.tags|기본값 없음|사용자 정의 메트릭 태그는 일반 이름 끝에 추가되어야 하는 MBean 객체 이름을 사용자 정의하기 위해 키-값 쌍을 허용하며, 각 키는 MBean 객체 이름에 대한 태그를 나타내며 해당 값은 해당 태그(key)의 값이 됨. 예: k1=v1,k2=v2.|
|errors.max.retries|-1|실패하기 전 재시도 가능한 오류(예: 연결 오류)에 대한 최대 재시도 횟수.(-1 = 제한 없음, 0 = 비활성화됨, > 0 = 재시도 횟수).|

#### Debezium SQL Server 커넥터 데이터베이스 스키마 기록 구성 속성

Debezium은 커넥터가 schema history topic과 상호 작용하는 방식을 제어하는 일련의 `schema.history.internal.*` 속성을 제공

다음 표에서는 Debezium 커넥터를 구성하기 위한 `schema.history.internal` 속성을 설명

|속성|기본 값|설명|
|-|-|-|
|schema.history.internal.kafka.topic|기본값 없음||
|schema.history.internal.kafka.bootstrap.servers|기본값 없음||
|schema.history.internal.kafka.recovery.poll.interval.ms|100||
|schema.history.internal.kafka.query.timeout.ms|3000||
|schema.history.internal.kafka.create.timeout.ms|30000||
|schema.history.internal.kafka.recovery.attempts|100||
|schema.history.internal.skip.unparseable.ddl|false||
|schema.history.internal.store.only.captured.tables.ddl|false||
|schema.history.internal.store.only.captured.databases.ddl|false||


##### Producer 및 Consumer 클라이언트 구성을 위한 Pass-through 데이터베이스 스키마 기록 속성

Debezium은 Kafka Producer를 사용하여 데이터베이스 스키마 기록 항목에 스키마 변경 사항을 기록.  
마찬가지로 커넥터가 시작될 때 Kafka Consumer를 사용하여 데이터베이스 스키마 기록 항목을 읽음.  
`schema.history.internal.producer.*` 및 `schema.history.internal.consumer.*` 접두사로 시작하는 일련의 Pass-through 구성 속성에 값을 할당하여 Kafka Producer 및 Consumer 클라이언트에 대한 구성을 정의.  
Pass-through Producer 및 Consumer 데이터베이스 스키마 기록 속성은 다음 예에 표시된 것처럼 이러한 클라이언트가 Kafka 브로커와의 연결을 보호하는 방법과 같은 다양한 동작을 제어.

```properties
schema.history.internal.producer.security.protocol=SSL
schema.history.internal.producer.ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
schema.history.internal.producer.ssl.keystore.password=test1234
schema.history.internal.producer.ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
schema.history.internal.producer.ssl.truststore.password=test1234
schema.history.internal.producer.ssl.key.password=test1234

schema.history.internal.consumer.security.protocol=SSL
schema.history.internal.consumer.ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
schema.history.internal.consumer.ssl.keystore.password=test1234
schema.history.internal.consumer.ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
schema.history.internal.consumer.ssl.truststore.password=test1234
schema.history.internal.consumer.ssl.key.password=test1234
```

Debezium은 속성을 Kafka 클라이언트에 전달하기 전에 속성 이름에서 접두사(`schema.history.internal.producer`, `schema.history.internal.consumer`)를 제거.

[Kafka Producer 구성 속성](https://kafka.apache.org/documentation.html#producerconfigs) 및 [Kafka Consumer 구성 속성](https://kafka.apache.org/documentation.html#consumerconfigs) 에 대한 자세한 내용은 Kafka 설명서를 참조.

##### Debezium 커넥터 Kafka 신호 구성 속성

Debezium은 커넥터가 Kafka Signal Topic과 상호 작용하는 방식을 제어하는 일련의 signal.* 속성을 제공.

다음 표에서는 Kafka Signal 속성을 설명.

|속성|기본값|설명|
|-|-|-|
|signal.kafka.topic|<topic.prefix>-signal|커넥터가 임시 신호를 모니터링하는 Kafka Topic의 이름.<br><br> * 주의 : [자동 Topic 생성](https://debezium.io/documentation/reference/stable/configuration/topic-auto-create-config.html#topic-auto-create-config)이 비활성화된 경우 필요한 신호 Topic을 수동으로 생성해야 함. 신호 순서를 유지하려면 신호 Topic이 필요. 신호 Topic에는 단일 파티션이 있어야 함.|
|signal.kafka.groupId|kafka-signal|Kafka Consumer가 사용하는 그룹 ID의 이름.|
|signal.kafka.bootstrap.servers|기본값 없음|커넥터가 Kafka 클러스터에 대한 초기 연결을 설정하는 데 사용하는 호스트/포트(host:port) 쌍 목록. 각 쌍은 Debezium Kafka Connect 프로세스에서 사용되는 Kafka 클러스터를 참조.|
|signal.kafka.poll.timeout.ms|100|커넥터가 신호를 polling할 때 기다리는 최대 시간(밀리초)을 지정하는 정수 값.|

##### Debezium 커넥터 통과 신호 Kafka 소비자 클라이언트 구성 속성

Debezium 커넥터는 Kafka Consumer 신호의 Pass-through 구성을 제공. Pass-through 신호 속성은 `signal.consumer.*` 접두사로 시작됩니다.  
예를 들어 커넥터는 `signal.consumer.security.protocol=SSL`과 같은 속성을 Kafka Consumer에게 전달.  
Debezium은 속성을 Kafka 신호 Consumer에게 전달하기 전에 속성에서 접두사를 제거.

##### Debezium 커넥터 싱크 알림 구성 속성

다음 표에서는 `notification`속성에 대해 설명.

|속성|기본값|설명|
|-|-|-|
|notification.sink.topic.name|기본값 없음|Debezium으로부터 알림을 받는 topic의 이름.<br>이 속성은 활성화된 알림 채널 중 하나로 `sink`를 포함하도록 `notification.enabled.channels` 속성을 구성할 때 필요.|

##### Debezium SQL Server 커넥터 Pass-through 데이터베이스 드라이버 구성 속성

Debezium 커넥터는 데이터베이스 드라이버의 Pass-through 구성을 제공.  
Pass-through 데이터베이스 속성은 접두사 `driver.*`로 시작함. 예를 들어 커넥터는 `driver.foobar=false`와 같은 속성을 JDBC URL에 전달.

[데이터베이스 스키마 기록 클라이언트에 대한 Pass-through 속성](#producer-및-consumer-클라이언트-구성을-위한-pass-through-데이터베이스-스키마-기록-속성)의 경우와 마찬가지로 Debezium은 속성을 데이터베이스 드라이버에 전달하기 전에 속성에서 접두사를 제거.

---

### 데이터베이스 스키마 변경

SQL Server 테이블에 대해 변경 데이터 캡처가 활성화되고 테이블에 변경이 발생하면 이벤트 레코드가 서버의 캡처 테이블에 저장됨. 예를 들어 새 열을 추가하여 원본 테이블의 구조를 변경하면 해당 변경 사항이 변경 테이블에 동적으로 반영되지 않음. 캡처 테이블이 오래된 스키마를 계속 사용하는 한 Debezium 커넥터는 테이블에 대한 데이터 변경 이벤트를 올바르게 내보낼 수 없음. 커넥터가 변경 이벤트 처리를 재개할 수 있도록 캡처 테이블을 새로 고치려면 별도 추가 작업이 필요.

SQL Server에서 CDC가 구현되는 방식으로 인해 Debezium을 사용하여 캡처 테이블의 업데이트 가능.  
캡처 테이블을 새로 고치려면 높은 권한을 가진 SQL Server 데이터베이스 운영자가 되어야 함.  
Debezium 사용자는 SQL Server 데이터베이스 운영자와 작업을 조정하여 스키마 새로 고침을 완료하고 Kafka 항목으로 스트리밍을 복원해야 함.

스키마 변경 후 다음 방법 중 하나를 사용하여 캡처 테이블의 업데이트 가능

* [오프라인 스키마 업데이트](#오프라인-스키마-업데이트)에서는 캡처 테이블을 업데이트하기 전에 Debezium 커넥터를 중지해야 합니다.
* [온라인 스키마 업데이트](#온라인-스키마-업데이트)는 Debezium 커넥터가 실행되는 동안 캡처 테이블을 업데이트할 수 있습니다.

각 유형의 절차를 사용하는 데는 장점과 단점이 있음.

> [!WARNING]  
> 온라인 업데이트 방법을 사용하든 오프라인 업데이트 방법을 사용하든 동일한 원본 테이블에 후속 스키마 업데이트를 적용하기 전에  
> 전체 스키마 업데이트 프로세스를 완료해야 합니다.  
> 가장 좋은 방법은 모든 DDL을 단일 배치로 실행하여 프로시저가 한 번만 실행될 수 있도록 하는 것입니다.  

> [!NOTE]  
> CDC가 활성화된 원본 테이블에서는 일부 스키마 변경 사항이 지원되지 않음.  
> 예를 들어 테이블에서 CDC가 활성화된 경우 SQL Server에서는 해당 열 중 하나의 이름을 바꾸거나  
> 열 유형을 변경한 경우 테이블의 스키마 변경을 허용하지 않음.  

> [!NOTE]  
> 원본 테이블의 열을 `NULL`에서 `NOT NULL`로 또는 그 반대로 변경한 후에는  
> 새 캡처 인스턴스를 생성할 때까지 SQL Server 커넥터가 변경된 정보를 올바르게 캡처할 수 없음.  
>  
> 열 지정을 변경한 후 새 캡처 테이블을 작성하지 않으면  
> 커넥터가 생성하는 변경 이벤트 레코드가 해당 열이 선택사항인지 여부를 올바르게 나타내지 않음.  
> 즉, 이전에 선택 사항(또는 `NULL`)으로 정의된 열은 현재는 `NOT NULL`으로 정의되어 있음에도 불구하고 계속해서 선택 사항으로 정의됨.  
> 마찬가지로, 필수(`NOT NULL`)로 정의된 열은 이제 `NULL`로 정의되어 있어도 해당 지정을 유지.

> [!NOTE]  
> 함수를 사용하여 테이블 이름을 바꾼 후에 sp_rename는 커넥터가 다시 시작될 때까지 이전 원본 테이블 이름으로 변경 사항을 계속 내보냄.  
> 커넥터를 다시 시작하면 새 원본 테이블 이름 아래에 변경 사항을 표시.

#### <u>오프라인 스키마 업데이트</u>

오프라인 스키마 업데이트는 캡처 테이블을 업데이트하는 가장 안전한 방법을 제공합니다. 그러나 고가용성이 필요한 애플리케이션에서는 오프라인 업데이트를 사용하지 못할 수도 있습니다.

전제조건

* CDC가 활성화된 SQL Server 테이블의 스키마에 업데이트가 커밋됨.
* 높은 권한을 가진 SQL Server 데이터베이스 운영자.

절차

1. 데이터베이스를 업데이트하는 애플리케이션을 일시중단.
2. Debezium 커넥터가 스트리밍되지 않은 모든 변경 이벤트 레코드를 스트리밍할 때까지 기다림.
3. Debezium 커넥터를 중지.
4. 원본 테이블 스키마에 모든 변경 사항을 적용.
5. @capture_instance 매개 변수에 고유한 값이 있는 sys.sp_cdc_enable_table 프로시저를 사용하여  
   업데이트 원본 테이블에 대한 새 캡처 테이블을 생성.
6. 1단계에서 일시 중단한 애플리케이션을 재개.
7. Debezium 커넥터를 시작.
8. Debezium 커넥터가 새 캡처 테이블에서 스트리밍을 시작한 후  
   매개변수 @capture_instance가 이전 캡처 인스턴스 이름으로 설정된 sys.sp_cdc_disable_table 저장 프로시저를 실행하여 이전 캡처 테이블을 삭제.

#### <u>온라인 스키마 업데이트</u>

온라인 스키마 업데이트를 완료하는 절차는 오프라인 스키마 업데이트를 실행하는 절차보다 간단하며 애플리케이션 및 데이터 처리에 다운타임이 필요 없이 완료 가능.  
그러나 온라인 스키마 업데이트를 사용하면 원본 데이터베이스에서 스키마를 업데이트한 후 새 캡처 인스턴스를 생성하기 전에 잠재적인 처리 공백이 발생할 수 있음.  
해당 간격 동안 변경 이벤트는 변경 테이블의 이전 인스턴스에 의해 계속 캡처되며 이전 테이블에 저장된 변경 데이터는 이전 스키마의 구조를 유지.  
따라서 원본 테이블에 새 열을 추가한 경우 새 캡처 테이블이 준비되기 전에 생성된 변경 이벤트에는 새 열에 대한 필드가 포함되지 않음.  
애플리케이션이 이러한 전환 기간(처리 공백)을 허용하지 않는 경우 오프라인 스키마 업데이트 절차를 사용하는 것으로 추천함.

전제조건

* CDC가 활성화된 SQL Server 테이블의 스키마에 업데이트가 커밋.
* 높은 권한을 가진 SQL Server 데이터베이스 운영자.

절차

1. 소스 테이블 스키마에 모든 변경 사항을 적용.
2. 매개변수 @capture_instance 에 대한 고유한 값을 사용하여 sys.sp_cdc_enable_table 저장 프로시저를 실행하여 업데이트 소스 테이블에 대한 새 캡처 테이블을 생성.
3. Debezium이 새 캡처 테이블에서 스트리밍을 시작하면 매개 변수 @capture_instance를 이전 캡처 인스턴스 이름으로 설정하여 sys.sp_cdc_disable_table저장 프로시저를 실행하여 이전 캡처 테이블 삭제 가능.

<u>예: 데이터베이스 스키마 변경 후 온라인 스키마 업데이트 실행</u>

온라인 스키마 업데이트를 보여주기 위해 SQL Server 기반 Debezium 튜토리얼을 배포

다음 예는 customers 테이블에 phone_number 컬럼이 추가됨.

1. 다음 쿼리를 실행하여 `customers` 테이블에 `phone_number` 컬럼을 추가함.

```sql
ALTER TABLE customers ADD phone_number VARCHAR(32);
```

2. `sys.sp_cdc_enable_table` 저장 프로시저를 실행하여 새 캡쳐 인스턴스 생성.

```sql
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'customers', @role_name = NULL, @supports_net_changes = 0, @capture_instance = 'dbo_customers_v2';
GO
```

3. 다음 쿼리를 실행하여 테이블 `customers`에 새 데이터를 추가.

```sql
INSERT INTO customers(first_name,last_name,email,phone_number) VALUES ('John','Doe','john.doe@example.com', '+1-555-123456');
GO
```

Kafka Connect 로그는 다음 메시지와 유사한 항목을 통해 구성 업데이트에 대해 보고합니다.

``` bash
connect_1    | 2019-01-17 10:11:14,924 INFO   ||  Multiple capture instances present for the same table: Capture instance "dbo_customers" [sourceTableId=testDB.dbo.customers, changeTableId=testDB.cdc.dbo_customers_CT, startLsn=00000024:00000d98:0036, changeTableObjectId=1525580473, stopLsn=00000025:00000ef8:0048] and Capture instance "dbo_customers_v2" [sourceTableId=testDB.dbo.customers, changeTableId=testDB.cdc.dbo_customers_v2_CT, startLsn=00000025:00000ef8:0048, changeTableObjectId=1749581271, stopLsn=NULL]   [io.debezium.connector.sqlserver.SqlServerStreamingChangeEventSource]
connect_1    | 2019-01-17 10:11:14,924 INFO   ||  Schema will be changed for ChangeTable [captureInstance=dbo_customers_v2, sourceTableId=testDB.dbo.customers, changeTableId=testDB.cdc.dbo_customers_v2_CT, startLsn=00000025:00000ef8:0048, changeTableObjectId=1749581271, stopLsn=NULL]   [io.debezium.connector.sqlserver.SqlServerStreamingChangeEventSource]
...
connect_1    | 2019-01-17 10:11:33,719 INFO   ||  Migrating schema to ChangeTable [captureInstance=dbo_customers_v2, sourceTableId=testDB.dbo.customers, changeTableId=testDB.cdc.dbo_customers_v2_CT, startLsn=00000025:00000ef8:0048, changeTableObjectId=1749581271, stopLsn=NULL]   [io.debezium.connector.sqlserver.SqlServerStreamingChangeEventSource]
```

`phone_number` 필드가 스키마에 추가되고 해당 값이 Kafka topic에 기록된 메시지에 표시됨.

```json
...
     {
        "type": "string",
        "optional": true,
        "field": "phone_number"
     }
...
    "after": {
      "id": 1005,
      "first_name": "John",
      "last_name": "Doe",
      "email": "john.doe@example.com",
      "phone_number": "+1-555-123456"
    },
```

4. 저장 프로시저 `sys.sp_cdc_disable_table`를 실행하여 이전 캡쳐 인스턴스를 삭제함.

```sql
EXEC sys.sp_cdc_disable_table @source_schema = 'dbo', @source_name = 'dbo_customers', @capture_instance = 'dbo_customers';
GO
```

### 모니터링

JMX를 통한 방법은 별도 Debizium 모니터링 설명서 참조.