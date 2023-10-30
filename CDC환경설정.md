# CDC 환경 설정

## SQL 서버

* SQL서버 복제본에서는 CDC를 직접 지원하지 않음.

* 구성 옵션 Database.applicationIntent는 ReadOnly로 설정됨. 이는 SQL Server에 필요. Debezium이 이 구성 옵션을 감지하면 다음 작업을 수행하여 응답.
  * `snapshot.isolation.mode`를 읽기 전용 복제본에 대해 지원되는 유일한 트랜잭션 격리 모드인 `snapshot`으로 설정.
  * CDC 데이터의 최신 보기를 가져오는 데 필요한 스트리밍 쿼리 루프를 실행할 때마다 (읽기 전용) 트랜잭션을 커밋.

* SQL서버 설치 이후에 컴퓨터명을 변경할 경우 CDC설정할 때 에러가 발생한다.
  * The failure occurred when executing the command '[sys].[sp_cdc_add_job] @job_type = N'capture''.
  
```sql
    -- SQL 서버 설치할 때 컴퓨터명
    SELECT srvname AS OldName FROM master.dbo.sysservers

    -- 현재 컴퓨터 명
    SELECT SERVERPROPERTY('ServerName') AS NewName

    -- 다를 경우 아래의 쿼리로 변경해야 함.
    sp_dropserver '<이전 컴퓨터명>';  
    GO  
    sp_addserver '<현재 컴퓨터명>', local;  
    GO  
```

* CDC 설정

```sql
    -- 데이터 베이스에 대한 CDC 활성화
    USE <DB_NAME>
    GO
    EXEC sys.sp_cdc_enable_db
    GO

    -- 아래의 sys.sp_cdc_enable_table에 포함된 프로시저
    -- 혹시 오류가 발생했을 때 아래의 쿼리를 실행해서 세부 항목 확인 필요.
    -- EXEC sys.sp_cdc_add_job @job_type = N'capture';
    -- GO

    USE <DB_NAME>
    GO
    exec sys.sp_cdc_enable_table 
    @source_schema = N'dbo', 
    @source_name = N'Customer', ①
    @role_name = null, ②
    @filegroup_name = N'MyDB_CT', ③
    @supports_net_changes = 1 
    GO
```

① 캡쳐하려는 테이블의 이름  

② 소스 테이블의 캡처된 열에 대한 SELECT 권한을 부여하려는 사용자를 추가할 수 있는 역할을 지정.  
　　sysadmin 또는 db_owner 역할의 사용자는 지정된 변경 테이블에도 액세스할 수 있음.  
　　@role_name 값을 NULL로 설정하면 sysadmin 또는 db_owner의 멤버만 캡처된 정보에 대한 전체 액세스 권한을 가질 수 있음.

③ SQL Server가 캡처된 테이블에 대한 변경 테이블을 배치하는 파일 그룹을 지정.  
　　명명된 파일 그룹이 이미 존재.  
　　<u>원본 테이블에서 사용하는 것과 동일한 파일 그룹에 변경 테이블을 두지 않는 것이 가장 좋음</u>.

* CDC 해제

```sql
    EXEC sys.sp_cdc_disable_table
    @source_schema = N'dbo',
    @source_name = N'Customer',
    @capture_instance = 'all'
    GO

    EXEC sys.sp_cdc_disable_db
    GO
```

* CDC 활성화 정보 추출

```sql
    EXEC sys.sp_cdc_help_change_data_capture
    GO
```

KAFKA CONNECT에 등록할 debezium MS-SQL 커넥터 환경 설정 파일 (테스트 완료)

```json
{
    "name": "mydb_connector", ①
    "config": {
        "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector", ②
        "tasks.max": "1",
        "database.hostname": "192.168.253.132", ③
        "database.port": "1433", ④ 
        "database.user": "sa", ⑤
        "database.password": "gk", ⑥
        "database.names": "MyDB", ⑦
        "topic.prefix": "mydb", ⑧
        "table.include.list": "dbo.Customer", ⑨
        "schema.history.internal.kafka.bootstrap.servers": "localhost:9092", ⑩
        "schema.history.internal.kafka.topic": "schema-changes.mssql.mydb", ⑪
        "database.encrypt": "false", ⑫
        "database.ssl.truststore": "path/to/trust-store", ⑬
        "database.ssl.truststore.password": "password-for-trust-store" ⑭
    }
}
```

① Kafka Connect 서비스에 등록할 때 커넥터의 이름입니다.  
② 이 SQL Server 커넥터 클래스의 이름입니다.  
③ SQL Server 인스턴스의 주소입니다.  
④ SQL Server 인스턴스의 포트 번호입니다.  
⑤ SQL Server 사용자의 이름  
⑥ SQL Server 사용자의 비밀번호  
⑦ 변경 사항을 캡처할 데이터베이스의 이름입니다.  
⑧ 네임스페이스를 형성하고 커넥터가 쓰는 Kafka 항목의 모든 이름에 사용되는 SQL Server 인스턴스/클러스터의 항목 접두사, Kafka Connect 스키마 이름 및 Avro 변환기가 실행될 때 해당 Avro 스키마의 네임스페이스 사용.  
⑨ Debezium이 캡처해야 하는 변경 사항이 있는 모든 테이블의 목록입니다. 컴마로 구분.  
⑩ 이 커넥터가 데이터베이스 스키마 기록 항목에 DDL 문을 작성하고 복구하는 데 사용할 Kafka 브로커 목록입니다.  
⑪ 커넥터가 DDL 문을 작성하고 복구할 데이터베이스 스키마 기록 항목의 이름입니다. 이 항목은 내부용으로만 사용되며 소비자가 사용해서는 안 됩니다.  
⑫ 데이터베이스 암호화가 비활성화  
⑬ 서버의 서명자 인증서를 저장하는 SSL 신뢰 저장소의 경로. 데이터베이스 암호화가 비활성화되지 않은 경우(database.encrypt=false) 이 속성은 필수.  
⑭ SSL 신뢰 저장소 비밀번호. 데이터베이스 암호화가 비활성화되지 않은 경우(database.encrypt=false) 이 속성은 필수.  

---
> [!WARNING]  
> zoo_keeper, kafka, connect를 모두 실행해서 데이터를 로딩하고 있는 중에 스키마를 변경하고 데이터를 전송하면  
> 다음 부터 데이터 전송이 안됨.
