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

  我们的任务是使得 TiDB 启动事务时能打印出一个 “hello transaction” 的日志，所以可以简单的认为，`Transaction` 每进行一次 `Commit` 就会有一次事务发生。所以顺藤摸瓜，找到了接口的实现`store/tikv/txn.go`

  ```go
  func (txn *tikvTxn) Commit(ctx context.Context) error {
  	// logutil.BgLogger().Info("[test] hello transaction")
  	if span := opentracing.SpanFromContext(ctx); span != nil && span.Tracer() != nil {
  		span1 := span.Tracer().StartSpan("tikvTxn.Commit", opentracing.ChildOf(span.Context()))
  		defer span1.Finish()
  		ctx = opentracing.ContextWithSpan(ctx, span1)
  	}
  
  	if !txn.valid {
  		return kv.ErrInvalidTxn
  	}
  	defer txn.close()
      ...
  }
  ```

  然后在函数开始执行的时候加入日志打印

  ## 3 编译部署

  首先我们使用 `make` 直接在代码目录下进行编译

  ```shell
  smilencer@smilencer-OptiPlex-7040 ~/Code/golang/src/github.com/pingcap/tidb (weak1) $ make
  CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=v4.0.0-beta.2-962-g738f677ee" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-16 10:42:11" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=738f677eee4c82e45aee3899b1be7b653bfaa759" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=weak1" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go
  Build TiDB Server successfully!
  ```

  然后我们使用 TiUP 进行临时二进制包部署

  > [https://docs.pingcap.com/zh/tidb/dev/tiup-playground#%E6%9B%BF%E6%8D%A2%E9%BB%98%E8%AE%A4%E7%9A%84%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%87%E4%BB%B6](https://docs.pingcap.com/zh/tidb/dev/tiup-playground#替换默认的二进制文件)

  ```shell
  tiup playground --db.binpath /xx/tidb-server
  Starting component `playground`: /home/smilencer/.tiup/components/playground/v1.0.9/tiup-playground --db.binpath /home/smilencer/Code/golang/src/github.com/pingcap/tidb/bin/tidb-server
  Use the latest stable version: v4.0.4
  
      Specify version manually:   tiup playground <version>
      The stable version:         tiup playground v4.0.0
      The nightly version:        tiup playground nightly
  
  Playground Bootstrapping...
  Start pd instance...
  Start tikv instance...
  Start tidb instance...
  .................................................
  Waiting for tikv 127.0.0.1:41187 ready 
  Start tiflash instance...
  Waiting for tiflash 127.0.0.1:3930 ready ...........................................
  CLUSTER START SUCCESSFULLY, Enjoy it ^-^
  To connect TiDB: mysql --host 127.0.0.1 --port 4000 -u root
  To view the dashboard: http://127.0.0.1:46263/dashboard
  To view the Prometheus: http://127.0.0.1:36189
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
  mysql> BEGIN;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> START TRANSACTION;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> START TRANSACTION WITH CONSISTENT SNAPSHOT;
  Query OK, 0 rows affected (0.00 sec)
  ```

  在 http://127.0.0.1:46263/dashboard/ 中寻找日志

  

  
