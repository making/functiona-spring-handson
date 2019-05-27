## 例外ハンドリングの改善


```
$ curl localhost:8080/expenditures -d "{\"unitPrice\":\"foo\"}" -H "Content-Type: application/json" -v
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> POST /expenditures HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 19
> 
* upload completely sent off: 19 out of 19 bytes
< HTTP/1.1 500 Internal Server Error
< content-length: 0
< 
* Connection #0 to host localhost left intact
```

### `ErrorResponseExceptionHandler`の作成


```java
package com.example.error;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.codec.EncoderHttpMessageWriter;
import org.springframework.http.codec.HttpMessageWriter;
import org.springframework.http.codec.json.Jackson2JsonEncoder;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.reactive.result.view.ViewResolver;
import org.springframework.web.server.ResponseStatusException;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebExceptionHandler;
import reactor.core.publisher.Mono;

import java.util.Collections;
import java.util.List;

import static org.springframework.http.HttpStatus.INTERNAL_SERVER_ERROR;
import static org.springframework.http.MediaType.APPLICATION_JSON;

public class ErrorResponseExceptionHandler implements WebExceptionHandler {

    private final Logger log = LoggerFactory.getLogger(ErrorResponseExceptionHandler.class);

    private ServerResponse.Context context = new ServerResponse.Context() {

        private List<HttpMessageWriter<?>> messageWriters = Collections.singletonList(new EncoderHttpMessageWriter<>(new Jackson2JsonEncoder()));

        @Override
        public List<HttpMessageWriter<?>> messageWriters() {
            return this.messageWriters;
        }

        @Override
        public List<ViewResolver> viewResolvers() {
            return Collections.emptyList();
        }
    };

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        log.warn(ex.getMessage(), ex);
        final HttpStatus status = (ex instanceof ResponseStatusException) ? ((ResponseStatusException) ex).getStatus() : INTERNAL_SERVER_ERROR;
        return ServerResponse.status(status)
            .contentType(APPLICATION_JSON)
            .syncBody(new ErrorResponseBuilder()
                .withStatus(status)
                .withMessage(ex.getMessage())
                .createErrorResponse())
            .flatMap(response -> response.writeTo(exchange, this.context));
    }
}
```

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
            // ↓追加
            .exceptionHandler(new ErrorResponseExceptionHandler())
            .build();
    }
```

```
$ curl localhost:8080/expenditures -d "{\"unitPrice\":\"foo\"}" -H "Content-Type: application/json" -v
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> POST /expenditures HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 19
> 
* upload completely sent off: 19 out of 19 bytes
< HTTP/1.1 400 Bad Request
< Content-Type: application/json
< Content-Length: 598
< 
* Connection #0 to host localhost left intact
{"status":400,"error":"Bad Request","message":"400 BAD_REQUEST \"Failed to read HTTP message\"; nested exception is org.springframework.core.codec.DecodingException: JSON decoding error: Cannot deserialize value of type `int` from String \"foo\": not a valid Integer value; nested exception is com.fasterxml.jackson.databind.exc.InvalidFormatException: Cannot deserialize value of type `int` from String \"foo\": not a valid Integer value\n at [Source: (io.netty.buffer.ByteBufInputStream); line: 1, column: 14] (through reference chain: com.example.expenditure.ExpenditureBuilder[\"unitPrice\"])"}
```
