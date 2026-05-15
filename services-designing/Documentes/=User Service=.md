**User Service** — это фундамент безопасности и персонализации всей системы. Он не только хранит данные профилей, но и выступает связующим звеном между аутентификацией (Keycloak) и бизнес-логикой других сервисов (например, когда Notification Service нужно узнать Email по ID).

Ниже приведено полное и структурированное описание сервиса с учетом твоих набросков и архитектурных правок.

---
## **User Service** (Профили и Документы)

### **Задачи:**
- **Управление профилями**: Хранение расширенных данных (ФИО, контакты, настройки).
- **Хранилище файлов (DMS)**: Загрузка и хранение аватаров и сканов 
	  личных документов (паспорта, права) в **MongoDB**.
- **Управление ролями**: Назначение единственной роли 
	  каждому пользователю (например, `CLIENT`, `DRIVER`, `MANAGER`, `ADMIN`)..    
- **Интеграция с IAM**: Синхронизация данных с **Keycloak** (UUID пользователя) для обеспечения единой точки входа (SSO). 

### **Стек:**
- **Java 17 / Spring Boot 3.5**.    
- **PostgreSQL**: Реляционные данные (профили, роли, адреса).    
- **MongoDB (GridFS)**: Хранение файлов. _Почему GridFS?_ Это позволяет хранить файлы любого размера внутри БД, обеспечивая легкий бэкап всей системы документов.    
- **REST (Feign)**: Синхронные запросы к Keycloak и другим сервисам.
- **Lombok** / **Swagger**
- **Keycloak**: Внешний провайдер для Auth (сервис хранит только ссылку на `keycloak_id`).

---
### **Сущность UserProfile (PostgreSQL)**

| **Имя переменной** | **Тип (код/ БД)**    | **Пояснение**                                              |
| ------------------ | -------------------- | ---------------------------------------------------------- |
| **id**             | `Long` / `BIGINT`    | Внутренний идентификатор в нашей системе.                  |
| **keycloakId**     | `UUID` / `UUID`      | Связь с ID пользователя в Keycloak.                        |
| **email**          | `String` / `VARCHAR` | Электронная почта (синхронизирована с Keycloak).           |
| **firstName**      | `String` / `VARCHAR` | Имя пользователя.                                          |
| **lastName**       | `String` / `VARCHAR` | Фамилия пользователя.                                      |
| **phone**          | `String` / `VARCHAR` | Номер телефона (для уведомлений).                          |
| **telegramId**     | `String` / `VARCHAR` | ID для Telegram-бота.                                      |
| **role**           | `Enum` / `VARCHAR`   | Единственная роль: `CLIENT`, `DRIVER`, `MANAGER`, `ADMIN`. |
| **avatarId**       | `String` / `VARCHAR` | ID документа в MongoDB (ссылка на аватар).                 |
| **createdAt**      | `Instant`            | Дата регистрации.                                          |
| **updatedAt**      | `Instant`            | Дата последнего изменения данных.                          |

---
### **User Service. Таблица эндпоинтов (REST API)**

Все эндпоинты защищены Gateway. 

| **Метод** | **Эндпоинт**                | **Описание**                                                 | **Доступ (Role)**      |
| --------- | --------------------------- | ------------------------------------------------------------ | ---------------------- |
| **POST**  | `/api/v1/users/register`    | Регистрация: создает запись в Keycloak и профиль в Postgres. | `public`               |
| **POST**  | `/api/v1/users/login`       | Аутентификация: возвращает JWT через Keycloak.               | `public`               |
| **GET**   | `/api/v1/users/{id}`        | Получить детальный профиль (включая линк на аватар).         | `USER (свой)`, `ADMIN` |
| **PUT**   | `/api/v1/users/{id}`        | Обновить данные (телефон, Telegram ID, имя).                 | `USER (свой)`, `ADMIN` |
| **PATCH** | `/api/v1/users/{id}/role`   | Смена роли пользователя (только администратором).            | `ADMIN`                |
| **POST**  | `/api/v1/users/{id}/avatar` | Загрузка/обновление фото профиля в MongoDB.                  | `USER (свой)`          |
| **POST**  | `/api/v1/users/{id}/docs`   | Загрузка документов (например, ВУ для водителя).             | `USER (свой)`          |

**Работа с документами (MongoDB):**
Путь: `/api/v1/users/...`

|**Метод**|**Эндпоинт**|**Описание**|**Доступ**|
|---|---|---|---|
|**POST**|`/{id}/avatar`|Загрузить аватар (MultipartFile)|`USER`|
|**GET**|`/{id}/avatar`|Стриминг аватара из MongoDB|`public`|
|**POST**|`/{id}/documents`|Загрузка документов (права, паспорт)|`USER`, `MANAGER`|
|**GET**|`/{id}/documents`|Список метаданных документов|`USER (свой)`, `MANAGER`|

---
### **Взаимодействие с другими сервисами:**
1. **Transport Service**: При регистрации водителя проверяет у `User Service`, имеет ли данный `userId` роль `DRIVER`.    
2. **Notification Service**: Запрашивает у `User Service` поля `email` или `telegramId` для отправки сообщений о статусе доставки.    
3. **Delivery Service**: Использует данные пользователя для заполнения информации об отправителе и получателе в накладной.

### **Структура в MongoDB (GridFS):**
Для документов мы используем коллекцию метаданных, где храним:
- `userId`: владелец документа.    
- `docType`: тип (AVATAR, LICENSE, PASSPORT).    
- `fileId`: ссылка на файл в GridFS.    
- `expiryDate`: дата истечения (актуально для водительских прав).

---
### SQL Миграция (PostgreSQL / Flyway)
Файл: `V1__Create_User_Profiles_Table.sql`
```sql
CREATE TABLE user_profiles (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    keycloak_id UUID NOT NULL UNIQUE,          -- Связь с Keycloak
    email VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    telegram_id VARCHAR(50),
    role VARCHAR(20) NOT NULL,                -- Одиночная роль: CLIENT, DRIVER, etc.
    avatar_id VARCHAR(255),                   -- Ссылка на GridFS в MongoDB
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Индексы для ускорения работы Notification и Transport сервисов
CREATE INDEX idx_users_keycloak ON user_profiles(keycloak_id);
CREATE INDEX idx_users_email ON user_profiles(email);
CREATE INDEX idx_users_role ON user_profiles(role);
```

### JPA Entity (Java)
```java
@Entity
@Table(name = "user_profiles")
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "keycloak_id", nullable = false, unique = true)
    private UUID keycloakId; // UUID из токена Keycloak

    @Column(nullable = false)
    private String email;

    @Column(name = "first_name")
    private String firstName;

    @Column(name = "last_name")
    private String lastName;

    private String phone;

    @Column(name = "telegram_id")
    private String telegramId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserRole role; // CLIENT, DRIVER, MANAGER, ADMIN

    @Column(name = "avatar_id")
    private String avatarId; // ID документа в MongoDB GridFS

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = Instant.now();
        updatedAt = Instant.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = Instant.now();
    }
}
```

### 3. Структура метаданных документов (MongoDB)

Хотя файлы лежат в GridFS, нам нужна коллекция `user_documents` для управления метаданными (особенно для водительских прав).
```Java
@Document(collection = "user_documents")
@Getter @Setter
@Builder
public class UserDocument {
    @Id
    private String id;
    
    private Long userId;          // Ссылка на ID из Postgres
    
    @Indexed
    private String docType;       // LICENSE, PASSPORT, MED_CERT
    
    private String fileId;        // Ссылка на сам файл в GridFS
    
    private LocalDate expiryDate; // Для валидации в Transport Service
    
    private Instant uploadedAt;
}
```

---
---
---
### Записки для реализации в коде:

- **Синхронизация ролей**: При вызове `PATCH /role` сервис должен не только обновить Postgres, но и отправить запрос в Keycloak API, чтобы обновить Role Mapping в токене пользователя.
    
- **Валидация водителей**: При регистрации роли `DRIVER` в `User Service`, сервис должен инициировать событие в Kafka, чтобы `Transport Service` создал пустую карточку водителя, ожидающую проверки документов.
    
- **Транзакции**: Метод регистрации должен быть транзакционным: если профиль в Postgres не создался, регистрация в Keycloak должна откатываться (или наоборот).

---


	

