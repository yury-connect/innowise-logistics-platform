### **Notification Service (Сервис уведомлений)**
Информационный шлюз системы, отвечающий за асинхронную доставку сообщений. Сервис не блокирует основные бизнес-процессы, вычитывая данные из брокера сообщений.

### **Задачи:**
- **Асинхронная обработка**: Вычитывание событий из Kafka и их распределение по каналам связи.
    
- **Идемпотентность**: Гарантия того, что пользователь не получит дубликат уведомления при повторном получении события из Kafka (реализовано через уникальные индексы в БД).
    
- **Мультиканальность**: Поддержка Telegram, Email, Push-уведомлений и SMS.
    
- **Шаблонизация**: Формирование текстов на основе кодов событий.
    
- **Гарантия доставки**: Механизм повторных попыток (Retry) при временной недоступности внешних API.

### **Стек:**
- **Java 17 / Spring Boot 3.5**;    
- **Spring Kafka**;    
- **PostgreSQL** (Единственное хранилище: логи, аудит, защита от дублей);    
- **Feign Client** (Запросы в User Service за контактами);    
- **Внешние SDK**: Spring Mail, Telegram Bots, Firebase Admin.    

---
### **Сценарии работы микросервиса:**

1. **Слушатель (Consumer)**: Мониторит `delivery.notify-event`. 
	   Определяет тип события (статус, инцидент, счет).
    
2. **Проверка на дубликаты**: Пытается создать запись в `notification_logs`. 
	   Если `kafka_event_id` уже существует — обработка прекращается.
    
3. **Обогащение (Enrichment)**: Запрашивает через Feign в **User Service** 
	   контактные данные (Email, TG ID, Phone) по `userId`.
    
4. **Генерация**: На основе `eventStatus` выбирается шаблон, 
	   куда подставляются данные из `messagePayload`.
    
5. **Отправка**: Сообщение уходит параллельно во все доступные каналы.
    
6. **Логирование**: Статус отправки обновляется в БД 
	   и транслируется в **Reporting Service**.

---
### **Структура события (DTO из Kafka)**

|**Поле**|**Тип**|**Пояснение**|
|---|---|---|
|**eventId**|`String`|Уникальный ID события (для идемпотентности).|
|**userId**|`Long`|Кому отправить уведомление.|
|**trackingNumber**|`String`|Номер доставки ("TRK-123").|
|**eventStatus**|`Enum`|Статус (ORDER_CREATED, PAID, INCIDENT, etc.).|
|**messagePayload**|`Map`|Переменные для шаблона (цена, причина задержки).|
|**priority**|`Enum`|LOW (можно ждать) / HIGH (мгновенно).|

---
### **Сущность NotificationLog (JPA Entity)**
```java
@Entity
@Table(name = "notification_logs", indexes = {
    @Index(name = "idx_event_id", columnList = "kafka_event_id", unique = true)
})
@Getter @Setter
@NoArgsConstructor
public class NotificationLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "kafka_event_id", nullable = false)
    private String kafkaEventId; // Уникальный ID для защиты от дублей

    @Column(nullable = false)
    private Long userId; 

    @Column(name = "delivery_id")
    private Long deliveryId; 

    @Enumerated(EnumType.STRING)
    private NotificationChannel channel; // EMAIL, TELEGRAM, PUSH, SMS

    @Column(columnDefinition = "TEXT")
    private String content; // Итоговый текст

    @Enumerated(EnumType.STRING)
    private NotificationStatus status; // PENDING, SENT, FAILED

    private Instant createdAt = Instant.now(); 
    private Instant sentAt; // Заполняется по факту успеха
}
```

---
### **Логика шаблонов**
- **ORDER_CREATED**: "Заказ {id} успешно создан. Ожидайте выставления счета."    
- **ORDER_PAID**: "Оплата получена. Заказ {id} передан на склад."    
- **INCIDENT_CRITICAL**: "Критическое ЧП: доставка {id} отменена. 
	  Средства будут возвращены."    
- **DELIVERED**: "Заказ {id} доставлен. Спасибо, что вы с нами!"    

---
### **🔄 Место в потоке данных:**
1. **Инициатор** (Delivery/Payment) -> **Kafka**.    
2. **Notification Service** -> вычитывает.    
3. **User Service** -> отдает контакты.    
4. **Внешние API** -> доставляют месседж.    
5. **Promtail/Loki** -> фиксируют ошибки (например, 403 от Telegram).

---
---
---
#### **Вторая очередь проекта:**
- **Тихие часы**: Настройка "Не беспокоить" в профиле пользователя (кроме критических алертов).    
- **Batching**: Склеивание мелких уведомлений в один ежедневный/ежечасный дайджест.

---


