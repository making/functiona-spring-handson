## 収入APIの実装

Web UIにはIncome(収入) API用の画面も含まれています。Expenditure(支出) APIを参考にして、Income APIも実装して見てください。

一部のコードを下記に示します。

### Incomeモデルの作成


```java
package com.example.income;

import am.ik.yavi.builder.ValidatorBuilder;
import am.ik.yavi.core.ConstraintViolations;
import am.ik.yavi.core.Validator;
import am.ik.yavi.fn.Either;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;

import java.time.LocalDate;

@JsonDeserialize(builder = IncomeBuilder.class)
public class Income {

    private final Integer incomeId;

    private final String incomeName;

    private final int amount;

    private final LocalDate incomeDate;

    private static final Validator<Income> validator = ValidatorBuilder.of(Income.class)
        .constraint(Income::getIncomeId, "incomeId", c -> c.isNull())
        .constraint(Income::getIncomeName, "incomeName", c -> c.notEmpty().lessThanOrEqual(255))
        .constraint(Income::getAmount, "amount", c -> c.greaterThan(0))
        .constraintOnObject(Income::getIncomeDate, "incomeDate", c -> c.notNull())
        .build();

    public Income(Integer incomeId, String incomeName, int amount, LocalDate incomeDate) {
        this.incomeId = incomeId;
        this.incomeName = incomeName;
        this.amount = amount;
        this.incomeDate = incomeDate;
    }

    public Integer getIncomeId() {
        return incomeId;
    }

    public String getIncomeName() {
        return incomeName;
    }

    public int getAmount() {
        return amount;
    }

    public LocalDate getIncomeDate() {
        return incomeDate;
    }

    public Either<ConstraintViolations, Income> validate() {
        return validator.validateToEither(this);
    }

    @Override
    public String toString() {
        return "Income{" +
            "incomeId=" + incomeId +
            ", incomeName='" + incomeName + '\'' +
            ", amount=" + amount +
            ", incomeDate=" + incomeDate +
            '}';
    }
}
```

```java
package com.example.income;

import com.fasterxml.jackson.databind.annotation.JsonPOJOBuilder;

import java.time.LocalDate;

@JsonPOJOBuilder(buildMethodName = "createIncome")
public class IncomeBuilder {

    private int amount;

    private LocalDate incomeDate;

    private Integer incomeId;

    private String incomeName;

    public IncomeBuilder() {
    }

    public IncomeBuilder(Income income) {
        this.amount = income.getAmount();
        this.incomeDate = income.getIncomeDate();
        this.incomeId = income.getIncomeId();
        this.incomeName = income.getIncomeName();
    }

    public Income createIncome() {
        return new Income(incomeId, incomeName, amount, incomeDate);
    }

    public IncomeBuilder withAmount(int amount) {
        this.amount = amount;
        return this;
    }

    public IncomeBuilder withIncomeDate(LocalDate incomeDate) {
        this.incomeDate = incomeDate;
        return this;
    }

    public IncomeBuilder withIncomeId(Integer incomeId) {
        this.incomeId = incomeId;
        return this;
    }

    public IncomeBuilder withIncomeName(String incomeName) {
        this.incomeName = incomeName;
        return this;
    }
}
```

### データベースの作成


```java
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