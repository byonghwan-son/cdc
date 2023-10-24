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

* snapshot.mode: "initial"
  * 구조와 값을 가져 온다.

1. 데이터 베이스에 대한 연결 설정
2. 캡쳐할 테이블을 결정. 기본적으로 시스템 테이블을 제외한 모든 테이블이 대상.
   1. table.include.list: "포함할 <테이블 명, 테이블 명, ...>"
   2. table.exclude.list: "제외할 <테이블 명, 테이블 명, ...>"
3. 스냅샷 도중 데이터 구조적 변경이 발생하지 않도록 테이블 락 적용.
   1. snapshot.isolation.mode 에 따라 잠금 수준이 정해짐.
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

* schema.history.internal.store.only.captured.tables.ddl : 스키마 정보를 캡처할 테이블을 지정하는 속성을 설정. (true/false)
* schema.history.internal.store.only.captured.databases.ddl : 스키마 변경 사항을 캡처할 논리적 데이터베이스를 지정하는 속성을 설정. (true/false)

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
         schema.history.internal.captured.tables.ddl : false
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

*초기 스냅샷이 모든 테이블의 스키마를 캡쳐했음.(**store.only.captured.tables.ddl : false**)*

1. 캡쳐할 테이블을 지정하려면 ```table.include.list```의 속성을 편집
2. 커넥터를 다시 시작
3. 새로 추가된 테이블에서 기존 데이터를 캡쳐하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작

*초기 스냅샷이 모든 테이블의 스키마를 캡쳐하지 못했음.(**store.only.captured.tables.ddl : true**)*

초기 스냅샷이 캡쳐하려는 테이블의 스키마를 저장하지 않은 경우

절차 1: 스키마 스냅샷에 이어진 증분 스냅샷

1. 커넥터를 중지
2. ```schema.history.internal.kafka.topic```속성에서 지정하는 내부 데이터베이스 Schema History Topic을 제거
3. Kafka Connect의 ```offset.storage.topic```에 구성된 오프셋을 지움. 오프셋 제거 방법은 [카프카커넥터](kafka_connect.md)를 참조
4. 다음 단계에 설명된 대로 커넥터 구성의 속성값을 설정
   1. ```snapshot.mode``` 속성값을 ```schema_only```로 변경
   2. 캡쳐하려는 테이블을 추가하려면 ```table.include.list```를 편집
5. 커넥터를 다시 시작
6. Debezium이 새 테이블과 기존 테이블의 스키마를 캡쳐할 때까지 기다림. 커넥터가 중지된 후 테이블에서 발생한 데이터 변경 사항은 캡쳐되지 않음(7번 항목에서 처리).
7. 데이터가 손실되지 않도록 하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작함.

절차 2 : 초기 스냅샷에 이어 선택적 증분 스냅샷

이 절차에서 커넥터는 데이터베이스의 전체 초기 스냅샷을 수행. 모든 초기 스냅샷과 마찬가지로 대규모 테이블이 많은 데이터베이스에서는 초기 스냅샷을 실행하는 데 시간이 많이 걸릴 수 있음. 스냅샷이 완료된 후 선택적으로 증분 스냅샷을 트리거하여 커넥터가 오프라인인 동안 발생하는 모든 변경 사항을 캡처할 수 있음.

1. 커넥터 중지
2. ```schema.history.internal.kafka.topic```속성에서 지정하는 내부 데이터베이스 Schema History Topic을 제거
3. Kafka Connect의 ```offset.storage.topic```에 구성된 오프셋을 지움. 오프셋 제거 방법은 [카프카커넥터](kafka_connect.md)를 참조
4. 캡쳐하려는 테이블을 추가하려면 ```table.include.list```를 편집.
5. 다음 단계에 설명된 대로 커넥터 구성의 속성 값을 설정
   1. 속성값 ```snapshot.mode```를 ```initial```로 설정
   2. (선택사항) ```schema.history.internal.store.only.captured.tables.ddl```을 ```false```로 설정
6. 커넥터를 다시 시작. 커넥터는 전체 데이터베이스 스냅샷을 생성. 스냅샷이 완료되면 커넥터가 스트리밍으로 전환.
7. (선택사항) 커넥터가 오프라인인 동안 변경된 데이터를 캡쳐하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작함.

#### 임시 스냅샷(adhoc snapshot)

기본적으로 커넥터는 처음 시작된 후에만 초기 스냅샷 작업을 실행함. 이 초기 스냅샷 이후에 정상적인 상황에서는 커넥터가 스냅샷 프로세스를 반복하지 않음. 향후 커넥터가 캡쳐하는 변경 데이터 이벤트는 스트리밍 프로세스를 통해서만 제공됨.

일부 상황에서는 커넥터가 가져온 초기 스냅샷 데이터가 오래되거나 손실되거나 불완전해질 수 있음. 테이블 데이터를 다시 캡쳐하는 메커니즘을 제공하기 위해 ***임시 스냅샷***을 수행하는 옵션이 있음.

전제 조건

* 커넥터 구성이 다른 테이블 세트를 캡쳐하도록 수정되었음.
* Kafka Topic이 삭제되었으므로 다시 빌드해야 함.
* 구성 오류 또는 기타 문제로 인해 데이터 손상이 발생함.

임시 스냅샷으로 이전에 스냅샷을 캡쳐한 테이블에 대에 스냅샷을 <u>다시 실행</u>할 수 있음. 임시 스냅샷에는 [신호 테이블](https://debezium.io/documentation/reference/stable/configuration/signalling.html#sending-signals-to-a-debezium-connector)을 사용해야 함. Debezium **Signal Table**에 신호 요청을 보내 임시 스냅샷을 시작함.

기존 테이블에 임시 스냅샷을 시작하면 해당 테이블을 위해 이미 존재하는 Topic에 콘텐츠를 추가. 혹시 Topic이 제거 된 상태에서 [Automatic Topic Creation](https://debezium.io/documentation/reference/stable/configuration/topic-auto-create-config.html#customizing-debezium-automatically-created-topics)이 활성화되어 있다면 Debezium은 자동으로 Topic을 생성함.

임시 스냅샷 Signal에는 스냅샷에 포함할 테이블을 명시. 스냅샷은 데이터베이스의 전체 내용을 캡쳐하거나 테이블의 하위 집합만 캡쳐할 수 있음. 테이블 내용의 하위 집합을 캡쳐할 수 있음.

Signal 테이블에 ```execute-snapshot``` 메시지를 보내 캡쳐할 테이블을 지정함. ```execute-snapshot``` Signal의 type을 ```incremental``` 혹은 ```blocking```으로 설정하고 스냅샷에 포함할 테이블의 이름을 작성함.

[임시 ```execute-snapshot``` signal 행의 필드 설명]
|Field|Default|Value|
|-|-|-|
|type|incremental<br>증분|실행하려는 스냅샷 유형을 지정<br>incremental, blocking|
|data-<br>collections|N/A<br>해당없음|스냅샷을 생성할 테이블의 정규화된 이름과 일치하는 정규식이 포함된 배열<br>signal.data.collection 구성 옵션과 동일|
|additional-<br>conditions|N/A<br>해당없음|스냅샷에 포함할 레코드(행)의 하위 집합을 결정하기 위해 커넥터가 평가하는 추가 조건 집합을 지정하는 선택적 배열<br>각 추가 조건은 임시 스냅샷이 캡쳐하는 데이터를 필터링하기 위한 기준을 지정하는 개체<br>각 추가 조건에 대해 다음 매개변수를 설정<br><br>*data-collection*<br>필터가 적용된 정규화된 이름. 각 테이블에 서로 다른 필터를 적용할 수 있음.<br><br>*filter*<br>스냅샷에 포함하기 위해 데이터베이스 레코드에 있어야 할 열 값을 지정<br>예) "color='blue'"<br><br>필터 매개변수에 할당하는 값은 blocking 스냅샷에 대한 ```snapshot.select.statement.overrides``` 속성을 설정할 때 SELECT 문의 WHERE 절에 지정할 수 있는 값과 동일한 유형입니다.|
|surrogate-<br>key|N/A<br>해당없음|스냅샷 프로세스 중에 커넥터가 테이블의 기본 키로 사용하는 열 이름을 지정하는 선택적 문자열|

<u>임시 증분(incremental) 스냅샷 트리거</u>

```execute-snapshot``` signal type을 Signal Table에 추가하여 임시 증분 스냅샷을 시작. 커넥터는 처리중인 메시지가 있다면 메시지를 처리하고 스냅샷 작업을 시작. 스냅샷 프로세스는 첫번째 및 마지막 키 값을 읽고 해당 값을 각 테이블의 시작과 끝 지점으로 사용. Debezium은 테이블의 항목 수와 구성된 청크 크기에 따라 테이블을 청크로 나누고 각 청크를 한 번에 하나씩 스냅샷 처리함.

<u>임시 블록(blocking) 스냅샷 트리거</u>

```execute-snapshot``` signal type을 Signal Table에 추가하여 임시 블록(Blocking) 스냅샷을 시작. 커넥터는 처리중인 메시지가 있다면 메시지를 처리하고 스냅샷 작업을 시작. 커넥터는 일시적으로 스트리밍을 중지한 다음 초그 스냅샷 중에 사용하는 것과 동일한 프로세스에 따라 지정된 테이블의 스냅샷을 시작함. 스냅샷이 완료되면 커넥터가 스트리밍을 재개함.

#### 증분 스냅샷(Incremental Snapshot)

Debezium에는 스냅샷 관리에 유연성을 제공하기 위해 증분 스냅샷이라는 스냅샷 메커니즘이 포함되어 있음. [증분 스냅샷은 Debezium 커넥터에 신호를 보내기]((https://debezium.io/documentation/reference/stable/configuration/signalling.html#sending-signals-to-a-debezium-connector)) 위해 Debezium 자체의 매커니즘을 사용.

Debezium은 증분 스냅샷에서 구성 가능한 일련의 청크로 각 테이블을 단계적으로 캡쳐. 스냅샷으로 캡처할 테이블과 [각 청크의 크기](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-property-incremental-snapshot-chunk-size)를 지정할 수 있음. 청크 크기는 데이터 베이스에서 각 가져오기 작업 중에 스냅샷이 수집하는 행 수를 결정함. 기본 청크 크기는 1024개의 행.

증분 스냅샷이 진행됨에 따라 워터마크(해시값)를 사용하여 진행 상황을 추적하고 캡쳐하는 각 테이블 행의 기록을 유지. 데이터 캡쳐에 대한 이러한 단계별 접근 방식은 다음과 같은 이점을 제공함.

* 스냅샷이 완료될 때까지 스트리밍을 연기하는 대신 스트리밍 데이터 캡쳐와 증분 스냅샷을 병렬로 실행할 수 있음. 커넥터는 스냅샷 프로세스 전반에 결쳐 변경 로그에서 거의 실시간 이벤트를 계속 캡쳐하며 두 작업 모두 다른 작업을 차단하지 않음.
* <u>증분 스냅샷의 진행이 중단된 경우 데이터 손실이 없이 재개할 수 있음.</u> 프로세스가 재개된 후에 처음부터 테이블을 다시 캡쳐하는 것이 아니라 중지된 지점에서 스냅샷이 시작됨.
* 언제든지 필요에 따라 증분 스냅샷을 실행하고 데이터베이스 업데이트에 적응하기 위해 필요에 따라 프로세스를 반복할 수 있음. 예를 들어 커넥터 구성을 수정하여 해당 ```table.include.list``` 속성에 테이블을 추가한 후 스냅샷을 다시 실행할 수 있음.

<u>증분 스냅샷 프로세스(작업 순서 요약)</u>

증분 스냅샷을 실행하면 기본 키를 기준으로 각 테이블을 정렬한 다음 [구성된 청크 크기](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-property-incremental-snapshot-chunk-size)에 따라 테이블을 청크로 분할함. 청크 단위로 작업한 다음 청크의 각 테이블 행을 캡쳐함. 스냅샷은 캡쳐하는 각 행에 대해 ```READ```(스냅샷) 이벤트를 생성. 해당 이벤트는 청크의 스냅샷이 시작될 당시의 행의 값을 나타냄.

스냅샷의 진행중에도 다른 프로세스가 계속해서 데이터베이스에 액세스하여 잠재적으로 테이블 레코드를 수정할 가능성이 있음. 이러한 변경 사항을 반영하기 위해 ```INSERT```, ```UPDATE```, ```DELETE``` 작업이 평소와 같이 트랜잭션 로그에 커밋됨. 마찬가지로 진행 중인 Debezium 스트리밍 프로세스도 이러한 변경 이벤트를 계속 감지해서 해당 변경 이벤트 레코드를 Kafka에 내보냄.

<u>Debezium이 동일한 기본 키를 가진 레코드 간의 충돌을 해결하는 방법</u>

순서 없이 도착하는 증분 스냅샷 이벤트가 올바른 논리적 순서로 처리되도록 하기 위해 충돌 해결을 위한 버퍼링 체계를 사용. 스냅샷 이벤트와 스트리밍 이벤트 간의 충돌이 해결된 후에만 Kafka에 이벤트 레코드를 전송.

<u>스냅샷 창(Snapshot Window)</u>

