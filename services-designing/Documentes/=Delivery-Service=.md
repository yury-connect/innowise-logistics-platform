# **Delivery Service**  (Центральное ядро)
Микровервис является "Центральным ядром" процесса доставки, реализует паттерн "Сага"

### **Задачи:**
- **Управление Сагой**: Инициация процесса 
	  "Заказ -> Расчет -> Бронирование -> Оплата -> Назначение".    
- **Валидация бизнес-цепочки**: Проверка возможности доставки 
	  (вес/ объем груза из `Cargo` против лимитов ТС из `Transport`).    
- **State Machine (Машина состояний)**: Жесткий контроль переходов статусов доставки.    
- **Отказоустойчивость**: Обработка сбоев:
	  (если оплата не прошла — разбронировать ресурсы/ 
	  если Cagro уведомил о просрочке брони - разбронирование ресурсов и анулирование счета, уведомление покупателя).    
- **Информационный хаб**: Сборка итогового объекта доставки для UI 
	  из данных 3-х разных микросервисов/ БД.    
- **Инциденты**: Обработка инцидентов по пути следования груза при доставке.
### **Стек:**
- **Java 17 / Spring Boot 3.5**.    
- **PostgreSQL**: Таблица `deliveries` (основной реестр).    
- **Redis:** Список активных доставок (на текущий/ настоящий момент)
- **Kafka**: Слушает `payment.payment-status` и отправляет события в `delivery.notify-event`.    
- **Feign Clients**: `CargoClient`, `TransportClient`, `PaymentClient`.    


---
### **Сущность Delivery (Опорный объект)**

```Java
@Entity
@Getter`, `@Setter`, `@NoArgsConstructor`
@Table(name = "deliveries")
public class Delivery {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private final Long id;

    @Column(unique = true, nullable = false)
    private String trackingNumber; // Номер для отслеживания "TRK-12345678"

    // Ссылки на внешние системы
    private Long cargoId; // id самого товара в доставку
    private Long transportId; // id ТС, на которм поедет
    private Long driverId; // id водителя на эту доставку
    private UUID billId; // id счета
    private Long clientId; // id покупателя/ пользователя

    // География
    private Address departureAddress; // адрес отправления
    private Address destinationAddress; // адрес назначения
    private Double distance; // км

    @Enumerated(EnumType.STRING)
    private DeliveryStatus status; // актуальный статус
       
    private BigDecimal totalCost; // Фиксируем сумму в момент расчета

    private Instant createdAt; // дата создания
    private Instant updatedAt; // дата последней модификации (любое изменение)
	private Instant statusAt = Instant.now(); // когда последнее изменение статуса
}
```

```java
@Entity
@Getter`, `@Setter`, `@NoArgsConstructor`
@Table(name = "addresses")
public class Address {
	private String country; // страна 
	private Integer zipCode; // почтовый индекс
	private String city; // город
	private String street; // улица
	private Integer house; // номер дома
	private String block; // корпус
}
```


---
### **🔄 Жизненный цикл** и **Машина состояний** (*State Machine*)

```text
`CREATED` → `WAITING_PAYMENT` → `PAID` → `STORED` → `IN_TRANSIT` → `DELIVERED`
   ↓                ↓             ↓         ↓          ↓
CANCELLED  ←  CANCELLED  ←  CANCELLED  ← CANCELLED ← CANCELLED
```

###### **Строгое правило**: статус может измениться **только вперед** по цепочке, либо в `CANCELLED` на любом этапе.
1. **CREATED**: Заявка создана. Запрошен счет в `Payment`.    
2. **WAITING_PAYMENT**: Счет выставлен, ресурсы (`Cargo`, `Driver` и `Vehicle`) забронированы. 
3. **PAID**: Подтверждение оплаты. (Переход по сигналу Kafka от `Payment`).    
4. **STORED**: Водитель подтвердил прием груза на складе.    
5. **IN_TRANSIT**: ТС в пути.    
6. **DELIVERED**: Груз передан получателю. Финиш.    
7. **CANCELLED**: Отмена на любом этапе до `DELIVERED`.    

## 📊 **Статусы доставки**  (Enum)
```java
public enum DeliveryStatus {
    CREATED,         // Заявка создана
    WAITING_PAYMENT, // Счет выставлен (не оплачен), ресурсы забронированы    
    PAID,            // Подтверждение оплаты (Счет оплачен)
    STORED,          // Груз принят ✅
    IN_TRANSIT,      // Груз в пути 🚚
    DELIVERED,       // Груз доставлен 📦
    CANCELLED        // Отменена ❌
}
```


## ДОСТАВКА
Перевозчик заходит в систему вручную под ролью `DRIVER` и меняет статус доставки товара. Статус может изменяться ТОЛЬКо по бизнес-правилам, описанным ниже по карте. Любое изменение статуса логируется в **Reporting Service**, а так-же отправляется заказчику уведомлением через **Notification Service**

#### Endpoint для захода перевозчика:

| **Метод** | **Эндпоинт**                     | **Описание**                                                              |
| --------- | -------------------------------- | ------------------------------------------------------------------------- |
| **PATCH** | `/api/v1/deliveries/{id}/status` | Водитель меняет состояние доставки. *(только для ролей `ADMIN`/`DRIVER`)* |
В логику `PATCH /status` необходимо добавить обязательную проверку через Feign в `Transport Service`: активен ли еще профиль этого водителя (`active=true`), прежде чем разрешить ему менять статус доставки.

---
## Инциденты:

Теперь опишем ЧП, произосшедшее в пути при доставке товара.
 При доставке любая непредвиденная остановка водителем/ задержка водителем/ авария ТС  обязана быть сопровождена спец. событием на спец. эндпоинт с описанием события.
 Далее система принимает решение в зависимости от типа события.

| **Метод** | **Эндпоинт**                             | **Описание**                                 |
| --------- | ---------------------------------------- | -------------------------------------------- |
| **POST**  | `/api/v1/deliveries/{id}/emergency-stop` | Экстренная остановка <br>доставки водителем. |

### DTO
```java
public class EmergencyIncidentRequest {
    private final String trackingNumber;  // Уникальный публичный код доставки (TRK-...)
    private final IncidentType incidentType; // Тип происшествия из фикс. списка
    private final String description;        // Подробности ЧП, введенные водителем
    private final String location;           // Географич. координаты или адрес места
    private final Instant timestamp;         // Момент времени, когда произошло событие
    private final Boolean isVehicleOperable; // Флаг: может ли ТС продолжать движение
}
```

### Типы ЧП `incidentType` (enum)
```java
public enum IncidentType {    
// --- ИНФОРМАЦИОННЫЕ (задержка, но доставка продолжается) ---
    TYRE_PUNCTURE(false),   // Прокол шины: требуется время на замену
    MINOR_BREAKDOWN(false), // Мелкая поломка: ремонт силами водителя
    OUT_OF_FUEL(false),     // Пустой бак: ожидание дозаправки
    MINOR_ISSUE(false),     // Прочие ситуации, не мешающие завершению заказа

    // --- КРИТИЧЕСКИЕ (требуют отмены доставки и аннулирования ресурсов) ---
    CARGO_LOST(true),       // Утеря груза: доставка невозможна физически
    CARGO_DAMAGED(true),    // Повреждение товара: клиент не примет заказ
    VEHICLE_ACCIDENT(true), // ДТП: требуется эвакуация и отмена рейса
    CRITICAL_FAILURE(true); // Форс-мажор, делающий доставку невозможной

    private final boolean requiresCancellation; // Внутренний флаг критичности

    IncidentType(boolean requiresCancellation) {
        this.requiresCancellation = requiresCancellation;
    }

    public boolean leadsToCancellation() {
        return requiresCancellation; // ведет ли данный инцидент к отмене заказа
    }
}
```

### Пример JSON: Критический инцидент (ДТП)
```JSON
{
  "trackingNumber": "TRK-98765432",
  "incidentType": "VEHICLE_ACCIDENT",
  "description": "Лобовое столкновение на трассе М-10, ТС сильно повреждено, груз уничтожен.",
  "location": "56.3269, 44.0059 (М-10, 402 км)",
  "timestamp": "2026-05-10T16:15:00Z",
  "isVehicleOperable": false
}
```

### JPA Entity
```java
@Entity
@Table(name = "delivery_incidents")
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class DeliveryIncident {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;                      // Первичный ключ записи в БД

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "delivery_id", nullable = false)
    private Delivery delivery;            // Ссылка на родительскую доставку (FK)

    @Enumerated(EnumType.STRING)
    @Column(name = "incident_type", nullable = false, length = 30)
    private IncidentType incidentType;    // Сохранение типа инцидента строкой

    @Column(columnDefinition = "TEXT")
    private String description;           // Неограниченный по объему текст описания

    private String location;              // Данные о местоположении

    @Column(name = "is_vehicle_operable")
    private Boolean isVehicleOperable;    // Состояние ТС на момент инцидента

    @Column(name = "is_critical", nullable = false)
    private boolean critical;             // Дениnormalization флага для быстрых отчетов

    @Column(name = "occurred_at", nullable = false)
    private Instant occurredAt;            // Время инцидента "на месте" (из DTO)

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;            // Время сохранения записи в нашей системе

    @PrePersist
    public void prePersist() {
        this.createdAt = Instant.now();  // Авто-заполнение времени создания
        if (this.incidentType != null) {
            // Авто-определение критичности на базе логики Enum
            this.critical = this.incidentType.leadsToCancellation();
        }
    }
}
```

### SQL Миграция (PostgreSQL)
```SQL
CREATE TABLE delivery_incidents (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY, -- Автоинкрементный ID
    delivery_id BIGINT NOT NULL,           -- Связь с основной таблицей доставок
    incident_type VARCHAR(30) NOT NULL,    -- Строковый код типа инцидента
    description TEXT,                      -- Текстовое поле для длинных заметок
    location VARCHAR(255),                 -- Строка локации
    is_vehicle_operable BOOLEAN DEFAULT TRUE, -- Флаг возможности движения
    is_critical BOOLEAN NOT NULL,          -- Флаг для аналитики и фильтрации
    occurred_at TIMESTAMPTZ NOT NULL,      -- Момент ЧП по часам водителя
    created_at TIMESTAMPTZ NOT NULL,       -- Момент регистрации сервером
    
    -- Гарантия целостности данных: нельзя создать инцидент для несуществующей доставки
    CONSTRAINT fk_delivery FOREIGN KEY (delivery_id) REFERENCES deliveries(id)
);

-- Индекс для быстрой выборки всех происшествий по конкретной доставке
CREATE INDEX idx_incident_delivery ON delivery_incidents(delivery_id);
```

### **в Service-слое:**
При получении критического инцидента (где `critical == true`), `Delivery Service` должен:
1. Установить `Delivery.status = CANCELLED`.
2. Отправить асинхронное сообщение в **Kafka** для `Payment Service` (аннулирование счета).
3. Отправить сообщение для `Cargo Service` (возврат товара в `AVAILABLE`).
4. Если `isVehicleOperable == false`, отправить сообщение в `Transport Service` для перевода ТС в статус `MAINTENANCE`.

---
### **Эндпоинты** (API Strategy)

#### **1. Клиентская зона (Роль: `USER`)**

| **Метод** | **Эндпоинт**                   | **Описание**         | **Логика**                                                                                                                   |
| --------- | ------------------------------ | -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **POST**  | `/api/v1/deliveries`           | **Создать доставку** | Идет в `Cargo` за весом -> ищет ТС в `Transport` -> запрашивает счет в `Payment` -> сохраняет со статусом `WAITING_PAYMENT`. |
| **GET**   | `/api/v1/deliveries/{trackNo}` | Трекинг              | Собирает из Pg текущий статус и местоположение.                                                                              |

#### **2. Операционная зона (Роль: DRIVER)**

| **Метод** | **Эндпоинт**                     | **Описание**  | **Логика**                                                                                              |
| --------- | -------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------- |
| **PATCH** | `/api/v1/deliveries/{id}/status` | Смена стату Водитель ставит: <br>- `STORED` (забрал) <br>- `IN_TRANSIT` (Груз в пути) <br>- `DELIVERED` (отдал).  `  -  |

#### **3. Управленческая зона (Роль: MANAGER / ADMIN)**

|**Метод**|**Эндпоинт**|**Описание**|**Логика**|
|---|---|---|---|
|**GET**|`/api/v1/deliveries`|Список всех доставок|Пагинация и фильтрация по статусам/датам.|
|**POST**|`/api/v1/deliveries/{id}/cancel`|Принудительная отмена|Шлет команды в Kafka на разбронирование товара и ТС.|

---
### **Детальный Flow создания доставки (Алгоритм дирижера):**

1. **Прием**: Получаем `cargoId`, `clientId`, `route`.
    
2. **Cargo Check (Feign)**:    
    - `GET /api/v1/internal/cargo/{id}/logistics`        
    - Проверяем, доступен ли товар (`AVAILABLE`). Получаем его вес.
        
3. **Transport Search (Feign)**:    
    - `GET /api/v1/internal/vehicles?status=AVAILABLE&minPayload=X`        
    - Находим подходящую машину.
    
4. **Cost Calculation (Feign)**:    
    - `POST /api/v1/internal/payments/calculate`        
    - `Payment Service` считает: `Base(100) + Weight*10 + Dist*5`.
    
5. **Payment Billing**:    
    - `POST /api/v1/internal/payments/bill`        
    - Создаем счет. Получаем `billId`.
    
6. **Resource Reservation**:    
    - `POST /api/v1/internal/cargo/reserve` (Статус товара -> `RESERVED`).        
    - `PATCH /api/v1/internal/vehicles/{id}/status` (Статус ТС -> `ON_ASSIGNMENT`).
    
7. **Finalize**: Сохраняем `Delivery` в БД со статусом `WAITING_PAYMENT`. Шлем приветственное сообщение в `Notification Service`.

---
### **Обработка ЧП** (Failure Handling)

- **Поломка ТС**: Если `Transport Service` шлет в Kafka сигнал "Vehicle Maintenance" для ТС, которое `ON_ASSIGNMENT`, `Delivery Service` автоматически пытается найти замену через `Transport Service`. Если замены нет — шлет алерт менеджеру.
    
- **Неоплата**: Если от `Payment Service` пришел сигнал `EXPIRED`, `Delivery Service` выполняет **компенсирующие транзакции**: освобождает товар и возвращает машину в пул `AVAILABLE`.


---
---
---
