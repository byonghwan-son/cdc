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

*초기 스냅샷이 모든 케이블의 스키마를 캡쳐했음.(**store.only.captured.tables.ddl : false**)*

1. 캡쳐할 테이블을 지정하려면 ```table.include.list```의 속성을 편집
2. 커넥터를 다시 시작
3. 새로 추가된 테이블에서 기존 데이터를 캡쳐하려면 [증분 스냅샷](https://debezium.io/documentation/reference/stable/connectors/sqlserver.html#sqlserver-incremental-snapshots)을 시작
