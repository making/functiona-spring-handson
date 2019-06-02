## Web UIの追加

APIに対するWeb UIを追加します。

`pom.xml`に次の`repository`を追加してください。

```xml
    <repositories>
    	<!-- 追加 -->
        <repository>
            <id>jitpack.io</id>
            <url>https://jitpack.io</url>
        </repository>
        <!-- ... -->
    </repositories>
```

`pom.xml`に次の`dependency`を追加してください。

```xml
        <dependency>
            <groupId>com.github.making</groupId>
            <artifactId>moneyger-ui</artifactId>
            <version>master-SNAPSHOT</version>
        </dependency>
```

`App.java`に次のメソッドを追加してください。

```java
    static RouterFunction<ServerResponse> staticRoutes() {
        return RouterFunctions.route()
            .GET("/", req -> ServerResponse.ok().syncBody(new ClassPathResource("META-INF/resources/index.html")))
            .resources("/**", new ClassPathResource("META-INF/resources/"))
            .filter((request, next) -> next.handle(request)
                .flatMap(response -> ServerResponse.from(response)
                    .cacheControl(CacheControl.maxAge(Duration.ofDays(3)))
                    .build(response::writeTo)))
            .build();
    }
```

`routes`メソッドを次の箇所を

```java
    static RouterFunction<ServerResponse> routes() {
        // ...

        return new ExpenditureHandler(new R2dbcExpenditureRepository(databaseClient, transactionalOperator)).routes();
    }
```

次のように変更してください。

```java
    static RouterFunction<ServerResponse> routes() {
        // ...

        return staticRoutes()
            .and(new ExpenditureHandler(new R2dbcExpenditureRepository(databaseClient, transactionalOperator)).routes());
    }
```

`App`クラスの`main`メソッドを実行して、[http://localhost:8080](http://localhost:8080)にアクセスしてください。

![image](https://user-images.githubusercontent.com/106908/58406424-8b34dc80-80a4-11e9-932d-1bcfd032a2f6.png)

![image](https://user-images.githubusercontent.com/106908/58406492-ad2e5f00-80a4-11e9-85c4-6a9452dd4589.png)


### Cloud Foundryにデプロイ

ビルドして`cf push`してください。

```
./mvnw clean package -DskipTests=true
cf push
```

![image](https://user-images.githubusercontent.com/106908/58406552-d2bb6880-80a4-11e9-8edf-e22d6015ebef.png)

---

興味があれば自分の好みのUIを実装してください。
