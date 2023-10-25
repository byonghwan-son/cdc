# CDC 환경 설정

## SQL 서버

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
    EXEC sys.sp_cdc_enable_db
    GO

    -- 아래의 sys.sp_cdc_enable_table에 포함된 프로시저
    -- 혹시 오류가 발생했을 때 아래의 쿼리를 실행해서 세부 항목 확인 필요.
    -- EXEC sys.sp_cdc_add_job @job_type = N'capture';
    -- GO

    exec sys.sp_cdc_enable_table 
    @source_schema = N'dbo', 
    @source_name = N'Customer', 
    @role_name = null, 
    @supports_net_changes = 1 
    GO
```

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

KAFKA CONNECT에 등록할 debezium MS-SQL 커넥터 환경 설정 파일 (테스트 완료)

```json
{
    "name": "mydb_connector",
    "config": {
        "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
        "tasks.max": "1",
        "database.hostname": "192.168.253.132",
        "database.port": "1433",
        "database.user": "sa",
        "database.password": "gk",
        "database.names": "MyDB",
        "table.include.list": "dbo.Customer",
        "topic.prefix": "mydb",
        "schema.history.internal.kafka.bootstrap.servers": "localhost:9092",
        "schema.history.internal.kafka.topic": "schema-changes.mssql.mydb",
        "database.encrypt": "false"
    }
}
```
