## YAVIによるValidationの実装

次はValidationを実装します。今回はBean Validationではなく、Spring WebFlux.fnによりFitした使い方ができる[YAVI](https://github.com/making/yavi)を使用します。


**TODO**部分を実装してください。動作を確認するためのテストコードは以下に続きます。TODOを実装する前にテストを実行してくだい。

* [参考資料](https://github.com/making/yavi)

```java
package com.example.expenditure;

import am.ik.yavi.builder.ValidatorBuilder;
import am.ik.yavi.core.ConstraintViolations;
import am.ik.yavi.core.Validator;
import am.ik.yavi.fn.Either;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;

import java.time.LocalDate;

@JsonDeserialize(builder = ExpenditureBuilder.class)
public class Expenditure {

    private final Integer expenditureId;

    private final String expenditureName;

    private final int unitPrice;

    private final int quantity;

    private final LocalDate expenditureDate;

    // 追加
    private static Validator<Expenditure> validator = ValidatorBuilder.of(Expenditure.class)
        .constraint(Expenditure::getExpenditureId, "expenditureId", c -> c.isNull())
        .constraint(Expenditure::getExpenditureName, "expenditureName", c -> c.notEmpty().lessThan(255))
        // TODO
        // "expenditureName"は空ではなく、文字数は255以下
        // "unitPrice"は0より大きい
        // "quantity"は0より大きい
        // .constraint(...)
        .constraintOnObject(Expenditure::getExpenditureDate, "expenditureDate", c -> c.notNull())
        .build();

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


    // 追加
    public Either<ConstraintViolations, Expenditure> validate() {
        return validator.validateToEither(this);
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

次にエラーレスポンス用のJavaクラスを作成します。
`com.example.error`パッケージを作って`ErrorResponse.java`を作成してください。


```java
package com.example.error;

import com.fasterxml.jackson.annotation.JsonInclude;

import java.util.List;
import java.util.Map;

public class ErrorResponse {

    private final int status;

    private final String error;

    @JsonInclude(JsonInclude.Include.NON_EMPTY)
    private final String message;

    @JsonInclude(JsonInclude.Include.NON_EMPTY)
    private final Map<String, List<String>> details;

    public ErrorResponse(int status, String error, String message, Map<String, List<String>> details) {
        this.status = status;
        this.error = error;
        this.message = message;
        this.details = details;
    }

    public int getStatus() {
        return status;
    }

    public String getError() {
        return error;
    }

    public String getMessage() {
        return message;
    }

    public Map<String, List<String>> getDetails() {
        return details;
    }
}
```

続いて`ErrorResponseBuilder.java`を作成してください。

```java
package com.example.error;

import am.ik.yavi.core.ConstraintViolations;
import org.springframework.http.HttpStatus;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;

import java.util.Collections;
import java.util.List;
import java.util.Map;

public class ErrorResponseBuilder {

    private Map<String, List<String>> details;

    private String error;

    private String message;

    private int status;

    public ErrorResponse createErrorResponse() {
        return new ErrorResponse(status, error, message, details);
    }

    public ErrorResponseBuilder withDetails(Map<String, List<String>> details) {
        this.details = details;
        return this;
    }

    public ErrorResponseBuilder withDetails(ConstraintViolations violations) {
        MultiValueMap<String, String> details = new LinkedMultiValueMap<>();
        violations.details().forEach(d -> details.add((String) d.getArgs()[0], d.getDefaultMessage()));
        this.details = Collections.unmodifiableMap(details);
        return this;
    }

    public ErrorResponseBuilder withMessage(String message) {
        this.message = message;
        return this;
    }

    public ErrorResponseBuilder withStatus(HttpStatus status) {
        this.status = status.value();
        this.error = status.getReasonPhrase();
        return this;
    }
}
```

`ExpenditureHandler`の`get`メソッドの次の部分(`LinkedHashMap`でエラーメッセージを作成している箇所)を、

```java
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
```

次のように`ErrorResponse`で置き換えてください。

```java
    Mono<ServerResponse> get(ServerRequest req) {
        return this.expenditureRepository.findById(Integer.valueOf(req.pathVariable("expenditureId")))
            .flatMap(expenditure -> ServerResponse.ok().syncBody(expenditure))
            .switchIfEmpty(Mono.defer(() -> ServerResponse.status(NOT_FOUND)
                .syncBody(new ErrorResponseBuilder()
                    .withMessage("The given expenditure is not found.")
                    .withStatus(NOT_FOUND)
                    .createErrorResponse())));
    }
```

また次の`post`メソッドにValidationを追加します。

```java
    Mono<ServerResponse> post(ServerRequest req) {
        return req.bodyToMono(Expenditure.class)
            .flatMap(this.expenditureRepository::save)
            .flatMap(created -> ServerResponse
                .created(UriComponentsBuilder.fromUri(req.uri()).path("/{expenditureId}").build(created.getExpenditureId()))
                .syncBody(created));
    }
```

`post`メソッドを次のように変更してください。

```java
    Mono<ServerResponse> post(ServerRequest req) {
        return req.bodyToMono(Expenditure.class)
            .flatMap(expenditure -> expenditure.validate()
                .bimap(v -> new ErrorResponseBuilder().withStatus(BAD_REQUEST).withDetails(v).createErrorResponse(), this.expenditureRepository::save)
                .fold(error -> ServerResponse.badRequest().syncBody(error),
                    result -> result.flatMap(created -> ServerResponse
                        .created(UriComponentsBuilder.fromUri(req.uri()).path("/{expenditureId}").build(created.getExpenditureId()))
                        .syncBody(created))));
    }
```

`ExpenditureHandlerTest`の`post_400`メソッドに付いているコメントを、

```java
    // TODO 後で実装します
    // @Test
    void post_400() {
      // ...
    }
```

次のように削除してください。

```java
    @Test
    void post_400() {
      // ...
    }
```

TODOを実装しないでテストを実行すると次のように`post_400`のテストが失敗します。

![image](https://user-images.githubusercontent.com/106908/58399914-9f70dd80-8094-11e9-8c90-83a3cff22aae.png)


```
java.lang.AssertionError: 
Expecting:
 <2>
to be equal to:
 <5>
but was not.

> POST /expenditures
> Content-Length: [95]
> Content-Type: [application/json]
> WebTestClient-Request-Id: [8]

{"expenditureId":1000,"expenditureName":"","unitPrice":-1,"quantity":-1,"expenditureDate":null}

< 400 BAD_REQUEST Bad Request
< Content-Type: [application/json]
< Content-Length: [158]

{"status":400,"error":"Bad Request","details":{"expenditureId":["\"expenditureId\" must be null"],"expenditureDate":["\"expenditureDate\" must not be null"]}}
```

`Expenditure`クラスのTODOを実装して、テストが通ることを確認してください。

TODOの実装例は次の通りです。

<details>
  <summary><code>Expenditure</code>の正解例</summary>

```java
package com.example.expenditure;

import am.ik.yavi.builder.ValidatorBuilder;
import am.ik.yavi.core.ConstraintViolations;
import am.ik.yavi.core.Validator;
import am.ik.yavi.fn.Either;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;

import java.time.LocalDate;

@JsonDeserialize(builder = ExpenditureBuilder.class)
public class Expenditure {

    private final Integer expenditureId;

    private final String expenditureName;

    private final int unitPrice;

    private final int quantity;

    private final LocalDate expenditureDate;

    private static Validator<Expenditure> validator = ValidatorBuilder.of(Expenditure.class)
        .constraint(Expenditure::getExpenditureId, "expenditureId", c -> c.isNull())
        .constraint(Expenditure::getExpenditureName, "expenditureName", c -> c.notEmpty().lessThanOrEqual(255))
        .constraint(Expenditure::getUnitPrice, "unitPrice", c -> c.greaterThan(0))
        .constraint(Expenditure::getQuantity, "quantity", c -> c.greaterThan(0))
        .constraintOnObject(Expenditure::getExpenditureDate, "expenditureDate", c -> c.notNull())
        .build();

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


    public Either<ConstraintViolations, Expenditure> validate() {
        return validator.validateToEither(this);
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

</details>

```
$ curl localhost:8080/expenditures -d "{\"expenditureId\":1000,\"expenditureName\":\"\",\"unitPrice\":-1,\"quantity\":-1,\"expenditureDate\":null}" -H "Content-Type: application/json"
{"status":400,"error":"Bad Request","details":{"expenditureId":["\"expenditureId\" must be null"],"expenditureName":["\"expenditureName\" must not be empty"],"unitPrice":["\"unitPrice\" must be greater than 0"],"quantity":["\"quantity\" must be greater than 0"],"expenditureDate":["\"expenditureDate\" must not be null"]}}
```
