# kakfa connect

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

