## Web UIの追加


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

```xml
        <dependency>
            <groupId>com.github.making</groupId>
            <artifactId>moneyger-ui</artifactId>
            <version>master-SNAPSHOT</version>
        </dependency>
```

```java
    static RouterFunction<ServerResponse> staticRoutes() {
        return RouterFunctions.route()
            .GET("/", req -> ServerResponse.ok().syncBody(new ClassPathResource("META-INF/resources/index.html")))
            .resources("/**", new ClassPathResource("META-INF/resources/"))
            .build();
    }
```

```java
    static RouterFunction<ServerResponse> routes() {
        // ...

        return new ExpenditureHandler(new R2dbcExpenditureRepository(databaseClient, transactionalOperator)).routes();
    }
```

↓

```java
    static RouterFunction<ServerResponse> routes() {
        // ...

        return staticRoutes()
            .and(new ExpenditureHandler(new R2dbcExpenditureRepository(databaseClient, transactionalOperator)).routes());
    }
```


![image](https://user-images.githubusercontent.com/106908/58406424-8b34dc80-80a4-11e9-932d-1bcfd032a2f6.png)

![image](https://user-images.githubusercontent.com/106908/58406492-ad2e5f00-80a4-11e9-85c4-6a9452dd4589.png)


### Cloud Foundryにデプロイ

```
./mvnw clean package
cf push
```

![image](https://user-images.githubusercontent.com/106908/58406552-d2bb6880-80a4-11e9-8edf-e22d6015ebef.png)

