# Техническое задание

## Система RAG-as-a-Service (Retrieval-Augmented Generation Platform)

**Дата:** 08.04.2026

---
# 1. ВВЕДЕНИЕ

## 1.1 Цель документа

Настоящий документ определяет требования к разработке платформы RAG-as-a-Service — масштабируемой backend-системы для работы с пользовательскими документами с использованием подхода Retrieval-Augmented Generation (RAG) .

## 1.2 Область применения

Система предназначена для:

* хранения и обработки пользовательских документов
* семантического поиска
* генерации ответов с использованием LLM
* обеспечения изоляции данных между организациями

## 1.3 Определения и сокращения

* RAG — Retrieval-Augmented Generation
* LLM — Large Language Model
* API — Application Programming Interface
* JWT — JSON Web Token
* S3 — объектное хранилище
* gRPC — протокол удалённых вызовов
* Multi-tenancy — мультиарендность

## 1.4 Целевая аудитория

* Backend-разработчики
* ML-инженеры
* Системные архитекторы
* DevOps-инженеры
* Заказчик

---

# 2. ОБЩЕЕ ОПИСАНИЕ СИСТЕМЫ

## 2.1 Контекст системы

Платформа обеспечивает обработку документов пользователей с последующим поиском и генерацией ответов через LLM. Система работает в многопользовательском режиме с полной изоляцией данных между организациями .

## 2.2 Архитектурный подход

* Архитектура: микросервисная
* Развертывание: cloud/on-premise (гибридно возможно)
* Коммуникация:

  * REST (внешние API)
  * gRPC (внутренние сервисы)
* Обработка: асинхронная (pipeline ingestion)

## 2.3 Основные функциональные блоки

* Аутентификация и управление пользователями
* Загрузка и обработка документов
* Генерация embeddings
* Семантический поиск
* Генерация ответов (LLM)
* UI для взаимодействия

---

# 3. ФУНКЦИОНАЛЬНЫЕ ТРЕБОВАНИЯ

## 3.1 Модуль аутентификации (Auth Service)

### 1.1 Аутентификация пользователей

**Приоритет:** Критический

Требования:

* Генерация JWT токенов
* Включение в payload:

  * user_id
  * organization_id
* Подпись токенов (RSA / EdDSA)

---

## 3.2 API Gateway

### 2.1 Управление запросами

**Приоритет:** Критический

Требования:

* Валидация JWT локально
* Роутинг запросов
* Генерация pre-signed URL для S3
* Проброс organization_id во все сервисы

---

## 3.3 Ingestion Service

### 3.1 Обработка документов

**Приоритет:** Высокий

Требования:

* Получение событий загрузки из S3
* Парсинг документов (PDF, DOCX, TXT)
* Разбиение на chunks (с overlap)
* Генерация embeddings
* Сохранение в векторную БД
* Асинхронная обработка

---

## 3.4 Embedding Service

### 4.1 Генерация embeddings

**Приоритет:** Критический

Требования:

* Методы:

  * Embed(text)
  * EmbedBatch(texts[])
* Stateless сервис
* Поддержка разных провайдеров

---

## 3.5 Retrieval Service

`Retrieval Service` - внутренний gRPC-сервис для семантического поиска по чанкам документов в Qdrant. Frontend не должен вызывать его напрямую: внешний трафик идет через API Gateway, а gateway прокидывает `organization_id` во внутренние gRPC-вызовы.

## Что умеет

* Принимает gRPC-запрос `retrieval.v1.RetrievalService/Search`.
* Получает `organization_id` из metadata `x-organization-id`.
* Не выполняет поиск, если `organization_id` отсутствует.
* Генерирует embedding запроса через общий пакет `pkg/common/embeddings` и Yandex AI.
* Выполняет поиск похожих векторов в Qdrant.
* Обязательно добавляет tenant-фильтр по `organization_id` в каждый Qdrant-запрос.
* Возвращает найденные чанки: `id`, `document_id`, `chunk_id`, `text`, `score`, `metadata`.
* Поддерживает `limit` и `score_threshold`.

## gRPC API

Proto-файл: `backend/pkg/common/proto/retrieval/v1/retrieval.proto`

Метод:

```proto
rpc Search(SearchRequest) returns (SearchResponse);
```

Пример запроса:

```json
{
  "query": "о чем этот документ?",
  "limit": 5,
  "score_threshold": 0.3
}
```

Пример ответа:

```json
{
  "results": [
    {
      "id": "point-id",
      "document_id": "document-id",
      "chunk_id": "chunk-id",
      "text": "релевантный текст чанка",
      "score": 0.87,
      "metadata": {
        "page": "1"
      }
    }
  ]
}
```

## Конфигурация

Пример env-файла: `backend/services/retrieval/.env.example`.

Основные переменные:

```env
GRPC_HOST=0.0.0.0
GRPC_PORT=50051

YANDEX_API_KEY=replace-me
YANDEX_FOLDER_ID=replace-me
YANDEX_EMBEDDING_MODEL=
YANDEX_EMBEDDING_BASE_URL=https://ai.api.cloud.yandex.net/v1

QDRANT_URL=http://qdrant:6333
QDRANT_COLLECTION=document_chunks
QDRANT_VECTOR_NAME=

RETRIEVAL_DEFAULT_LIMIT=5
RETRIEVAL_MAX_LIMIT=20
```

Если `YANDEX_EMBEDDING_MODEL` не задан, сервис использует модель:

```text
emb://<YANDEX_FOLDER_ID>/text-search-query/latest
```

## Локальный запуск

Из корня репозитория:

```bash
docker compose -f backend/docker-compose.yml up --build qdrant retrieval
```

Retrieval публикуется наружу на `localhost:50053`, внутри docker-сети слушает `50051`.

Пример ручного запроса через `grpcurl`:

```bash
grpcurl -plaintext \
  -H "x-organization-id: <organization-id>" \
  -d '{"query":"текст запроса","limit":5}' \
  localhost:50053 retrieval.v1.RetrievalService/Search
```

## Ожидаемый payload в Qdrant

Чтобы поиск работал корректно и безопасно, при индексации каждый point в Qdrant должен иметь payload минимум с такими полями:

```json
{
  "organization_id": "tenant-id",
  "document_id": "document-id",
  "chunk_id": "chunk-id",
  "text": "chunk text"
}
```

`organization_id` обязателен: Retrieval фильтрует по нему каждый поиск и не должен возвращать чанки других организаций.


---

## 3.6 LLM Router Service

### 6.1 Генерация ответов

**Приоритет:** Критический

Требования:

* Выбор модели LLM
* Формирование prompt
* Генерация ответа
* Возврат источников (chunks)

---

## 3.7 Document Service

### 7.1 Общая роль сервиса

**Приоритет:** Критический

Document Service является control-plane сервисом для управления документами и их метаданными.

**Особенности:**

* Не принимает прямые HTTP-запросы от клиентов
* Доступен только через gRPC
* Используется API Gateway как единственная точка входа
* Является tenant-aware сервисом (работает в рамках `organization_id`)

---

### 7.2 Управление документами (Control Plane)

**Приоритет:** Критический

**Требования:**

* Создание записи документа в БД (`rag_app.documents`)
* Хранение метаданных:
  * `document_id`
  * `organization_id`
  * `user_id`
  * `filename`
  * `content_type`
  * `size`
  * `status`
  * `created_at`, `updated_at`
* Все операции выполняются строго в рамках `organization_id`
* Валидация входных данных

---

### 7.3 Генерация S3 Pre-signed URL

**Приоритет:** Критический

**Требования:**

* Генерация pre-signed URL для загрузки файлов
* URL возвращается через API Gateway клиенту
* Ограниченный TTL
* Поддержка метода `PUT`
* Формирование tenant-safe ключа:

```
{organization_id}/documents/{document_id}/source.{ext}
```


* Проверка соответствия `organization_id` из gRPC-контекста (проброшенного из JWT)

---

### 7.4 Управление статусами документа

**Приоритет:** Критический

**Статусы:**

* `pending_upload`
* `uploaded`
* `processing`
* `indexed`
* `failed`

**Требования:**

* Контроль корректности переходов состояний
* Обновление статусов:
  * после загрузки
  * после обработки ingestion сервисом
* Логирование изменений

---

### 7.5 Взаимодействие с API Gateway

**Приоритет:** Критический

**Требования:**

* Все клиентские запросы проходят через API Gateway
* Gateway:
  * валидирует JWT
  * извлекает `user_id`, `organization_id`
  * передаёт их в gRPC metadata
* Document Service:
  * не выполняет аутентификацию
  * доверяет Gateway
  * использует переданные данные для tenant-изоляции

---

### 7.6 Взаимодействие с Ingestion Service

**Приоритет:** Высокий

**Требования:**

* Публикация события после загрузки документа
* Использование message broker (Kafka / RabbitMQ / NATS)
* Передача:
  * `document_id`
  * `organization_id`
  * `s3_path`
* Retry механизм

---

### 7.7 gRPC API

**Приоритет:** Критический

**Методы:**

* `CreateDocument`
  * создаёт документ
  * возвращает `document_id` и `upload_url`

* `ListDocuments`
  * возвращает список документов

* `GetDocument`
  * возвращает метаданные документа

* `DeleteDocument`
  * удаляет документ (логически/физически)

* `UpdateDocumentStatus`
  * обновление статуса (используется ingestion сервисом)

---

### 7.8 Удаление данных

**Приоритет:** Высокий

**Требования:**

* Удаление:
  * метаданных
  * файла из S3
  * embeddings (асинхронно)
* Гарантия tenant-изоляции
* Поддержка soft delete (опционально)

---

### 7.9 Безопасность

**Приоритет:** Критический

**Требования:**

* Использование только внутренних gRPC вызовов
* Проверка `organization_id` в каждом методе
* Исключение доступа к чужим данным
* Защита от неконсистентных запросов

---

## 3.8 Frontend

### 8.1 Пользовательский интерфейс

Требования:

* Авторизация
* Загрузка документов
* Дашборд
* Чат-интерфейс

---

# 4. НЕФУНКЦИОНАЛЬНЫЕ ТРЕБОВАНИЯ

## 4.1 Производительность

* Время ответа запроса: ≤ 1 сек
* Latency RAG pipeline: ≤ 1.5 сек
* Поддержка batch embedding

---

## 4.2 Надежность

* Репликация сервисов
* Автоматический failover
* Асинхронные очереди

---

## 4.3 Безопасность

* JWT аутентификация
* Изоляция данных по organization_id
* Шифрование данных
* Безопасное хранение файлов

---

## 4.4 Масштабируемость

* Горизонтальное масштабирование сервисов
* Масштабирование векторной БД
* Поддержка multiple LLM providers

---

## 4.5 Сопровождаемость

* Логирование
* Мониторинг
* CI/CD

---

# 5. ТЕХНИЧЕСКИЕ ОГРАНИЧЕНИЯ

## 5.1 Ограничения

* Использование S3 для хранения
* Использование Qdrant для embeddings
* Все сервисы tenant-aware

## 5.2 Предположения

* Наличие облачной инфраструктуры
* Доступ к LLM API
* Стабильная сеть

---

# 6. СИСТЕМНЫЕ ТРЕБОВАНИЯ

## 6.1 Серверная инфраструктура

* S3-хранилище
* Векторная БД (Qdrant)
* PostgreSQL

---

## 6.2 Программное обеспечение

* Go / Python
* gRPC
* PostgreSQL
* Qdrant
* React / Next.js

---

# 7. АРХИТЕКТУРА СИСТЕМЫ

## 7.1 Микросервисы

* Auth Service
* API Gateway
* Ingestion Service
* Embedding Service
* Document Service
* Retrieval Service
* LLM Router
* Frontend

---

## 7.2 Коммуникация

* REST API
* gRPC
* Event-driven pipeline
* Kafka

---

## 7.3 Потоки данных

### Поток обработки документа

S3 → Ingestion → Embeddings → Qdrant

### Поток запроса

User → API → Retrieval → LLM → Ответ

---

# 8. МОДЕЛИ МАШИННОГО ОБУЧЕНИЯ

## 8.1 Embeddings

* Генерация векторных представлений текста

## 8.2 LLM

* Генерация ответов
* Поддержка нескольких моделей

---

# 9. ИНТЕРФЕЙСЫ

## 9.1 REST API

* POST /documents
* POST /query
* GET /models

---

## 9.2 Frontend

* Dashboard
* Chat UI

---

# 10. ТЕСТИРОВАНИЕ

## 10.1 Виды тестирования

* Unit
* Integration
* Load testing

---

## 10.2 Критерии приемки

* Работает RAG pipeline
* Обеспечена изоляция данных
* API соответствует спецификации

---

# 11. ЭТАПЫ РАЗРАБОТКИ

1. Проектирование
2. Разработка сервисов
3. Интеграция
4. Тестирование
5. Деплой

---

# 12. РИСКИ

* Нарушение изоляции данных
* Высокая latency LLM
* Проблемы масштабирования

---

# 13. ЭКСПЛУАТАЦИЯ

* Мониторинг
* Обновления
* Поддержка пользователей

---

# 14. СТОИМОСТЬ

(Требует отдельной оценки)

---

# 15. ГЛОССАРИЙ

* Embedding — векторное представление текста
* Chunk — часть документа
* Retrieval — поиск релевантных данных
* Attribution — указание источников

---

# 16. ЗАКЛЮЧЕНИЕ

Система RAG-as-a-Service обеспечивает масштабируемую платформу для интеллектуальной работы с документами, включая поиск и генерацию ответов, с полной изоляцией данных между организациями .

---


# Диаграмма контекста
<img width="465" height="424" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/bcf9d79f-f5d8-4027-91a1-68b5110e0b14" />


# Диаграмма компонентов
<img width="821" height="958" alt="mermaid-diagram (1)" src="https://github.com/user-attachments/assets/81b92b38-7334-4c5d-a7d9-aeea8553e3b7" />
