## R2DBCによるデータベースアクセス

次は[R2DBC](https://r2dbc.io)を用いて`ExpenditureBuilderRepository`のデータベース実装を作成します。

`pom.xml`に次の`dependency`を追加してください。

```xml
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-r2dbc</artifactId>
            <version>1.0.0.M2</version>
        </dependency>
        <dependency>
            <groupId>io.r2dbc</groupId>
            <artifactId>r2dbc-h2</artifactId>
        </dependency>
        <dependency>
            <groupId>io.r2dbc</groupId>
            <artifactId>r2dbc-postgresql</artifactId>
        </dependency>
```

`R2dbcExpenditureRepository`を作成して次の内容を記述してください。**TODO部分を実装してください**。

* [参考資料](https://docs.spring.io/spring-data/r2dbc/docs/1.0.0.M2/reference/html/#r2dbc.datbaseclient.queries)


```java
package com.example.expenditure;

import org.springframework.data.r2dbc.core.DatabaseClient;
import org.springframework.transaction.reactive.TransactionalOperator;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import static org.springframework.data.r2dbc.query.Criteria.where;

public class R2dbcExpenditureRepository implements ExpenditureRepository {

    private final DatabaseClient databaseClient;

    private final TransactionalOperator tx;

    public R2dbcExpenditureRepository(DatabaseClient databaseClient, TransactionalOperator tx) {
        this.databaseClient = databaseClient;
        this.tx = tx;
    }

    @Override
    public Flux<Expenditure> findAll() {
        return this.databaseClient.select().from(Expenditure.class)
            .as(Expenditure.class)
            .all();
    }

    @Override
    public Mono<Expenditure> findById(Integer expenditureId) {
        // TODO
        // "expenditure_id"が引数のexpenditureIdに一致する1件のExpenditureを返す
        return Mono.empty();
    }

    @Override
    public Mono<Expenditure> save(Expenditure expenditure) {
        return this.databaseClient.insert().into(Expenditure.class)
            .using(expenditure)
            .fetch()
            .one()
            .map(map -> new ExpenditureBuilder(expenditure)
                .withExpenditureId((Integer) map.get("expenditure_id"))
                .createExpenditure())
            .as(this.tx::transactional);
    }

    @Override
    public Mono<Void> deleteById(Integer expenditureId) {
        return this.databaseClient.delete().from(Expenditure.class)
            .matching(where("expenditure_id").is(expenditureId))
            .then()
            .as(this.tx::transactional);
    }
}
```

`App.java`に次のメソッドを追加してください。


```java
    static ConnectionFactory connectionFactory() {
        // postgresql://username:password@hostname:5432/dbname
        String databaseUrl = Optional.ofNullable(System.getenv("DATABASE_URL")).orElse("h2:file:///./target/demo?options=DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE");
        return ConnectionFactories.get("r2dbc:" + url(databaseUrl));
    }

    static String url(String databaseUrl) {
        URI uri = URI.create(databaseUrl);
        // https://github.com/r2dbc/r2dbc-postgresql/pull/117
        return ("postgres".equals(uri.getScheme()) ? databaseUrl.replace("postgres", "postgresql") : databaseUrl);
    }

    public static Mono<Void> initializeDatabase(String name, DatabaseClient databaseClient) {
        if ("H2".equals(name)) {
            return databaseClient.execute()
                .sql("CREATE TABLE IF NOT EXISTS expenditure (expenditure_id INT PRIMARY KEY AUTO_INCREMENT, expenditure_name VARCHAR(255), unit_price INT NOT NULL, quantity INT NOT NULL, " +
                    "expenditure_date DATE NOT NULL)")
                .then();
        } else if ("PostgreSQL".equals(name)) {
            return databaseClient.execute()
                .sql("CREATE TABLE IF NOT EXISTS expenditure (expenditure_id SERIAL PRIMARY KEY, expenditure_name VARCHAR(255), unit_price INT NOT NULL, quantity INT NOT NULL, " +
                    "expenditure_date DATE NOT NULL)")
                .then();
        }
        return Mono.error(new IllegalStateException(name + " is not supported."));
    }
```

また、`routes`メソッドも次のように変更してください。

```java
    static RouterFunction<ServerResponse> routes() {
        final ConnectionFactory connectionFactory = connectionFactory();
        final DatabaseClient databaseClient = DatabaseClient.builder()
            .connectionFactory(connectionFactory)
            .build();
        final TransactionalOperator transactionalOperator = TransactionalOperator.create(new R2dbcTransactionManager(connectionFactory));

        initializeDatabase(connectionFactory.getMetadata().getName(), databaseClient).subscribe();

        return new ExpenditureHandler(new R2dbcExpenditureRepository(databaseClient, transactionalOperator)).routes();
    }
```

`App`クラスの`main`メソッドを実行して、次のリクエストを送り、正しくレスポンスが返ることを確認してください。

```
$ curl localhost:8080/expenditures -d "{\"expenditureName\":\"コーヒー\",\"unitPrice\":300,\"quantity\":1,\"expenditureDate\":\"2019-06-03\"}" -H "Content-Type: application/json"
{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":"2019-06-03"}
```

```
$ curl localhost:8080/expenditures
[{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":"2019-06-03"}]
```

```
$ curl localhost:8080/expenditures/1
{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":"2019-06-03"}
```

```
$ curl -XDELETE localhost:8080/expenditures/1
```

```
$ curl localhost:8080/expenditures
[]
```

<details>
  <summary><code>R2dbcExpenditureRepository</code>の正解例</summary>

```java
package com.example.expenditure;

import org.springframework.data.r2dbc.core.DatabaseClient;
import org.springframework.transaction.reactive.TransactionalOperator;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import static org.springframework.data.r2dbc.query.Criteria.where;

public class R2dbcExpenditureRepository implements ExpenditureRepository {

    private final DatabaseClient databaseClient;

    private final TransactionalOperator tx;

    public R2dbcExpenditureRepository(DatabaseClient databaseClient, TransactionalOperator tx) {
        this.databaseClient = databaseClient;
        this.tx = tx;
    }

    @Override
    public Flux<Expenditure> findAll() {
        return this.databaseClient.select().from(Expenditure.class)
            .as(Expenditure.class)
            .all();
    }

    @Override
    public Mono<Expenditure> findById(Integer expenditureId) {
        return this.databaseClient.select().from(Expenditure.class)
            .matching(where("expenditure_id").is(expenditureId))
            .as(Expenditure.class)
            .one();
    }

    @Override
    public Mono<Expenditure> save(Expenditure expenditure) {
        return this.databaseClient.insert().into(Expenditure.class)
            .using(expenditure)
            .fetch()
            .one()
            .map(map -> new ExpenditureBuilder(expenditure)
                .withExpenditureId((Integer) map.get("expenditure_id"))
                .createExpenditure())
            .as(this.tx::transactional);
    }

    @Override
    public Mono<Void> deleteById(Integer expenditureId) {
        return this.databaseClient.delete().from(Expenditure.class)
            .matching(where("expenditure_id").is(expenditureId))
            .then()
            .as(this.tx::transactional);
    }
}
```

</details>

### Cloud Foundryへのデプロイ

Pivotal Web ServicesでPostgreSQLサービスをプロビジョニングします。`cf create-service`コマンドで`moneyger-db`インスタンスを作成してください。

```
cf create-service elephantsql turtle moneyger-db
```

`manifest.yml`も更新してください。

```yaml
applications:
- name: moneyger
  path: target/moneyger-1.0.0-SNAPSHOT.jar
  memory: 128m
  env:
    JAVA_OPTS: '-XX:ReservedCodeCacheSize=22M -XX:MaxDirectMemorySize=22M -XX:MaxMetaspaceSize=54M -Xss512K'
    JBP_CONFIG_OPEN_JDK_JRE: '[memory_calculator: {stack_threads: 30}]'
  services:
  - moneyger-db
```

ビルドして`cf push`してください。

```
./mvnw clean package -DskipTests=true
cf push
```

次のリクエストを送り、正しくレスポンスが返ることを確認してください。

```
curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures -d "{\"expenditureName\":\"コーヒー\",\"unitPrice\":300,\"quantity\":1,\"expenditureDate\":\"2019-06-03\"}" -H "Content-Type: application/json"
cf restart moneyger
curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures/1
```

### Connection Poolの設定

今回使用したPostgreSQLサービスは無料プランを使用しており、同時接続数が4という制限があります。
現在のコードでは5コネクション以上同時に接続するとエラーが発生します。

Connection Poolを設定し、最大Pool数を4にすることでエラーを防ぎます。

`pom.xml`に次の`dependency`を追加してください。

```xml
        <dependency>
            <groupId>io.r2dbc</groupId>
            <artifactId>r2dbc-pool</artifactId>
        </dependency>
```

`App.java`に次のメソッドを追加してください。

```java
    static ConnectionPool connectionPool(ConnectionFactory connectionFactory) {
        return new ConnectionPool(ConnectionPoolConfiguration.builder(connectionFactory)
            .maxSize(4)
            .maxIdleTime(Duration.ofSeconds(3))
            .validationQuery("SELECT 1")
            .build());
    }
```

`routes`メソッドの次の箇所を、


```java
    static RouterFunction<ServerResponse> routes() {
        final ConnectionFactory connectionFactory = connectionFactory();
        // ...
    }
```

次のように変更してください。

```java
    static RouterFunction<ServerResponse> routes() {
        final ConnectionFactory connectionFactory = connectionPool(connectionFactory());
        // ...
    }
```

ビルドして`cf push`してください。

```
./mvnw clean package -DskipTests=true
cf push
```

[`wrk`](https://github.com/wg/wrk)コマンドで負荷をかけてもエラーが発生しないことが確認できます。

```
wrk -t32 -c100 -d60s --latency --timeout 30s https://moneyger-<CHANGE ME>.cfapps.io/expenditures
```

```
Running 1m test @ https://moneyger-chatty-zebra.cfapps.io/expenditures
  32 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   266.43ms  121.88ms   1.41s    88.77%
    Req/Sec    12.60      5.94    30.00     57.44%
  Latency Distribution
     50%  223.84ms
     75%  292.53ms
     90%  404.77ms
     99%  745.04ms
  21965 requests in 1.00m, 6.37MB read
Requests/sec:    365.52
Transfer/sec:    108.51KB
```
