# kakfa connect

## 환경설정

### server.properties

카프카 서버를 구동하기 위한 환경 설정  

```bash
$ vi ~/confluent/etc/kafka/server.properties
############################# Socket Server Settings #############################

# The address the socket server listens on. It will get the value returned from
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
# 모든 외부 네트워크에서 접속을 허용하겠다는 뜻.
listeners=PLAINTEXT://0.0.0.0:9092

# Hostname and port the broker will advertise to producers and consumers. If not set,
# it uses the value for "listeners" if configured.  Otherwise, it will use the value
# returned from java.net.InetAddress.getCanonicalHostName().
advertised.listeners=PLAINTEXT://localhost:9092
```

### connect-distributed.properties

kafka connector에 사용할 환경 설정  

```bash
$ vi ~/confluent/etc/kafka/connect-distributed.properties
# A list of host/port pairs to use for establishing the initial connection to the Kafka cluster.
bootstrap.servers=localhost:9092
```

## topic 리셋

* 작업하다가 보면 삭제가 제대도 되지 않는 토픽 발생
* 모든 토픽을 다 지우고 새롭게 하려면 아래의 순서대로 하기

1. 커넥터 종료
2. 카프카 서비스 종료
3. zookeeper만 서비스가 된 상태로 아래의 명령어(zookeeper-shell)를 순서대로 실행함.
4. 삭제 후 데이터 폴더도 삭제함.
5. 카프카 서비스 시작
6. 커넥터 서비스 시작
7. 데이터 폴더에 정상적으로 topic이 생성되어 있음.

```bash
$ zookeeper-shell localhost:2181
# Connecting to localhost:2181
# Welcome to ZooKeeper!
# JLine support is disabled
# 
# WATCHER::
# 
# WatchedEvent state:SyncConnected type:None path:null
ls /brokers/topics      # 모든 topic의 이름이 표시된다. connect의 경우 offset 토픽도 모두 표시된다.
# [__consumer_offsets, connect-configs, connect-offsets, connect-status]
deleteall /brokers/topics  # 아무런 메세지가 없어도 삭제가 됨
# 확인하기
ls /brokers/topics
# Node does not exist: /brokers/topics
```

## offset.storage.topic에 구성된 오프셋을 지움

Kafka Connect 내부를 조작하는 고도의 기술적인 작업이며 이 방법은 최후의 수단으로만 사용.

1. 플러그인 오프셋이 포함된 주제의 이름을 찾는 것. ```offset.storage.topic``` 옵션에 구성되어 있음.
2. 지정된 커넥터에 대한 마지막 오프셋, 즉 해당 커넥터가 저장된 키를 찾고 오프셋을 저장하는 데 사용된 파티션을 식별하는 것. 다음 명령어로 찾을 수 있음.

```bash
$ kafkacat -b localhost -C -t my_connect_offsets -f 'Partition(%p) %k %s\n'
# ----- 실행 결과 -----------------
Partition(11) ["inventory-connector",{"server":"dbserver1"}] {"ts_sec":1530088501,"file":"mysql-bin.000003","pos":817,"row":1,"server_id":223344,"event":2}
Partition(11) ["inventory-connector",{"server":"dbserver1"}] {"ts_sec":1530168941,"file":"mysql-bin.000004","pos":3261,"row":1,"server_id":223344,"event":2}
```

* inventory-connector를 위한 key : ```["inventory-connector",{"server":"dbserver1"}]```
* partition number : 11
* 마지막 offset : ```{"ts_sec":1530168941,"file":"mysql-bin.000004","pos":3261,"row":1,"server_id":223344,"event":2}```

제거 방법
* 키를 이용해서 해당 값을 찾아 모두 null 로 변경한다.

```bash
$ echo '["inventory-connector",{"server":"dbserver1"}]|' | \
kafkacat -P -Z -b localhost -t my_connect_offsets -K \| -p 11
```

## C# debezium topic consumer source

.NET Core 6.0 으로 개발환경을 설정해야 System.Text.Json을 사용할 수 있음.

```C#
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Text;
using System.Text.Json.Nodes;
using System.Text.RegularExpressions;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using Confluent.Kafka;

namespace KafkaConsumer {
    internal class Program {
        static void Main(string[] args) {

            var conf = new ConsumerConfig {
                GroupId = "test-consumer-group",
                ClientId = "mydb",
                BootstrapServers = "localhost:9092",
                // 참고: AutoOffsetReset 속성은 이벤트의 시작 오프셋을 결정합니다.
                // Consumer 그룹에 대한 커밋된 오프셋이 아직 없습니다.
                // 관심 있는 topic/파티션. 기본적으로 오프셋이 커밋됩니다.
                // 자동으로, 이 예에서는 Consumer가 다음부터 시작됩니다.
                // 프로그램을 처음 실행할 때 'my-topic' 주제에 가장 먼저 메시지가 표시됩니다.
                AutoOffsetReset = AutoOffsetReset.Latest,
                EnableAutoCommit = false
            };

            CancellationTokenSource cts = new CancellationTokenSource();
            Console.CancelKeyPress += (_, e) => {
                e.Cancel = true;
                cts.Cancel();
            };

            var c = new ConsumerBuilder<Ignore, string>(conf);

            using var cb = c.Build();
            TopicPartitionOffset tps = new TopicPartitionOffset(new TopicPartition("mydb.MyDB.dbo.Customer", 0), Offset.Beginning);
            cb.Assign(tps);
            // c.Subscribe("mydb.MyDB.dbo.Customer");
            cb.Subscribe("mydb.MyDB.dbo.Customer");

            try {
                while (true) {
                    try {
                        var cr = cb.Consume(cts.Token);
                        JsonNode json = JsonObject.Parse(cr.Message.Value);
                        Console.WriteLine($"{cr.TopicPartitionOffset} - {DecodeEncodedNonAsciiCharacters(json["payload"]["after"].ToString())}");
                        //Console.WriteLine($"Consumed Message '{cr.Value}' at '{cr.TopicPartitionOffset}'.");
                    }
                    catch (ConsumeException ce) {
                        Console.WriteLine($"Error occured : {ce.Error.Reason}");
                    }
                }
            }
            catch (OperationCanceledException oe) {
                cb.Close();
            }
        }

        /// <summary>
        /// 유니코드를 한글로 변환하기
        /// </summary>
        /// <param name="value">문자열</param>
        /// <returns></returns>
        static string DecodeEncodedNonAsciiCharacters(string value) {
            return Regex.Replace(
                value,
                @"\\u(?<Value>[a-zA-Z0-9]{4})",
                m => ((char)int.Parse(m.Groups["Value"].Value, NumberStyles.HexNumber)).ToString());
        }

    }
}
```