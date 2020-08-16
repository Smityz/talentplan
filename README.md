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

  

  
