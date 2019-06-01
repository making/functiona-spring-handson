## ウォームアップ

まずはSpring Bootを使ったSpring WebFlux.fnのHello World!アプリケーションを作成します。


以下のコマンドを実行すると、hello-sourceフォルダに雛形プロジェクトが生成されます。Windowsの場合はGit BashなどのBash実行環境を利用してください。

```sh
curl https://start.spring.io/starter.tgz \
       -d artifactId=hello-fn \
       -d baseDir=hello-fn \
       -d packageName=com.example \
       -d dependencies=webflux \
       -d applicationName=HelloFnApplication | tar -xzvf -
```

`hello-fn/src/main/java/com/example/HelloFnApplication.java`を次のように修正してください。

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
// ↓ここから追加
import org.springframework.context.annotation.Bean;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.RouterFunctions;
import org.springframework.web.reactive.function.server.ServerResponse;
// ↑ここまで

@SpringBootApplication
public class HelloFnApplication {

    // ↓ここから追加
    @Bean
    public RouterFunction<ServerResponse> routes() {
        return RouterFunctions.route()
            .GET("/", req -> ServerResponse.ok().syncBody("Hello World!"))
            .build();
    }
    // ↑ここまで

    public static void main(String[] args) {
        SpringApplication.run(HelloFnApplication.class, args);
    }

}
```

ビルドして実行してください。

```
cd hello-fn
./mvnw clean package -DskipTests=true
java -jar target/hello-fn-0.0.1-SNAPSHOT.jar
```


```sh
$ curl localhost:8080
Hello World!
```

このあとはSpring Bootを使わない方法でアプリケーションを構築します。