В нашей экосистеме `Cargo Service` является «хранителем» информации о товарах и их физических характеристиках, необходимых для расчета доставки.

### **Cargo Service**

### Задачи:
**Учет и управление характеристиками грузов**
- Хранит информацию о габаритах, весе и категории товаров;    
- Предоставляет данные для расчета стоимости в `Payment Service`;    
- Резервирует товар на складе при формировании счета (интеграция с `Delivery Service`);    
- Управляет инвентаризацией и остатками.    
### Стек:
- **Java 17 / Spring Boot 3.5**;    
- **Spring Data JPA**;    
- **PostgreSQL** (структурированные данные) + **MongoDB** (гибкие характеристики товаров);    
- **REST (Feign)** для внутренних запросов;    
- **Lombok / Swagger**;    

---
### Сценарии работы микросервиса:
1. **Обзор имеющихся товаров** их фотографий  характеристик. Пользователь может зайти по GET и получить DTO с описанием и всеми характеристиками товара (вес/ цвет/ размер/ цена), а так-же просмотреть имеющиеся фото товара.;
2. **Предоставление данных**: Когда покупатель хочет узнать цену доставки, `Delivery Service` запрашивает у `Cargo Service` вес и объем конкретного товара по его ID.    
3. **Бронирование**: В момент формирования счета (`bill`), `Cargo Service` ставит товар в статус «Забронировано», чтобы его не купил кто-то другой, пока ожидается оплата.    
4. **Подтверждение/Отмена**:    
    - Если приходит сигнал `PAID` (через Kafka, topic: `payment.payment-status`), товар переводится в статус «К отгрузке».
    - Если счет `EXPIRED` или `CANCELLED`, `Cargo Service` автоматически снимает бронь и возвращает товар в доступные остатки.
5. **Синхронизация**: При поступлении новых товаров от внешних источников, сервис обновляет базу данных.
6. **Наполнением/ списанием *(т.е. менеджментом)*** товаров на складе занимаются работники склада, менеджеры т.е., пользователи системы с ролью `MANAGER`.
7. **Контроль консистентности** (Cleanup): Если `Payment Service` по какой-то причине не прислал сигнал об отмене, а время бронирования (из метаданных счета) вышло, `Cargo Service` может иметь внутренний планировщик (`Scheduler`), который сверяет затянувшиеся `RESERVED` статусы и возвращает их в `AVAILABLE`.

---

### **Сущность Cargo (Товар/Груз)**
Каждая единица товара фиксируется отдельно, т.е. если есть 100 батареек, то будет 100 товаров "батарейка".

| **Имя переменной** | **Тип** (код/ БД)             | **Пояснение**                                                                     |
| ------------------ | ----------------------------- | --------------------------------------------------------------------------------- |
| id                 | `Long` / `Long`               | *Ключи:* Уникальный идентификатор товара.                                         |
| sku                | `String` / `VARCHAR(50)`      | *Ключи:* Уникальный строковый артикул (например, BAT-AA-001) для поиска.          |
| billId             | `UUID` / `UUID`               | *Ключи:* ID/ Линк счета, под который забронирован това                            |
| mongoDocId         | `String`                      | *Ключи:* Линк на документ в MongoDB (связь двух БД)/ там фото и ТТХ.              |
| name               | `String`/ `VARCHAR(100)`      | *Инфо:* Название/ название товара.                                                |
| category           | `Enum`                        | *Инфо:* Категория товара (электроника, мебель и т.д.)/ для фильтров               |
| weight             | `Double`                      | *Логистика:* Вес (используется для формулы в Payment Service).                    |
| dimensions         | `String`/ `VARCHAR(50)`       | *Логистика:* Габариты (длина, ширина, высота).                                    |
| price              | `BigDecimal`/ `NUMERIC(19,2)` | Стоимость товара.                                                                 |
| location           | `String`/ `VARCHAR(50)`       | *Склад:* Место на складе (Стеллаж/Полка). Это добавит веса роли менеджера склада. |
| status             | `Enum` / `VARCHAR(30)`        | *Склад:* Текущее состояние. CargoStatus: `AVAILABLE`, `RESERVED`, `SHIPPED`.      |
| createdAt          | `Instant`/ `TIMESTAMPTZ`      | *Аудит:* Дата поступления товара на склад                                         |
| statusAt           | `Instant`/ `TIMESTAMPTZ`      | *Аудит:* Дата последнего изменения статуса товара                                 |

### CargoStatus

| **Имя переменной** | **Пояснение**                                              |
| ------------------ | ---------------------------------------------------------- |
| AVAILABLE          | Доступен для бронирования                                  |
| RESERVED           | Забронирован в доставку                                    |
| SHIPPED            | Отгружен                                                   |
| DAMAGED            | Товар найден на складе поврежденным (списание менеджером). |
| LOST               | Утерян при инвентаризации                                  |
### Cargo
```java
@Entity
@Table(name = "cargo")
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
public class Cargo {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	
	@Column(nullable = false, unique = true, length = 50)
	private String sku; // Артикул (BAT-AA-001)
	
	@Column(name = "bill_id")
	private UUID billId; // ID счета из Payment Service
	
	@Column(name = "mongo_doc_id")
	private String mongoDocId; // Линк на MongoDB
	
	@Column(nullable = false, length = 100)
	private String name;
	
	@Enumerated(EnumType.STRING)
	@Column(nullable = false, length = 50)
	private CargoCategory category;
	
	@Column(nullable = false)
	private Double weight;
	
	@Column(length = 50)
	private String dimensions;
	
	@Column(nullable = false, precision = 19, scale = 2)
	private BigDecimal price;
	
	@Column(length = 50)
	private String location; // Складская ячейка (A-12-03)
	
	@Enumerated(EnumType.STRING)
	@Column(nullable = false, length = 30)
	private CargoStatus status = CargoStatus.AVAILABLE;
	
	@Column(name = "created_at", nullable = false, updatable = false)
	private Instant createdAt = Instant.now();
	
	@Column(name = "status_at", nullable = false)
	private Instant statusAt = Instant.now();
	
	// Helper для обновления статуса
	public void changeStatus(CargoStatus newStatus) {
		this.status = newStatus;
		this.statusAt = Instant.now();
	}
}
```

### V1__Create_Cargo_Table.sql
```sql
CREATE TABLE payment_bill ( -- Для справки, если нужно связать );
CREATE TABLE cargo (
id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
sku VARCHAR(50) NOT NULL UNIQUE,
bill_id UUID,
mongo_doc_id VARCHAR(255),
name VARCHAR(100) NOT NULL,
category VARCHAR(50) NOT NULL,
weight DOUBLE PRECISION NOT NULL,
dimensions VARCHAR(50),
price NUMERIC(19, 2) NOT NULL,
location VARCHAR(50),
status VARCHAR(30) NOT NULL DEFAULT 'AVAILABLE',
created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
status_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

-- Ограничения (Constraints)
CONSTRAINT positive_weight CHECK (weight > 0),
CONSTRAINT positive_price CHECK (price >= 0)
);

-- Индексы для быстрого поиска и фильтрации
CREATE INDEX idx_cargo_sku ON cargo(sku);
CREATE INDEX idx_cargo_bill_id ON cargo(bill_id);
CREATE INDEX idx_cargo_status_lookup ON cargo(status, category);
```

---
## DTO
### 1. **CargoResponseDTO** (*Для витрины товаров*)
Этот объект возвращается клиенту при просмотре каталога. Он объединяет данные из PostgreSQL и MongoDB.

|**Поле**|**Тип**|**Описание**|
|---|---|---|
|`id`|`Long`|ID товара.|
|`sku`|`String`|Артикул.|
|`name`|`String`|Название.|
|`price`|`BigDecimal`|Цена.|
|`category`|`CargoCategory`|Категория.|
|**`properties`**|`Map<String, Object>`|**Из MongoDB**: динамические ТТХ (цвет, материал и т.д.).|
|**`photos`**|`List<byte[]>`|**Из MongoDB**: список изображений в формате Binary.|
```Java
@Data
@Builder
public class CargoResponseDTO {
    private Long id;
    private String sku;
    private String name;
    private BigDecimal price;
    private CargoCategory category;
    private CargoStatus status;
    
    // Данные, подтянутые из MongoDB по mongoDocId
    private Map<String, Object> properties; 
    private List<byte[]> photos; 
}
```

### 2. **CargoManagerRequestDTO** (*Для создания/обновления*)
Используется пользователем с ролью `MANAGER` для заведения товара в систему.
```Java
@Data
public class CargoManagerRequestDTO {
    @NotBlank
    private String sku;
    
    @NotBlank
    private String name;
    
    @NotNull
    private BigDecimal price;
    
    @NotNull
    private Double weight;
    private String dimensions; // "10x20x30"
    private String location;   // "B-01-05"
    private CargoCategory category;

    // Поля для сохранения в MongoDB
    private Map<String, Object> extraProperties; 
}
```

### 3. **CargoLogisticsDTO** (*Внутренний запрос*)
Этот легковесный DTO передается в `Delivery Service` и далее в `Payment Service`. В нем нет лишней информации вроде фото или цвета, только то, что влияет на логистику.

|**Поле**|**Тип**|**Описание**|
|---|---|---|
|`id`|`Long`|ID товара.|
|`weight`|`Double`|Вес для расчета стоимости.|
|`dimensions`|`String`|Габариты для подбора транспорта.|

---
### Структура в **MongoDB** (Гибкие характеристики и фото)
- **Гибкие атрибуты**: Для батарейки это "емкость", для одежды — "состав ткани". В Mongo это будет просто JSON-объект attributes.
- **Хранение фото**:
    - Вариант А (Простой): Хранить фото как Binary (BSON) прямо в документе.
    - ~~Вариант Б (Правильный): Использовать GridFS (встроено в MongoDB) для хранения файлов крупного размера.~~
- **Метаданные фото**: Добавь поле contentType (image/jpeg) и originalName.
	  - Фото должны быть небольшие (до 16 МБ)
	  - **Гибкие атрибуты**: В документе MongoDB создай вложенный объект `properties` или `specs`.
		  _Пример:_ `{"brand":"Energizer", "capacity":"2500 mAh", "rechargeable":true }`.

### Структура документа в MongoDB (Техническая сущность)
Поскольку был выбрал **Вариант А** (хранение фото прямо в документе), документ в Mongo будет выглядеть так:
```java
@Document(collection = "cargo_details")
@Data
public class CargoDetailDoc {
    @Id
    private String id; // Это значение хранится в Postgres в поле mongoDocId
    
    private Map<String, Object> properties; // {"color": "red", "capacity": "5000mAh"}
    
    private List<CargoPhoto> photos;
}

@Data
public class CargoPhoto {
    private byte[] content;
    private String contentType; // "image/jpeg"
    private String originalName;
}
```

### Как это работает вместе (Flow):

1. **Создание**: Менеджер шлет `CargoManagerRequestDTO`. Сервис сначала сохраняет фото и свойства в **MongoDB**, получает `mongoDocId`, а затем сохраняет основные данные в **PostgreSQL**.
    
2. **Чтение**: При вызове `GET /cargo/{id}`, сервис берет запись из Postgres, видит `mongoDocId`, делает запрос в Mongo, «склеивает» данные в `CargoResponseDTO` и отдает пользователю.
    
3. **Логистика**: `Delivery Service` запрашивает товар по ID, сервис возвращает только `CargoLogisticsDTO` (быстро, без тяжелых фото).

**Для проекта**: Для реализации загрузки фото в контроллере лучше использовать `MultipartFile`. Если фото несколько, принимть `List<MultipartFile>`.

---

### 🔄 Место в потоке данных (согласно схеме):

1. **API Gateway** направляет запрос на просмотр товаров в `Cargo Service`.
    
2. **Delivery Service** выступает "оркестратором": он берет данные о весе из `Cargo Service` и отправляет их в `Payment Service` для генерации счета.
    
3. **Monitoring**: Сервис также подключен к **Promtail/Loki**, отправляя логи о движении товаров на складе.

---
## **Эндпоинты**
Эндпоинты разделены на **три** логические группы: для **покупателей** (*витрина*), для **менеджеров** (*управление складом*) и для **внутренних нужд** системы (*межсервисное взаимодействие*).

### 1. Группа: **Покупатель** (*Витрина товаров*)
Эти эндпоинты доступны пользователям с ролью `USER` через API Gateway.

| **Метод** | **Эндпоинт**           | **Описание**                           | **Бизнес-логика**                                                |
| --------- | ---------------------- | -------------------------------------- | ---------------------------------------------------------------- |
| **GET**   | `/api/v1/cargo`        | Получить список всех доступных товаров | Возвращает `CargoResponseDTO` (с данными из Pg и Mongo).         |
| **GET**   | `/api/v1/cargo/{id}`   | Детальная информация о товаре          | Склеивает данные из PostgreSQL и MongoDB (фото, характеристики). |
| **GET**   | `/api/v1/cargo/search` | Поиск и фильтрация товаров             | Поиск по названию, категории или SKU.                            |

### 2. Группа: **Менеджер** (*Управление складом*)
Доступно только пользователям с ролью `MANAGER`. Позволяет управлять жизненным циклом каждой единицы товара.

|**Метод**|**Эндпоинт**|**Описание**|**Бизнес-логика**|
|---|---|---|---|
|**POST**|`/api/v1/manager/cargo`|Регистрация нового груза|Принимает `CargoManagerRequestDTO` и файлы фото. Сохраняет метаданные в Pg, а фото/ТТХ в MongoDB.|
|**PUT**|`/api/v1/manager/cargo/{id}/status`|Ручное изменение статуса|Позволяет менеджеру списать товар (`DAMAGED`, `LOST`).|
|**DELETE**|`/api/v1/manager/cargo/{id}`|Удаление записи о грузе|Полное удаление из обеих баз данных (используется редко).|
|**GET**|`/api/v1/manager/reports/status`|Отчет по статусам товаров|Статистика: сколько товаров `AVAILABLE`, `RESERVED` и т.д..|

### 3. Группа: **Системные** (*Внутреннее взаимодействие*)

Эндпоинты для использования другими микросервисами через Feign Client. Защищены ролью `SERVICE`.

|**Метод**|**Эндпоинт**|**Описание**|**Кто использует**|
|---|---|---|---|
|**GET**|`/api/v1/internal/cargo/{id}/logistics`|Получить весогабаритные данные|**Delivery Service**: для расчета стоимости в Payment Service.|
|**POST**|`/api/v1/internal/cargo/reserve`|Забронировать товары под счет|**Delivery Service**: переводит список ID товаров в `RESERVED` и прописывает `billId`.|
|**POST**|`/api/v1/internal/cargo/release`|Освободить бронь|**Scheduler / Payment**: если счет просрочен, возвращает товары в `AVAILABLE`.|

---
### Важные уточнения по реализации:

1. **Загрузка фотографий**: В методе создания товара (POST для менеджера) используйте `MultipartFile` для передачи изображений. Фотографии сохраняются в MongoDB, а в PostgreSQL записывается только `mongoDocId`.
    
2. **Валидация**: Все входящие DTO должны проверяться аннотациями `@Valid` (проверка на положительный вес, наличие SKU и т.д.).
    
3. **Обработка ошибок**: Если товар с запрошенным ID не найден, сервис должен возвращать `404 Not Found` с понятным JSON-описанием ошибки.
    
4. **Отчетность**: Согласно заданию , добавлены эндпоинты для менеджера, позволяющие видеть состояние склада.

---
