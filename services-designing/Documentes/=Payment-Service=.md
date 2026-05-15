# **Payment Service**
### Задачи:
**Работа с покупателем** *(только для роли `USER`)*
- Рассчитывает и формирует стоимость доставки товара *(функция "узнать стоимость")*;
- Выставляет счет на оплату;
- Работа с банковской системой (*принимает инфу об **успешной оплате** или **отказе***);
~~- Может получать из внешних источников курсы валют для расчета стоимости доставки;~~
### Стек:
- Java 17
- Spring Boot 3.5;
- Spring Data JPA;
- PostgreSQL;
- REST (Feign);
- Maven;
- Lombok;
- Swagger;
- JUnit / Mockito;
### Сценарии работы мипкросервиса:
1. Покупатель запрашивает стоимость товара. `Payment Service` рассчитывает по формуле и возвращает просто число (сумму);
2. Если стоимость товара устроила, то покупатель запрашивает счет на оплату. `Payment Service` **формирует полноценный счет** на оплату, ходит в `User Service` за данными по пользователю *(для счета)*. Все сохраняет в своей БД;
   *(При этом `Delivery Service` позаботится о подборе/ бронировании транспорта/ водителя и бронировании самого товара на срок, указанные в счете на оплату.)*
3. Если крайний срок оплаты, указанный в счете **истек**, то `Payment Service` аннулирует счет 
   *(При этом `Delivery Service` позаботится о разбронировании/ освобождении транспорта/ водителя и самого товара соответственно.)*
4. Как только покупатель **оплатил** выставленный счет - Банк входит в систему с ролью `BANK` и присылает *DTO* `BankConfirmationRequest`, в котором содержится уникальный `id` *(обязательно)* выставленного счета (тип `UUID`), `сумма по счету` *(обязательно)* и `статус оплаты` *(обязательно)*, `PAID` или `CANCELLED` *(успешно оплачено или отмена оплаты)*. Так-же может присутствовать поле `ФИО плательщика` *(не обязательно)* и поле `дата/время оплаты` *(не обязательно)* . Если поле `ФИО плательщика` = `null` -то система сходит в `User Service` за данными, если `дата оплаты` = `null` -то система пропишет текущую дату/ время *(не все банки сообщают ФИО плательщика и дату/время оплаты)*;
5. В ответ `Payment Service` отправляет *DTO* `BankConfirmationResponse`, в котором дублирует принятые/ дополненные данные. Ответ высылается только после того, как принятые данные лягут в БД сервиса;
6. `Payment Service` уведомляет по средствам сообщения в в **Kafka** (topic: `payment.payment-status`, статус: `COMPLETED` или `CANSELED`) `Delivery Service` о изменении оплаты счета (успех или отмена);
7. `Payment Service` может ходить в банк за данными по курсам валют для расчетов.

---
### **Платежи** (Paymants):

| Метод | Эндпоинт                     | Описание (кратко)                               | Бизнес-правила                                                                    |
| ----- | ---------------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------- |
| GET   | `/api/v1/payments/{id}/cost` | Рассчитать стоимость *(просто узнать цену)*     | с ролью SERVICE `Базовая_ставка` + `вес*10` + `расстояние*5`                      |
| POST  | `/api/v1/payments/bill`      | Сформировать и предоставить счет на оплату      | синхронно идет за расчетами в `Payment Service`  с ролью `SERVICE`                |
| POST  | `/api/v1/payments/callback`  | успех/ отмену оплаты счета, возвращает сам банк | Банк высылает на указанный эндпоинт DTO со статусом по оплате/ Только роль `BANK` |

### Формирование счета:
1. Исходя из `Базоваой_ставки`, `веса` груза и `расстояния` формируется стоимость доставки `cost`;
   стоимость = `базовая_ставка` + (`вес` `*` `коэффициент_веса`) + (`расстояние` `*` `коэффициент_расстояния`)
2. Вычисленная величина `cost` включается в сущность `bill` и записывается в БД. Счету присваивается уникальный `id` с типом `UUID`;
3. Клиенту возвращается счет для оплаты `bill`, который включает `сумму` для оплаты,  `id` платежа и срок действия счета `durationDays` (до какого оплатить, включительно);

---
### Сущность **bill** - (БД & Entity) // Таблица `payment_bill`

| имя переменной | тип                            | Пояснение                                                                                |
|----------------|--------------------------------|------------------------------------------------------------------------------------------|
| `id`           | `UUID`                         | Уникальный идентификатор счета                                                           |
| `clientId`     | `Long` / `Long`                | `id` залогиненного пользователя, кот. формирует счет.                                    |
| `externalId`   | `String` / `VARCHAR(500)`      | Номер транзакции (сокр.) из _API_ банка. полное именнование: `external_transaction_id`   |
| `bankId`       | `Long` / `Long`                | `id` сообщения от Банка об оплате/ анулировании счета.                                   |
| `description`  | `String` / `VARCHAR(500)`      | Описание услуги/ товара                                                                  |
| `amount`       | `BigDecimal` / `NUMERIC(19,2)` | Стоимость в рублях <br>(2 знака после запятой)                                           |
| `status`       | `Enum BillStatus` / `Enum`     | Состояние счета на текущий момент <br>(enum: `ACTIVE`, `PAID`, `VOIDED`, `EXPIRED`)      |
| `createdAt`    | `Instant` / `TIMESTAMPTZ`      | Дата (часы/ минуты) **выставления** счета <br>(тип в коде/ тип в БД)                     |
| `dueDate`      | `Instant` / `TIMESTAMPTZ`      | Крайняя дата/ срок оплаты счета (часы/ минуты), включительно <br>(тип в коде/ тип в БД)  |
| `statusAt`     | `Instant` / `TIMESTAMPTZ`      | Дата (часы/ минуты) последнего изменения статуса по данному счету _(Универсальное поле)_ |

> **Важные моменты по `Entity`:**
> - `Enum` как `String`: Обязательно `@Enumerated(EnumType.STRING)`. 
> ~~_По умолчанию Hibernate ставит ORDINAL (индексы 0, 1, 2), что превратит БД в "кашу", если вы решите добавить новый статус в середину Enum._~~
> - Индекс на `clientId`, `status` и `dueDate`
> - Валидация на уровне БД (Constraints) `amount` > 0

```java
@Entity
@Table(name = "payment_bill", indexes = {
    @Index(name = "idx_bill_client", columnList = "client_id"),
    @Index(name = "idx_bill_status_due", columnList = "status, due_date")
})
@Getter @Setter // Lombok
public class Bill {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id; // Уникальный идентификатор счета

    @Column(name = "client_id", nullable = false)
    private Long clientId; //`id` залогиненного пользователя, кот. формирует счет.

    @Column(name = "external_transaction_id", length = 500)
    private String externalId; //  Номер транзакции (сокр.) из _API_ банка.

    @Column(name = "bank_id")
    private Long bankId; // `id` сообщения от Банка об оплате/ анулировании счета.

    @Column(length = 500)
    private String description;// Описание услуги/ товара

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal amount; // Стоимость в рублях (2 знака после запятой)

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private BillStatus status = BillStatus.ACTIVE;  // Статус счета на текущий момент

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt = Instant.now(); // Дата выставления счета

    @Column(name = "due_date", nullable = false)
    private Instant dueDate; // Крайняя дата/ срок оплаты счета, включительно

    @Column(name = "status_at", nullable = false)
    private Instant statusAt = Instant.now(); // Дата последнего изменения статуса
    
    // Вспомогательный метод для смены статуса
    public void updateStatus(BillStatus newStatus) {
        this.status = newStatus;
        this.statusAt = Instant.now();
    }
}
```

```sql
CREATE TABLE payment_bill (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id BIGINT NOT NULL,
  external_transaction_id VARCHAR(500),
  bank_id BIGINT,
  description VARCHAR(500),
  amount NUMERIC(19, 2) NOT NULL CHECK (amount > 0),
  status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  due_date TIMESTAMPTZ NOT NULL,
  status_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Бизнес-валидация
  CONSTRAINT positive_amount CHECK (amount > 0),
  CONSTRAINT valid_enum_status CHECK (status IN ('ACTIVE', 'PAID', 'VOIDED', 'EXPIRED'))
);

-- Индексы для ускорения поиска и работы Scheduler
CREATE INDEX idx_payment_bill_client ON payment_bill(client_id);
CREATE INDEX idx_payment_bill_lookup ON payment_bill(status, due_date);
```

#### Сущность `BillStatus` - это статус счета
```java
public enum BillStatus {
    ACTIVE,     // Счет активен (ожидает оплаты)
    PAID,       // Оплачен 
    VOIDED,     // Аннулирован (вручную) 
    EXPIRED     // Срок оплаты истек 
}
```

### BillRequestDTO
Этот объект приходит из `Delivery Service`. В нем только те данные, которые необходимы нам для расчета стоимости и создания записи.

| **Поле**       | **Тип**   | **Описание**                                                    |
| -------------- | --------- | --------------------------------------------------------------- |
| `clientId`     | `Long`    | ID пользователя (берется из контекста или передается сервисом). |
| `description`  | `String`  | Описание (например, "Доставка заказа #123").                    |
| `weight`       | `Double`  | Вес для расчета стоимости.                                      |
| `distance`     | `Double`  | Расстояние для расчета стоимости.                               |
| `durationDays` | `Integer` | На сколько дней выставляется счет (для вычисления `dueDate`).   |

```java
@Data
public class BillRequestDTO {
    @NotNull
    private Long clientId;
    
    @NotBlank
    @Size(max = 500)
    private String description;
    
    @Positive
    private Double weight;
    
    @Positive
    private Double distance;
    
    @Min(1)
    private Integer durationDays; 
}
```

### BillResponseDTO
Этот объект мы возвращаем покупателю. Здесь мы «склеиваем» данные из нашей БД и, возможно, данные из `User Service` (ФИО клиента).

| **Поле**      | **Тип** *(в коде)* | **Описание**                                     |
| ------------- | ------------------ | ------------------------------------------------ |
| `id`          | `UUID`             | Уникальный номер счета для оплаты.               |
| `amount`      | `BigDecimal`       | Итоговая сумма к оплате.                         |
| `description` | `String`           | За что платим.                                   |
| `clientName`  | `String`           | **ФИО**, которое мы подтянули из `User Service`. |
| `dueDate`     | `Instant`          | До какого времени нужно успеть оплатить.         |
| `status`      | `BillStatus`       | Текущий статус (обычно `ACTIVE`).                |
| `createdAt`   | `Instant`          | Время выставления.                               |
```Java
@Data
@Builder
public class BillResponseDTO {
    private UUID id;
    private BigDecimal amount;
    private String description;
    private String clientName; // Обогащенное поле
    private Instant dueDate;
    private BillStatus status;
    private Instant createdAt;
}
```

---
Банковская система залогинивается под ролью `BANK` и именем - для каждого банка своим, после чего отправляет на этот эндпоинт оплаченный счет, что свидетельствует об оплате данного счета. Все поля счета должны совпасть. После этого счет считается оплаченным и берется в работу, а в ответ бан получает статус 200, что свидетельствует о принятии платформой сообщения банка об оплате.. 

## BankConfirmation...
- `id` (UUID), `сумма` и `статус` — **обязательны**.    
- `ФИО` и `дата` — **опциональны** (могут быть `null`).    
- Статус оплаты — это `PAID` или `CANCELLED`.

### **BankConfirmationRequest** (Входящий от Банка)
Это то, что банк присылает на эндпоинт `/api/v1/payments/callback`.
```Java
@Data
public class BankConfirmationRequest {

    @NotNull(message = "ID счета обязателен")
    private UUID billId; // Уникальный id выставленного счета

    @NotNull(message = "Сумма обязательна")
    @Positive(message = "Сумма должна быть больше нуля")
    private BigDecimal amount; // Сумма по счету

    @NotNull(message = "Статус оплаты обязателен")
    private BankPaymentStatus status; // PAID или CANCELLED

    private String externalTransactionId; // Номер транзакции из системы банка (наш externalId)

    private String payerFullName; // ФИО плательщика (может быть null)

    private Instant paymentDateTime; // Дата/время оплаты (может быть null)
}
```

### **BankConfirmationResponse** (Ответ Банка)
Мы должны «продублировать принятые данные и дополнить их». Дополняем мы их теми данными, которые «добыли» сами (текущее время, если банк его не прислал, или ФИО из `User Service`).
```Java
@Data
@Builder
public class BankConfirmationResponse {

    private UUID billId;
    private BigDecimal amount;
    private BankPaymentStatus status;
    private String externalTransactionId;
    
    // Дополненные данные
    private String payerFullName; // Если от банка null, тут будет ФИО из User Service
    private Instant paymentDateTime; // ..от банк null, тут будет текущее время системы
    
    // Информационное сообщение (например, "Счет успешно обработан")
    private String message; 
}
```

## Вспомогательный **Enum** для Банка
Важно: этот Enum отражает то, что присылает **банк**, а не наше внутреннее состояние счета. Мы потом смапим это в наш `BillStatus`.
```Java
public enum BankPaymentStatus {
    PAID,      // Успешно оплачено
    CANCELLED  // Отмена оплаты
}
```

### Логика обработки:
1. **Если `payerFullName == null`**: Сервис идет в `User Service` по `clientId` 
	   (который уже есть в базе в нашей сущности `Bill`) и подтягивает имя.    
2. **Если `paymentDateTime == null`**: Сервис берет `Instant.now()`.    
3. **Запись в БД**: Данные обновляются в сущности `Bill` 
	   (меняется статус на `PAID`, обновляется `statusAt` и записывается `externalId`).
	   Если банк прислал `CANCELLED`, статус счета должен стать `VOIDED` или `CANCELED`.
	   В Entity `status_at` обновится в любом случае    
4. **Ответ**: Только после `repository.save()` отправляется `BankConfirmationResponse`.

> - Идемпотентность Callback: Банки могут присылать одно и то же уведомление несколько раз (retry). Сервис должен проверять статус счета перед обновлением. Если счет уже PAID, просто возвращай банку 200 OK без повторной отправки в Kafka.
> - Транзакционность: Обработка банковского колбэка должна быть в одной транзакции (@Transactional). Сохранение в БД и отправка сообщения в Kafka должны быть согласованы (используй паттерн Transactional Outbox, если требуется 100% гарантия, или отправляй в Kafka сразу после успешного коммита транзакции).
> - **Безопасность**: Эндпоинт `/callback` должен быть доступен только для роли `BANK`. В реальной системе также стоит проверять IP-адреса банка или использовать подпись (Signature) в заголовках.

---
## 🔄 Сценарий работы (Data Flow)
1. **Delivery Service** → `POST /payments/cost` (расчет суммы).    
2. **Delivery Service** → `POST /payments/bill` (создание счета).    
3. **Payment Service**:    
    - Считает `cost = baseRate + weight*10 + distance*5`.        
    - Создает `Bill` (status: `ACTIVE`, `dueDate` = `now + durationDays`).        
    - Запрашивает `User Service` для получения ФИО.        
    - Возвращает **BillResponseDTO**.    
4. **Клиент оплачивает** → **Банк** отправляет `POST /payments/callback`.    
5. **Payment Service**:    
    - Находит `Bill` по `id`.        
    - Если статус `ACTIVE` → переводит в `PAID`, обновляет `statusAt` и `externalId`.        
6. **Уведомление**:    
    - **Payment Service** отправляет сообщение в **Kafka** 
	      (topic: `payment.payment-status`, статус: `COMPLETED`).        
7. **Delivery Service**:    
    - Слушает Kafka и при статусе `COMPLETED` начинает процесс доставки.    
    - Слушает Kafka и при статусе `CANCELED` освобождает ресурсы.
8. `Delivery Service` присылает **BillRequestDTO**.
9. `Payment Service` берет `weight` и `distance`, считает `amount`.
10. Создает **Entity Bill**, сохраняет в БД.
11. Вызывает `User Service` по `clientId`, получает имя.
12. Формирует **BillResponseDTO** и отдает его.

---
