## R2DBCによるデータベースアクセス

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


```java
    static ConnectionFactory connectionFactory() {
        // postgresql://username:password@hostname:5432/dbname
        String databaseUrl = Optional.ofNullable(System.getenv("DATABASE_URL")).orElse("h2:file:///./target/demo?options=DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE");
        return ConnectionFactories.get("r2dbc:" + url(databaseUrl));
    }

    static String url(String databaseUrl) {
        URI uri = URI.create(databaseUrl);
        return ("postgres".equals(uri.getScheme()) ? databaseUrl.replace("postgres", "postgresql") : databaseUrl);
    }

    static void initializeDatabase(String name, DatabaseClient databaseClient) {
        if ("H2".equals(name)) {
            databaseClient.execute()
                .sql("CREATE TABLE IF NOT EXISTS expenditure (expenditure_id INT PRIMARY KEY AUTO_INCREMENT, expenditure_name VARCHAR(255), unit_price INT NOT NULL, quantity INT NOT NULL, " +
                    "expenditure_date DATE NOT NULL)")
                .then()
                .then(databaseClient.execute()
                    .sql("CREATE TABLE IF NOT EXISTS income (income_id INT PRIMARY KEY AUTO_INCREMENT, income_name VARCHAR(255), amount INT NOT NULL, income_date DATE NOT NULL)")
                    .then())
                .subscribe();
        } else if ("PostgreSQL".equals(name)) {
            databaseClient.execute()
                .sql("CREATE TABLE IF NOT EXISTS expenditure (expenditure_id SERIAL PRIMARY KEY, expenditure_name VARCHAR(255), unit_price INT NOT NULL, quantity INT NOT NULL, " +
                    "expenditure_date DATE NOT NULL)")
                .then()
                .then(databaseClient.execute()
                    .sql("CREATE TABLE IF NOT EXISTS income (income_id SERIAL PRIMARY KEY, income_name VARCHAR(255), amount INT NOT NULL, income_date DATE NOT NULL)")
                    .then())
                .subscribe();
        }
    }
```

```java
    static RouterFunction<ServerResponse> routes() {
        final ConnectionFactory connectionFactory = connectionFactory();
        final DatabaseClient databaseClient = DatabaseClient.builder()
            .connectionFactory(connectionFactory)
            .build();
        final TransactionalOperator transactionalOperator = TransactionalOperator.create(new R2dbcTransactionManager(connectionFactory));

        initializeDatabase(connectionFactory.getMetadata().getName(), databaseClient);

        return new ExpenditureHandler(new R2dbcExpenditureRepository(databaseClient, transactionalOperator)).routes();
    }
```

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

```
cf create-service elephantsql turtle moneyger-db
```

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

```
./mvnw clean package
cf push
```


```
curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures -d "{\"expenditureName\":\"コーヒー\",\"unitPrice\":300,\"quantity\":1,\"expenditureDate\":\"2019-06-03\"}" -H "Content-Type: application/json"
cf restart moneyger
curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures/1
```

### Connection Poolの設定

```xml
        <dependency>
            <groupId>io.r2dbc</groupId>
            <artifactId>r2dbc-pool</artifactId>
        </dependency>
```

```java
    static ConnectionPool connectionPool(ConnectionFactory connectionFactory) {
        return new ConnectionPool(ConnectionPoolConfiguration.builder(connectionFactory)
            .maxSize(4)
            .maxIdleTime(Duration.ofSeconds(3))
            .validationQuery("SELECT 1")
            .build());
    }
```

```java
    static RouterFunction<ServerResponse> routes() {
        final ConnectionFactory connectionFactory = connectionFactory();
        // ...
    }
```

↓

```java
    static RouterFunction<ServerResponse> routes() {
        final ConnectionFactory connectionFactory = connectionPool(connectionFactory());
        // ...
    }
```

```
./mvnw clean package
cf push
```


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
