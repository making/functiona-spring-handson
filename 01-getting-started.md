## 簡易家計簿Moneygerプロジェクトの作成

簡易家計簿Moneygerのプロジェクトを作成します。

プロジェクトの雛形はMaven Archetypeの[vanilla-spring-webflux-fn-blank](https://github.com/making/vanilla-spring-webflux-fn-blank)を使用します。


次のコマンドで雛形プロジェクトを作成してください。Windowsの場合はGit BashなどのBash実行環境を使用してください。

```
mvn archetype:generate\
 -DarchetypeGroupId=am.ik.archetype\
 -DarchetypeArtifactId=vanilla-spring-webflux-fn-blank-archetype\
 -DarchetypeVersion=0.2.7\
 -DgroupId=com.example\
 -DartifactId=moneyger\
 -Dversion=1.0.0-SNAPSHOT\
 -B
```

同梱のサンプルコードの動作を確認します。

```
cd moneyger
chmod +x mvnw
./mvnw clean package -DskipTests=true

java -jar target/moneyger-1.0.0-SNAPSHOT.jar
```

```
$ curl localhost:8080
Hello World!

$ curl localhost:8080/messages -d "{\"text\":\"Hello\"}" -H "Content-Type: application/json"
{"text":"Hello"}

$ curl localhost:8080/messages
[{"text":"Hello"}]
```

### `Expenditure`(支出)モデルの作成

家計簿の支出モデルを作成します。

`com.example.expenditure`パッケージを作って以下の`Expenditure.java`と`ExpenditureBuilder.java`を作成してください。

```java
package com.example.expenditure;

import com.fasterxml.jackson.databind.annotation.JsonDeserialize;

import java.time.LocalDate;

@JsonDeserialize(builder = ExpenditureBuilder.class)
public class Expenditure {

    private final Integer expenditureId;

    private final String expenditureName;

    private final int unitPrice;

    private final int quantity;

    private final LocalDate expenditureDate;

    Expenditure(Integer expenditureId, String expenditureName, int unitPrice, int quantity, LocalDate expenditureDate) {
        this.expenditureId = expenditureId;
        this.expenditureName = expenditureName;
        this.unitPrice = unitPrice;
        this.quantity = quantity;
        this.expenditureDate = expenditureDate;
    }

    public Integer getExpenditureId() {
        return expenditureId;
    }


    public String getExpenditureName() {
        return expenditureName;
    }


    public int getUnitPrice() {
        return unitPrice;
    }


    public int getQuantity() {
        return quantity;
    }


    public LocalDate getExpenditureDate() {
        return expenditureDate;
    }


    @Override
    public String toString() {
        return "Expenditure{" +
            "expenditureId=" + expenditureId +
            ", expenditureName='" + expenditureName + '\'' +
            ", unitPrice=" + unitPrice +
            ", quantity=" + quantity +
            ", expenditureDate=" + expenditureDate +
            '}';
    }
}
```


```java
package com.example.expenditure;

import com.fasterxml.jackson.databind.annotation.JsonPOJOBuilder;

import java.time.LocalDate;

@JsonPOJOBuilder(buildMethodName = "createExpenditure")
public class ExpenditureBuilder {

    private int unitPrice;

    private LocalDate expenditureDate;

    private Integer expenditureId;

    private String expenditureName;

    private int quantity;

    public ExpenditureBuilder() {
    }

    public ExpenditureBuilder(Expenditure expenditure) {
        this.unitPrice = expenditure.getUnitPrice();
        this.expenditureDate = expenditure.getExpenditureDate();
        this.expenditureId = expenditure.getExpenditureId();
        this.expenditureName = expenditure.getExpenditureName();
        this.quantity = expenditure.getQuantity();
    }

    public Expenditure createExpenditure() {
        return new Expenditure(expenditureId, expenditureName, unitPrice, quantity, expenditureDate);
    }

    public ExpenditureBuilder withUnitPrice(int unitPrice) {
        this.unitPrice = unitPrice;
        return this;
    }

    public ExpenditureBuilder withExpenditureDate(LocalDate expenditureDate) {
        this.expenditureDate = expenditureDate;
        return this;
    }

    public ExpenditureBuilder withExpenditureId(Integer expenditureId) {
        this.expenditureId = expenditureId;
        return this;
    }

    public ExpenditureBuilder withExpenditureName(String expenditureName) {
        this.expenditureName = expenditureName;
        return this;
    }

    public ExpenditureBuilder withQuantity(int quantity) {
        this.quantity = quantity;
        return this;
    }
}
```

### `ExpenditureRepository`の作成

`com.example.expenditure`パッケージに`ExpenditureBuilderRepository`インタフェースを作成してください。

```java
package com.example.expenditure;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface ExpenditureRepository {

    Flux<Expenditure> findAll();

    Mono<Expenditure> findById(Integer expenditureId);

    Mono<Expenditure> save(Expenditure expenditure);

    Mono<Void> deleteById(Integer expenditureId);
}
```

まずは`ExpenditureBuilderRepository`インタフェースのインメモリ実装である`InMemoryExpenditureRepository`を作成します。

```java
package com.example.expenditure;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Objects;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.atomic.AtomicInteger;

public class InMemoryExpenditureRepository implements ExpenditureRepository {

    final List<Expenditure> expenditures = new CopyOnWriteArrayList<>();

    final AtomicInteger counter = new AtomicInteger(1);

    @Override
    public Flux<Expenditure> findAll() {
        return Flux.fromIterable(this.expenditures);
    }

    @Override
    public Mono<Expenditure> findById(Integer expenditureId) {
        return Mono.justOrEmpty(this.expenditures.stream()
            .filter(x -> Objects.equals(x.getExpenditureId(), expenditureId))
            .findFirst());
    }

    @Override
    public Mono<Expenditure> save(Expenditure expenditure) {
        return Mono.fromCallable(() -> {
            Expenditure created = new ExpenditureBuilder(expenditure)
                .withExpenditureId(this.counter.getAndIncrement())
                .createExpenditure();
            this.expenditures.add(created);
            return created;
        });
    }

    @Override
    public Mono<Void> deleteById(Integer expenditureId) {
        return Mono.defer(() -> {
            this.expenditures.removeIf(x -> Objects.equals(x.getExpenditureId(), expenditureId));
            return Mono.empty();
        });
    }
}
```

### `ExpenditureHandler`の作成

次に`com.example.expenditure`パッケージに`ExpenditureHandler`クラスを作成し、WebFlux.fnの`RouterFunction`を実装します。

次の**TODO(3箇所)は実装しないといけない部分です**。動作を確認するためのテストコードは以下に続きます。TODOを実装する前にテストを実行してくだい。

* [参考資料](https://docs.spring.io/spring/docs/5.2.0.M2/spring-framework-reference/web-reactive.html#webflux-fn)

```java
package com.example.expenditure;

import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.RouterFunctions;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.util.UriComponentsBuilder;
import reactor.core.publisher.Mono;

import java.util.LinkedHashMap;

import static org.springframework.http.HttpStatus.NOT_FOUND;

public class ExpenditureHandler {

    private final ExpenditureRepository expenditureRepository;

    public ExpenditureHandler(ExpenditureRepository expenditureRepository) {
        this.expenditureRepository = expenditureRepository;
    }

    public RouterFunction<ServerResponse> routes() {
        return RouterFunctions.route()
            // TODO Routingの定義
            // GET /expenditures
            // POST /expenditures
            // GET /expenditures/{expenditureId}
            .DELETE("/expenditures/{expenditureId}", this::delete)
            .build();
    }

    Mono<ServerResponse> list(ServerRequest req) {
        return ServerResponse.ok().body(this.expenditureRepository.findAll(), Expenditure.class);
    }

    Mono<ServerResponse> post(ServerRequest req) {
        return req.bodyToMono(Expenditure.class)
            // TODO
            // ExpenditureRepositoryでExpenditureを保存
            .flatMap(created -> ServerResponse
                .created(UriComponentsBuilder.fromUri(req.uri()).path("/{expenditureId}").build(created.getExpenditureId()))
                .syncBody(created));
    }

    Mono<ServerResponse> get(ServerRequest req) {
        return this.expenditureRepository.findById(Integer.valueOf(req.pathVariable("expenditureId")))
            .flatMap(expenditure -> ServerResponse.ok().syncBody(expenditure))
            // TODO
            // expenditureが存在しない場合は404を返す。エラーレスポンスは{"status":404,"error":"Not Found","message":"The given expenditure is not found."}
            ;
    }

    Mono<ServerResponse> delete(ServerRequest req) {
        return ServerResponse.noContent()
            .build(this.expenditureRepository.deleteById(Integer.valueOf(req.pathVariable("expenditureId"))));
    }
}
```

`src/main/java/com/example/App.java`の`routes`メソッドを下のコードに変更してください。

```java
    static RouterFunction<ServerResponse> routes() {
        return new ExpenditureHandler(new InMemoryExpenditureRepository()).routes();
    }
```

`src/test/java/com/example/expenditure`に`ExpenditureHandlerTest`を作成して、次のテストコードを記述してください。TODO部分はそのままにしてください。

```java
package com.example.expenditure;

import com.fasterxml.jackson.databind.JsonNode;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.net.URI;
import java.time.LocalDate;
import java.util.Arrays;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

public class ExpenditureHandlerTest {

    private WebTestClient testClient;

    private InMemoryExpenditureRepository expenditureRepository = new InMemoryExpenditureRepository();

    private ExpenditureHandler expenditureHandler = new ExpenditureHandler(this.expenditureRepository);

    private List<Expenditure> fixtures = Arrays.asList(
        new ExpenditureBuilder()
            .withExpenditureId(1)
            .withExpenditureName("本")
            .withUnitPrice(2000)
            .withQuantity(1)
            .withExpenditureDate(LocalDate.of(2019, 4, 1))
            .createExpenditure(),
        new ExpenditureBuilder()
            .withExpenditureId(2)
            .withExpenditureName("コーヒー")
            .withUnitPrice(300)
            .withQuantity(2)
            .withExpenditureDate(LocalDate.of(2019, 4, 2))
            .createExpenditure());

    @BeforeAll
    void before() {
        this.testClient = WebTestClient.bindToRouterFunction(this.expenditureHandler.routes())
            .build();
    }

    @BeforeEach
    void reset() {
        this.expenditureRepository.expenditures.clear();
        this.expenditureRepository.expenditures.addAll(this.fixtures);
        this.expenditureRepository.counter.set(100);
    }

    @Test
    void list() {
        this.testClient.get()
            .uri("/expenditures")
            .exchange()
            .expectStatus().isOk()
            .expectBody(JsonNode.class)
            .consumeWith(result -> {
                JsonNode body = result.getResponseBody();
                assertThat(body).isNotNull();
                assertThat(body.size()).isEqualTo(2);

                assertThat(body.get(0).get("expenditureId").asInt()).isEqualTo(1);
                assertThat(body.get(0).get("expenditureName").asText()).isEqualTo("本");
                assertThat(body.get(0).get("unitPrice").asInt()).isEqualTo(2000);
                assertThat(body.get(0).get("quantity").asInt()).isEqualTo(1);
                // TODO 後で実装します
                //assertThat(body.get(0).get("expenditureDate").asText()).isEqualTo("2019-04-01");

                assertThat(body.get(1).get("expenditureId").asInt()).isEqualTo(2);
                assertThat(body.get(1).get("expenditureName").asText()).isEqualTo("コーヒー");
                assertThat(body.get(1).get("unitPrice").asInt()).isEqualTo(300);
                assertThat(body.get(1).get("quantity").asInt()).isEqualTo(2);
                // TODO 後で実装します
                //assertThat(body.get(1).get("expenditureDate").asText()).isEqualTo("2019-04-02");
            });
    }

    @Test
    void get_200() {
        this.testClient.get()
            .uri("/expenditures/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(JsonNode.class)
            .consumeWith(result -> {
                JsonNode body = result.getResponseBody();
                assertThat(body).isNotNull();

                assertThat(body.get("expenditureId").asInt()).isEqualTo(1);
                assertThat(body.get("expenditureName").asText()).isEqualTo("本");
                assertThat(body.get("unitPrice").asInt()).isEqualTo(2000);
                assertThat(body.get("quantity").asInt()).isEqualTo(1);
                // TODO 後で実装します
                //assertThat(body.get("expenditureDate").asText()).isEqualTo("2019-04-01");
            });
    }

    @Test
    void get_404() {
        this.testClient.get()
            .uri("/expenditures/10000")
            .exchange()
            .expectStatus().isNotFound()
            .expectBody(JsonNode.class)
            .consumeWith(result -> {
                JsonNode body = result.getResponseBody();
                assertThat(body).isNotNull();

                assertThat(body.get("status").asInt()).isEqualTo(404);
                assertThat(body.get("error").asText()).isEqualTo("Not Found");
                assertThat(body.get("message").asText()).isEqualTo("The given expenditure is not found.");
            });
    }

    @Test
    void post_201() {
        Expenditure expenditure = new ExpenditureBuilder()
            .withExpenditureName("ビール")
            .withUnitPrice(250)
            .withQuantity(1)
            .withExpenditureDate(LocalDate.of(2019, 4, 3))
            .createExpenditure();

        this.testClient.post()
            .uri("/expenditures")
            .syncBody(expenditure)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(JsonNode.class)
            .consumeWith(result -> {
                URI location = result.getResponseHeaders().getLocation();
                assertThat(location.toString()).isEqualTo("/expenditures/100");
                JsonNode body = result.getResponseBody();
                assertThat(body).isNotNull();

                assertThat(body.get("expenditureId").asInt()).isEqualTo(100);
                assertThat(body.get("expenditureName").asText()).isEqualTo("ビール");
                assertThat(body.get("unitPrice").asInt()).isEqualTo(250);
                assertThat(body.get("quantity").asInt()).isEqualTo(1);
                // TODO 後で実装します
                //assertThat(body.get("expenditureDate").asText()).isEqualTo("2019-04-03");
            });

        this.testClient.get()
            .uri("/expenditures/100")
            .exchange()
            .expectStatus().isOk()
            .expectBody(JsonNode.class)
            .consumeWith(result -> {
                JsonNode body = result.getResponseBody();
                assertThat(body).isNotNull();

                assertThat(body.get("expenditureId").asInt()).isEqualTo(100);
                assertThat(body.get("expenditureName").asText()).isEqualTo("ビール");
                assertThat(body.get("unitPrice").asInt()).isEqualTo(250);
                assertThat(body.get("quantity").asInt()).isEqualTo(1);
                // TODO 後で実装します
                //assertThat(body.get("expenditureDate").asText()).isEqualTo("2019-04-03");
            });
    }

    // TODO 後で実装します
    //@Test
    void post_400() {
        Expenditure expenditure = new ExpenditureBuilder()
            .withExpenditureId(1000)
            .withExpenditureName("")
            .withUnitPrice(-1)
            .withQuantity(-1)
            .withExpenditureDate(null)
            .createExpenditure();

        this.testClient.post()
            .uri("/expenditures")
            .syncBody(expenditure)
            .exchange()
            .expectStatus().isBadRequest()
            .expectBody(JsonNode.class)
            .consumeWith(result -> {
                JsonNode body = result.getResponseBody();
                assertThat(body).isNotNull();

                assertThat(body.get("status").asInt()).isEqualTo(400);
                assertThat(body.get("error").asText()).isEqualTo("Bad Request");
                assertThat(body.get("details").size()).isEqualTo(5);
                assertThat(body.get("details").get("expenditureId").get(0).asText()).isEqualTo("\"expenditureId\" must be null");
                assertThat(body.get("details").get("expenditureName").get(0).asText()).isEqualTo("\"expenditureName\" must not be empty");
                assertThat(body.get("details").get("unitPrice").get(0).asText()).isEqualTo("\"unitPrice\" must be greater than 0");
                assertThat(body.get("details").get("quantity").get(0).asText()).isEqualTo("\"quantity\" must be greater than 0");
                assertThat(body.get("details").get("expenditureDate").get(0).asText()).isEqualTo("\"expenditureDate\" must not be null");
            });
    }

    @Test
    void delete() {
        this.testClient.delete()
            .uri("/expenditures/1")
            .exchange()
            .expectStatus().isNoContent();

        this.testClient.get()
            .uri("/expenditures/1")
            .exchange()
            .expectStatus().isNotFound();
    }
}
```

TODOを実装しないでテストを実行すると次のように`delete`以外のテストが失敗します。

![image](https://user-images.githubusercontent.com/106908/58398135-e8be2e80-808e-11e9-900d-744ad0961a6a.png)


TODOを実装して、全てのテストが成功したら、`App`クラスの`main`メソッドを実行して、次のリクエストを送り、正しくレスポンスが返ることを確認してください。

```
$ curl localhost:8080/expenditures -d "{\"expenditureName\":\"コーヒー\",\"unitPrice\":300,\"quantity\":1,\"expenditureDate\":[2019,6,3]}" -H "Content-Type: application/json"
{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":[2019,6,3]}
```

```
$ curl localhost:8080/expenditures
[{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":[2019,6,3]}]
```

```
$ curl localhost:8080/expenditures/1
{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":[2019,6,3]}
```

```
$ curl -XDELETE localhost:8080/expenditures/1
```

```
$ curl localhost:8080/expenditures
[]
```

TODOの実装例は次の通りです。

<details>
  <summary><code>ExpenditureHandler</code>の正解例</summary>

```java
package com.example.expenditure;

import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.RouterFunctions;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.util.UriComponentsBuilder;
import reactor.core.publisher.Mono;

import java.util.LinkedHashMap;

import static org.springframework.http.HttpStatus.NOT_FOUND;

public class ExpenditureHandler {

    private final ExpenditureRepository expenditureRepository;

    public ExpenditureHandler(ExpenditureRepository expenditureRepository) {
        this.expenditureRepository = expenditureRepository;
    }

    public RouterFunction<ServerResponse> routes() {
        return RouterFunctions.route()
            .GET("/expenditures", this::list)
            .POST("/expenditures", this::post)
            .GET("/expenditures/{expenditureId}", this::get)
            .DELETE("/expenditures/{expenditureId}", this::delete)
            .build();
    }

    Mono<ServerResponse> list(ServerRequest req) {
        return ServerResponse.ok().body(this.expenditureRepository.findAll(), Expenditure.class);
    }

    Mono<ServerResponse> post(ServerRequest req) {
        return req.bodyToMono(Expenditure.class)
            .flatMap(this.expenditureRepository::save)
            .flatMap(created -> ServerResponse
                .created(UriComponentsBuilder.fromUri(req.uri()).path("/{expenditureId}").build(created.getExpenditureId()))
                .syncBody(created));
    }

    Mono<ServerResponse> get(ServerRequest req) {
        return this.expenditureRepository.findById(Integer.valueOf(req.pathVariable("expenditureId")))
            .flatMap(expenditure -> ServerResponse.ok().syncBody(expenditure))
            .switchIfEmpty(Mono.defer(() -> ServerResponse.status(NOT_FOUND).syncBody(new LinkedHashMap<String, Object>() {

                {
                    put("status", 404);
                    put("error", "Not Found");
                    put("message", "The given expenditure is not found.");
                }
            })));
    }

    Mono<ServerResponse> delete(ServerRequest req) {
        return ServerResponse.noContent()
            .build(this.expenditureRepository.deleteById(Integer.valueOf(req.pathVariable("expenditureId"))));
    }
}
```

</details>

### 日付フィールドの変更

`expenditureDate`が`[year, month, day]`のリスト形式で返却されているので、`"year-month-day"`の文字列形式に変更しましょう。

`App.java`に以下のメソッドを追加して、

```java
    public static HandlerStrategies handlerStrategies() {
        return HandlerStrategies.empty()
            .codecs(configure -> {
                configure.registerDefaults(true);
                ServerCodecConfigurer.ServerDefaultCodecs defaults = configure
                    .defaultCodecs();
                ObjectMapper objectMapper = Jackson2ObjectMapperBuilder.json()
                    .dateFormat(new StdDateFormat())
                    .build();
                defaults.jackson2JsonEncoder(new Jackson2JsonEncoder(objectMapper));
                defaults.jackson2JsonDecoder(new Jackson2JsonDecoder(objectMapper));
            })
            .build();
    }
```

`main`メソッド中の次のコード

```java
        HttpHandler httpHandler = RouterFunctions.toHttpHandler(App.routes(),
            HandlerStrategies.builder().build());
```
を
```java
        HttpHandler httpHandler = RouterFunctions.toHttpHandler(App.routes(),
            App.handlerStrategies());
```
に変更してください。


これに合わせて`ExpenditureHandlerTest`の`before`メソッドも以下のように修正してください。

```java
    @BeforeAll
    void before() {
        this.testClient = WebTestClient.bindToRouterFunction(this.expenditureHandler.routes())
            .handlerStrategies(App.handlerStrategies()) // 追加
            .build();
    }
```


`ExpenditureHandlerTest`コード内のTODO部分、次のような箇所に対して

```java
                // TODO 後で実装します
                //assertThat(body.get(0).get("expenditureDate").asText()).isEqualTo("2019-04-01");
```

次のようにコメントを削除してください。

```java
                assertThat(body.get(0).get("expenditureDate").asText()).isEqualTo("2019-04-01");
```

5箇所あります。

変更後の全てのテストが成功したら、`App`クラスの`main`メソッドを実行して、次のリクエストを送り、正しくレスポンスが返ることを確認してください。

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

### Cloud Foundryへのデプロイ

作成したアプリケーションをCloud Foundry([Pivotal Web Services](https://run.pivotal.io))にデプロイします。

`cf login`でログインします。

```
cf login -a api.run.pivotal.io
```

プロジェクト直下で`manifest.yml`を作成し、次の内容を記述してください。

```yaml
applications:
- name: moneyger
  path: target/moneyger-1.0.0-SNAPSHOT.jar
  memory: 128m
  env:
    JAVA_OPTS: '-XX:ReservedCodeCacheSize=22M -XX:MaxDirectMemorySize=22M -XX:MaxMetaspaceSize=54M -Xss512K'
    JBP_CONFIG_OPEN_JDK_JRE: '[memory_calculator: {stack_threads: 30}]'
```

> `manifest.yml`はかなり小さなメモリを使用するようにチューニングされています。コンテナのメモリサイズは128MBで、そのうち
> ReservedCodeCacheSizeが22MB、MaxDirectMemorySizeが22MB、MaxMetaspaceSizeが54MB、スレッドスタックが512KB * 30 = 15MB
> 残る15MBがヒープサイズに使用されます。
> このメモリサイズをローカル環境で実行する場合は次のように実行してください。
>
> ```sh
> JAVA_OPTS="-Xmx15m -XX:ReservedCodeCacheSize=22M -XX:MaxDirectMemorySize=22M -XX:MaxMetaspaceSize=54M -Xss512K"
> java $JAVA_OPTS -jar target/moneyger-1.0.0-SNAPSHOT.jar
> ```

ビルドして、`cf push`コマンドでデプロイします。

```
$ ./mvnw clean package -DskipTests=true
$ cf push --random-route
```

次のようなログが出力されてデプロイが完了します。

```
demo@example.com としてマニフェストから組織 APJ / スペース production にプッシュしています...
マニフェスト・ファイル /tmp/moneyger/manifest.yml を使用しています
アプリ情報を取得しています...
これらの属性でアプリを作成しています...
+ 名前:       moneyger
  パス:       /private/tmp/moneyger/target/moneyger-1.0.0-SNAPSHOT.jar
+ メモリー:   128M
  環境:
+   JAVA_OPTS
+   JBP_CONFIG_OPEN_JDK_JRE
  経路:
+   moneyger-chatty-zebra.cfapps.io

アプリ moneyger を作成しています...
経路をマップしています...
ローカル・ファイルをリモート・キャッシュと比較しています...
Packaging files to upload...
ファイルをアップロードしています...
 248.00 KiB / 248.00 KiB [=======================================================================================================================================================================================================================================] 100.00% 1s

API がファイルの処理を完了するのを待機しています...

アプリをステージングし、ログをトレースしています...
   Downloading dotnet_core_buildpack_beta...
   Downloading staticfile_buildpack...
   Downloading nodejs_buildpack...
   Downloading dotnet_core_buildpack...
   Downloading java_buildpack...
   Downloaded staticfile_buildpack
   Downloading ruby_buildpack...
   Downloaded nodejs_buildpack
   Downloading php_buildpack...
   Downloaded dotnet_core_buildpack_beta
   Downloading go_buildpack...
   Downloaded dotnet_core_buildpack
   Downloading python_buildpack...
   Downloaded java_buildpack
   Downloading binary_buildpack...
   Downloaded ruby_buildpack
   Downloaded php_buildpack
   Downloaded go_buildpack
   Downloaded python_buildpack
   Downloaded binary_buildpack
   Cell f237c36d-0a62-4579-a51b-b11ee1d58145 creating container for instance b1036536-3a31-4d47-8039-83952134de33
   Cell f237c36d-0a62-4579-a51b-b11ee1d58145 successfully created container for instance b1036536-3a31-4d47-8039-83952134de33
   Downloading app package...
   Downloaded app package (12.4M)
   -----> Java Buildpack v4.19 (offline) | https://github.com/cloudfoundry/java-buildpack.git#3f4eee2
   -----> Downloading Jvmkill Agent 1.16.0_RELEASE from https://java-buildpack.cloudfoundry.org/jvmkill/bionic/x86_64/jvmkill-1.16.0_RELEASE.so (found in cache)
   -----> Downloading Open Jdk JRE 1.8.0_202 from https://java-buildpack.cloudfoundry.org/openjdk/bionic/x86_64/openjdk-jre-1.8.0_202-bionic.tar.gz (found in cache)
          Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.8s)
          JVM DNS caching disabled in lieu of BOSH DNS caching
   -----> Downloading Open JDK Like Memory Calculator 3.13.0_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/bionic/x86_64/memory-calculator-3.13.0_RELEASE.tar.gz (found in cache)
          Loaded Classes: 11266, Threads: 30
   -----> Downloading Client Certificate Mapper 1.8.0_RELEASE from https://java-buildpack.cloudfoundry.org/client-certificate-mapper/client-certificate-mapper-1.8.0_RELEASE.jar (found in cache)
   -----> Downloading Container Security Provider 1.16.0_RELEASE from https://java-buildpack.cloudfoundry.org/container-security-provider/container-security-provider-1.16.0_RELEASE.jar (found in cache)
   -----> Downloading Spring Auto Reconfiguration 2.7.0_RELEASE from https://java-buildpack.cloudfoundry.org/auto-reconfiguration/auto-reconfiguration-2.7.0_RELEASE.jar (found in cache)
   Exit status 0
   Uploading droplet, build artifacts cache...
   Uploading droplet...
   Uploading build artifacts cache...
   Uploaded build artifacts cache (128B)
   Uploaded droplet (55.9M)
   Uploading complete
   Cell f237c36d-0a62-4579-a51b-b11ee1d58145 stopping instance b1036536-3a31-4d47-8039-83952134de33
   Cell f237c36d-0a62-4579-a51b-b11ee1d58145 destroying container for instance b1036536-3a31-4d47-8039-83952134de33

アプリが開始するのを待機しています...

名前:                   moneyger
要求された状態:         started
経路:                   moneyger-chatty-zebra.cfapps.io
最終アップロード日時:   Mon 27 May 16:16:10 JST 2019
スタック:               cflinuxfs3
ビルドパック:           client-certificate-mapper=1.8.0_RELEASE container-security-provider=1.16.0_RELEASE java-buildpack=v4.19-offline-https://github.com/cloudfoundry/java-buildpack.git#3f4eee2 java-main java-opts java-security jvmkill-agent=1.16.0_RELEASE
                        open-jdk-...

タイプ:           web
インスタンス:     1/1
メモリー使用量:   128M
開始コマンド:     JAVA_OPTS="-agentpath:$PWD/.java-buildpack/open_jdk_jre/bin/jvmkill-1.16.0_RELEASE=printHeapHistogram=1 -Djava.io.tmpdir=$TMPDIR -XX:ActiveProcessorCount=$(nproc)
                  -Djava.ext.dirs=$PWD/.java-buildpack/container_security_provider:$PWD/.java-buildpack/open_jdk_jre/lib/ext -Djava.security.properties=$PWD/.java-buildpack/java_security/java.security $JAVA_OPTS" &&
                  CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-3.13.0_RELEASE -totMemory=$MEMORY_LIMIT -loadedClasses=12567 -poolType=metaspace -stackThreads=30 -vmOptions="$JAVA_OPTS") && echo JVM Memory Configuration:
                  $CALCULATED_MEMORY && JAVA_OPTS="$JAVA_OPTS $CALCULATED_MEMORY" && MALLOC_ARENA_MAX=2 SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher
     状態   開始日時               cpu    メモリー            ディスク         詳細
#0   実行   2019-05-27T07:16:32Z   0.0%   128M の中の 12.3M   1G の中の 124M  
```

アプリケーションのURLは`https://moneyger-<ランダムな文字列>.cfapps.io`です。

次のリクエストを送り、正しくレスポンスが返ることを確認してください。

```
$ curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures -d "{\"expenditureName\":\"コーヒー\",\"unitPrice\":300,\"quantity\":1,\"expenditureDate\":\"2019-06-03\"}" -H "Content-Type: application/json"
{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":"2019-06-03"}
```

```
$ curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures
[{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":"2019-06-03"}]
```

```
$ curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures/1
{"expenditureId":1,"expenditureName":"コーヒー","unitPrice":300,"quantity":1,"expenditureDate":"2019-06-03"}
```

```
$ curl -XDELETE https://moneyger-<CHANGE ME>.cfapps.io/expenditures/1
```

```
$ curl https://moneyger-<CHANGE ME>.cfapps.io/expenditures
[]
```

> Pivotal Web Servicesはメモリ課金であり、128MB使用の場合にかかる費用は常時起動した状態で、$2.70 (約300円) /月です。

### 補足

`routes()`の定義は次のようにネストして記述することもできます。

```java
    public RouterFunction<ServerResponse> routes() {
        return RouterFunctions.route()
            .path("/expenditures", b -> b
                .GET("/", this::list)
                .POST("/", this::post)
                .GET("/{expenditureId}", this::get)
                .DELETE("/{expenditureId}", this::delete))
            .build();
    }
```
