- # Talent plan weak1

  ## 1 下载源码

  ```shell
  cd $GOPATH/src/github.com/pingcap
  git clone https://github.com/pingcap/tidb
  gmake #编译源码
  ```

  ## 2 改写代码

  > https://pingcap.com/blog-cn/tidb-source-code-reading-2/#kv-api-layer

  通过这一篇文章我们可以找到  `tidb/kv/kv.go` 这个文件，它定义的 `Transaction` 接口描述了 TiDB 对于事务的基本操作

  ```go
  // Transaction defines the interface for operations inside a Transaction.
  // This is not thread safe.
  type Transaction interface {
  	...
  	// Commit commits the transaction operations to KV store.
  	Commit(context.Context) error
  	...
  }
  ```

  然后顺藤摸瓜，我们找到了其同文件夹下的 `kv/txn.go`，通过名字我们大致推测出他跟事务操作有关。

  ```go
  // RunInNewTxn will run the f in a new transaction environment.
  func RunInNewTxn(store Storage, retryable bool, f func(txn Transaction) error) error {
  	var (
  		err           error
  		originalTxnTS uint64
  		txn           Transaction
  	)
      // hello transaction
  	logutil.BgLogger().Info("[test] hello transaction")
  	for i := uint(0); i < maxRetryCnt; i++ {
  		txn, err = store.Begin()
  		if err != nil {
  			logutil.BgLogger().Error("RunInNewTxn", zap.Error(err))
  			return err
  		}
  ```

  然后找到了 `RunInNewTxn` 这个函数 ，在函数开始执行的时候加入日志打印结果

  ## 3 编译部署

  首先我们使用 `make` 直接在代码目录下进行编译

  ```shell
  smilencer@smilencer-OptiPlex-7040 ~/Code/golang/src/github.com/pingcap/tidb (weak1) $ make
  CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=v4.0.0-beta.2-962-g738f677ee" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-16 10:42:11" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=738f677eee4c82e45aee3899b1be7b653bfaa759" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=weak1" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go
  Build TiDB Server successfully!
  ```

  然后我们使用 TiUP 进行临时二进制包部署，根据课程要求设为 `--db 1 --pd 1 --kv 3`

  > [https://docs.pingcap.com/zh/tidb/dev/tiup-playground#%E6%9B%BF%E6%8D%A2%E9%BB%98%E8%AE%A4%E7%9A%84%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%87%E4%BB%B6](https://docs.pingcap.com/zh/tidb/dev/tiup-playground#替换默认的二进制文件)

  ```shell
  smilencer@smilencer-OptiPlex-7040 ~ $ tiup playground --db.binpath  /home/smilencer/Code/golang/src/github.com/pingcap/tidb/bin/tidb-server --db 1 --pd 1 --kv 3 --monitor
  Starting component `playground`: /home/smilencer/.tiup/components/playground/v1.0.9/tiup-playground --db.binpath /home/smilencer/Code/golang/src/github.com/pingcap/tidb/bin/tidb-server --db 1 --pd 1 --kv 3 --monitor
  Use the latest stable version: v4.0.4
  
      Specify version manually:   tiup playground <version>
      The stable version:         tiup playground v4.0.0
      The nightly version:        tiup playground nightly
  
  Playground Bootstrapping...
  Start pd instance...
  Start tikv instance...
  Start tikv instance...
  Start tikv instance...
  Start tidb instance...
  ............................................................To view the dashboard: http://127.0.0.1:37207/dashboard
  To view the Prometheus: http://127.0.0.1:39749
  To view the Grafana: http://127.0.0.1:3000
  ```

  ## 4 运行结果

  新开启一个 session 以访问 TiDB 数据库

  ```SHELL
  mysql --host 127.0.0.1 --port 4000 -u root
  ```

  尝试使用开启事务的语句

  > [https://docs.pingcap.com/zh/tidb/dev/transaction-overview#%E5%B8%B8%E7%94%A8%E4%BA%8B%E5%8A%A1%E8%AF%AD%E5%8F%A5](https://docs.pingcap.com/zh/tidb/dev/transaction-overview#常用事务语句)

  ```sql
  ...
  begin;
  insert into test values (1);
  insert into test values (2);
  insert into test values (3);
  commit;
  ```

  在 http://127.0.0.1:37207/dashboard 中寻找日志

  ```
  [2020/08/16 20:31:12.600 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:13.600 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:13.600 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.424 +08:00] [Info] [session.go:2257] ["CRUCIAL OPERATION"] [conn=15] [schemaVersion=23] [cur_db=mysql] [sql="CREATE TABLE test(\nid INT(11)\n)"] [user=root@127.0.0.1]
  [2020/08/16 20:31:14.425 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.492 +08:00] [Warn] [2pc.go:1006] ["schemaLeaseChecker is not set for this transaction, schema check skipped"] [connID=0] [startTS=418796293169086467] [commitTS=418796293182193665]
  [2020/08/16 20:31:14.556 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.600 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.600 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.701 +08:00] [Warn] [2pc.go:1006] ["schemaLeaseChecker is not set for this transaction, schema check skipped"] [connID=0] [startTS=418796293208408065] [commitTS=418796293234622466]
  [2020/08/16 20:31:14.852 +08:00] [Info] [ddl_worker.go:260] ["[ddl] add DDL jobs"] ["batch count"=1] [jobs="ID:48, Type:create table, State:none, SchemaState:none, SchemaID:3, TableID:47, RowCount:0, ArgLen:1, start time: 2020-08-16 20:31:14.556 +0800 CST, Err:<nil>, ErrCount:0, SnapshotVersion:0; "]
  [2020/08/16 20:31:14.852 +08:00] [Info] [ddl.go:475] ["[ddl] start DDL job"] [job="ID:48, Type:create table, State:none, SchemaState:none, SchemaID:3, TableID:47, RowCount:0, ArgLen:1, start time: 2020-08-16 20:31:14.556 +0800 CST, Err:<nil>, ErrCount:0, SnapshotVersion:0"] [query="CREATE TABLE test(\nid INT(11)\n)"]
  [2020/08/16 20:31:14.852 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.853 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:14.853 +08:00] [Info] [ddl_worker.go:589] ["[ddl] run DDL job"] [worker="worker 3, tp general"] [job="ID:48, Type:create table, State:none, SchemaState:none, SchemaID:3, TableID:47, RowCount:0, ArgLen:0, start time: 2020-08-16 20:31:14.556 +0800 CST, Err:<nil>, ErrCount:0, SnapshotVersion:0"]
  [2020/08/16 20:31:14.854 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:15.352 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:15.600 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:15.620 +08:00] [Warn] [2pc.go:1006] ["schemaLeaseChecker is not set for this transaction, schema check skipped"] [connID=0] [startTS=418796293273944066] [commitTS=418796293483659265]
  [2020/08/16 20:31:15.645 +08:00] [Info] [prewrite.go:166] ["prewrite encounters lock"] [conn=0] [lock="key: []byte{0x6d, 0x44, 0x42, 0x3a, 0x33, 0x0, 0x0, 0x0, 0x0, 0xfb, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x48}, primary: []byte{0x6d, 0x44, 0x42, 0x3a, 0x33, 0x0, 0x0, 0x0, 0x0, 0xfb, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x48}, txnStartTS: 418796293273944066, lockForUpdateTS:0, ttl: 3001, type: Put"]
  [2020/08/16 20:31:15.697 +08:00] [Warn] [txn.go:67] [RunInNewTxn] ["retry txn"=418796293273944068] ["original txn"=418796293273944068] [error="[kv:9007]Write conflict, txnStartTS=418796293273944068, conflictStartTS=418796293273944066, conflictCommitTS=418796293483659265, key=[]byte{0x6d, 0x44, 0x42, 0x3a, 0x33, 0x0, 0x0, 0x0, 0x0, 0xfb, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x48} primary=[]byte{0x6d, 0x44, 0x42, 0x3a, 0x33, 0x0, 0x0, 0x0, 0x0, 0xfb, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x48} [try again later]"]
  [2020/08/16 20:31:15.754 +08:00] [Info] [domain.go:127] ["diff load InfoSchema success"] [usedSchemaVersion=23] [neededSchemaVersion=24] ["start time"=56.598995ms] [phyTblIDs="[47]"] [actionTypes="[8]"]
  [2020/08/16 20:31:15.852 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:15.863 +08:00] [Info] [2pc.go:717] ["2PC clean up done"] [txnStartTS=418796293273944068]
  [2020/08/16 20:31:15.863 +08:00] [Info] [syncer.go:399] ["[ddl] syncer check all versions, someone is not synced, continue checking"] [ddl=/tidb/ddl/all_schema_versions/8a240190-cef1-4fc2-a5d0-a068e3081dbe] [currentVer=23] [latestVer=24]
  [2020/08/16 20:31:15.884 +08:00] [Info] [ddl_worker.go:783] ["[ddl] wait latest schema version changed"] [worker="worker 3, tp general"] [ver=24] ["take time"=188.518564ms] [job="ID:48, Type:create table, State:done, SchemaState:public, SchemaID:3, TableID:47, RowCount:0, ArgLen:1, start time: 2020-08-16 20:31:14.556 +0800 CST, Err:<nil>, ErrCount:0, SnapshotVersion:0"]
  [2020/08/16 20:31:15.884 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:15.887 +08:00] [Info] [ddl_worker.go:366] ["[ddl] finish DDL job"] [worker="worker 3, tp general"] [job="ID:48, Type:create table, State:synced, SchemaState:public, SchemaID:3, TableID:47, RowCount:0, ArgLen:0, start time: 2020-08-16 20:31:14.556 +0800 CST, Err:<nil>, ErrCount:0, SnapshotVersion:0"]
  [2020/08/16 20:31:16.014 +08:00] [Warn] [2pc.go:1006] ["schemaLeaseChecker is not set for this transaction, schema check skipped"] [connID=0] [startTS=418796293549195265] [commitTS=418796293588516865]
  [2020/08/16 20:31:16.014 +08:00] [Warn] [2pc.go:1006] ["schemaLeaseChecker is not set for this transaction, schema check skipped"] [connID=0] [startTS=418796293496766467] [commitTS=418796293588516866]
  [2020/08/16 20:31:16.186 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:16.187 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:16.187 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:16.188 +08:00] [Info] [ddl.go:507] ["[ddl] DDL job is finished"] [jobID=48]
  [2020/08/16 20:31:16.299 +08:00] [Info] [domain.go:643] ["performing DDL change, must reload"]
  [2020/08/16 20:31:16.300 +08:00] [Info] [split_region.go:59] ["split batch regions request"] ["split key count"=1] ["batch count"=1] ["first batch, region ID"=4] ["first split key"=74800000000000002f]
  [2020/08/16 20:31:16.599 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:16.599 +08:00] [Info] [txn.go:35] ["[test] hello transaction"]
  [2020/08/16 20:31:16.631 +08:00] [Info] [split_region.go:156] ["batch split regions complete"] ["batch region ID"=4] ["first at"=74800000000000002f] ["first new region left"="{Id:103 StartKey:7480000000000000ff2d00000000000000f8 EndKey:7480000000000000ff2f00000000000000f8 RegionEpoch:{ConfVer:5 Version:23} Peers:[id:104 store_id:1  id:105 store_id:2  id:106 store_id:3 ]}"] ["new region count"=1]
  [2020/08/16 20:31:16.631 +08:00] [Info] [split_region.go:205] ["split regions complete"] ["region count"=1] ["region IDs"="[103]"]
  ```

  发现出现了很多`hello transaction`

  

  
