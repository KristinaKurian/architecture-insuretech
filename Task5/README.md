# Task5. Проектирование GraphQL API

## Цель

Перевести API сервиса `client-info` с REST на GraphQL, чтобы потребители могли получать только необходимые данные клиента в одном запросе.

Исходный REST API предоставляет отдельные ресурсы:

```text
GET /clients/{id}
GET /clients/{id}/documents
GET /clients/{id}/relatives
```

При использовании REST сценарий, которому одновременно нужны основные данные клиента, документы и родственники, требует нескольких HTTP-запросов.

GraphQL позволяет получить эти данные одним запросом и при этом выбрать только необходимые поля.

## 1. Анализ существующего REST API

### Ресурс Client

Endpoint:

```text
GET /clients/{id}
```

Поля:

- `id`;
- `name`;
- `age`.

### Ресурс Document

Endpoint:

```text
GET /clients/{id}/documents
```

Поля:

- `id`;
- `type`;
- `number`;
- `issueDate`;
- `expiryDate`.

Связь:

```text
Client 1 → N Document
```

### Ресурс Relative

Endpoint:

```text
GET /clients/{id}/relatives
```

Поля:

- `id`;
- `relationType`;
- `name`;
- `age`.

Связь:

```text
Client 1 → N Relative
```

## 2. GraphQL-модель

Корневой объект — `Client`.

Связанные данные становятся полями клиента:

```text
Client
├── id
├── name
├── age
├── documents
└── relatives
```

Таким образом, вместо нескольких REST endpoint потребитель использует один GraphQL query и сам определяет необходимый набор данных.

## 3. Query

Для операций из исходного Swagger достаточно одного корневого запроса:

```graphql
type Query {
  client(id: ID!): Client
}
```

Он покрывает все три REST-операции:

| REST | GraphQL |
|---|---|
| `GET /clients/{id}` | `client(id) { ... }` |
| `GET /clients/{id}/documents` | `client(id) { documents { ... } }` |
| `GET /clients/{id}/relatives` | `client(id) { relatives { ... } }` |

## 4. Примеры использования

### Только основные данные клиента

```graphql
query {
  client(id: "123") {
    id
    name
    age
  }
}
```

### Только документы

```graphql
query {
  client(id: "123") {
    documents {
      id
      type
      number
      expiryDate
    }
  }
}
```

### Клиент, документы и родственники одним запросом

```graphql
query {
  client(id: "123") {
    id
    name

    documents {
      type
      number
    }

    relatives {
      relationType
      name
    }
  }
}
```

REST в таком сценарии потребовал бы несколько отдельных HTTP-запросов.

## 5. Почему GraphQL подходит для client-info

Карточка клиента может содержать большое количество атрибутов, а разным сценариям требуется разный набор данных.

GraphQL позволяет:

- запрашивать только необходимые поля;
- получать связанные сущности одним запросом;
- уменьшить количество HTTP-вызовов;
- избежать передачи всей клиентской карточки;
- сохранить единый API при появлении новых полей и связанных объектов.

При расширении модели новые данные можно добавлять в `Client` как дополнительные поля без создания отдельного REST endpoint для каждого варианта использования.

## Файлы

- `schema.graphql` — GraphQL SDL-схема;