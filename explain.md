# Как работает проект

Этот проект - демонстрационное Spring Boot приложение для изучения Spring AI, tool-calling, простых agentic workflow, RAG через pgvector и MCP server.

Главная точка входа для пользовательских HTTP-запросов - `SupportAssistantController`.

Основные роли приложения:

- REST API: принимает запросы на `/ai/support/...`.
- OpenAI client: само вызывает LLM через Spring AI `ChatClient`.
- MCP server: поднимает MCP server через `spring-ai-starter-mcp-server-webmvc`.
- Не MCP client: в проекте нет `spring-ai-starter-mcp-client-*` и настроек `spring.ai.mcp.client.*`.

Отдельный MCP client, судя по `project-structure.md`, находится в другом проекте `mcpClient` и может использовать это приложение как MCP server.

## Основные endpoint'ы

Контроллер находится в `SupportAssistantController`.

Доступные группы запросов:

- `/ai/support/{user}/orchestrator/simple`
- `/ai/support/{user}/orchestrator/aibased`
- `/ai/support/{user}/orchestrator/toolbased`
- `/ai/support/{user}/chain/seq`
- `/ai/support/{user}/chain/par`
- `/ai/support/{user}/chain/rep`

Пример:

```bash
http GET http://localhost:8080/ai/support/12/orchestrator/simple question=="Найди название чего-то сладенького"
```

Параметр `{user}` используется как `customerId` для tools через `ToolContext`. Например, `OrderAgentImpl` достает его из context при создании заказа.

## Конфигурация AI и MCP

Основная AI-конфигурация находится в `AIConfig`.

Там создаются:

- `openAiChatClient` - клиент Spring AI для общения с OpenAI.
- `openAiChatModel1` - модель OpenAI.
- `ToolCallingManager` - менеджер исполнения tool calls.
- `ToolExecutionExceptionProcessor` - обработчик ошибок tools.
- `ToolCallbackProvider` - provider для MCP server tools.

В `application.properties` заданы:

```properties
spring.ai.openai.api-key=${OPEN_AI_KEY}
spring.ai.openai.chat.options.model=gpt-4o
spring.ai.mcp.server.annotation-scanner.enabled= true
spring.ai.mcp.server.type=sync
```

Важно: `ChatClient` здесь сконфигурирован через `OpenAiChatModel.builder()`. При этом model name задан в properties, но в ручном builder-коде он явно не прокидывается. Стоит проверять фактическую модель в логах/запросах, если это критично.

MCP server включен зависимостью:

```xml
<artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
```

И настройкой:

```properties
spring.ai.mcp.server.annotation-scanner.enabled= true
```

Это позволяет Spring AI находить методы с `@Tool` и экспонировать их как MCP tools для внешнего MCP client.

## Что такое tools в этом проекте

Tool - это обычный Java method, помеченный аннотацией `@Tool`. LLM может попросить приложение вызвать такой метод, если tool передан в текущий запрос.

Главные tools:

- `ProductAgent.handle(query)` - ищет продукты в vector store по описанию.
- `OrderAgentImpl.handle(productName, quantity, ToolContext)` - создает заказ.
- `DocumentLoaderAgent.loadProductsJsonFile()` - загружает продукты из `products-data.json`.
- `DocumentLoaderAgent.loadCustomerJsonFile()` - загружает покупателей из `customer-data.json`.
- `FindToolNameAgent.findToolAgent(query, toolEnum)` - помогает выбрать группу tools.
- `FallbackAgentImpl.handle()` - fallback-ответ, но в текущих основных оркестраторах практически не используется.

Tools не загружаются в LLM при старте приложения.

При старте Spring только создает beans и tool definitions. В LLM tools попадают только во время конкретного вызова `ChatClient`, когда код явно передает:

```java
.tools(...)
```

или кладет callbacks в:

```java
ToolCallingChatOptions.builder()
    .toolCallbacks(...)
```

## Поток запроса через simple orchestrator

Endpoint:

```text
GET /ai/support/{user}/orchestrator/simple?question=...
```

Flow:

```text
HTTP request
 -> SupportAssistantController
 -> SimpleOrchestrator.orchestrate(userId, query)
 -> создает ToolContext с CUSTOMER_ID
 -> создает Prompt
 -> передает tools в ChatClient
 -> LLM решает, нужен ли tool
 -> Spring AI вызывает выбранный tool
 -> результат tool возвращается в LLM
 -> LLM формирует финальный ответ
```

Важный нюанс: `SimpleOrchestrator` сейчас передает все основные tools:

```java
simpleToolSelector.getAllTools(query)
```

А `getAllTools()` возвращает:

```java
orderAgent, documentLoaderAgent, productAgent
```

То есть в `simple` варианте нет предварительного сужения tools. LLM сама видит все переданные tools и решает, какой вызвать.

Хотя в `SimpleToolSelector` есть метод `determineAllowedTools(query)`, он в `SimpleOrchestrator.orchestrate()` сейчас не используется.

## Поток через aiBased orchestrator

Endpoint:

```text
GET /ai/support/{user}/orchestrator/aibased?question=...
```

Здесь используется двухшаговая логика.

Шаг 1: `AiBasedToolSelector` делает дополнительный LLM-запрос:

```text
Определи с чем связан запрос
1. Заказ продукта пользователем
2. Настройка базы данных
3. Ничего
Верни только цифру 1, 2, или 3.
```

Шаг 2: по ответу выбирается набор tools:

- `1` -> `orderAgent`, `productAgent`
- `2` -> `documentLoaderAgent`
- другое -> пустой список tools

Шаг 3: основной пользовательский запрос отправляется в LLM уже только с выбранным набором tools.

Это полезно, когда tools много и не хочется отдавать модели весь список на каждый запрос.

Плюсы:

- меньше tool definitions в основном запросе;
- меньше шанс, что модель вызовет опасный или нерелевантный tool;
- дешевле по токенам, если tools много.

Минусы:

- появляется дополнительный LLM-запрос;
- если классификация ошиблась, нужный tool не попадет в основной запрос;
- ответ парсится через `contains("1")`, `contains("2")`, что хрупко.

## Поток через toolBased orchestrator

Endpoint:

```text
GET /ai/support/{user}/orchestrator/toolbased?question=...
```

Идея похожа на `aiBased`, но классификация делается через отдельный tool:

```java
.tools(findToolNameAgent)
```

`FindToolNameAgent` принимает:

```java
String query,
ToolEnum toolEnum
```

`ToolEnum` содержит варианты:

- `PRODUCT` -> `1`
- `DB_LOADER` -> `2`
- `NOTHING` -> `3`

Дальше `ToolBasedToolSelector` смотрит ответ:

- если есть `1`, возвращает `orderAgent`, `productAgent`;
- если есть `2`, возвращает `documentLoaderAgent`;
- иначе возвращает пустой список.

Этот вариант показывает хитрость: tool может использоваться не для ответа пользователю, а для выбора других tools.

## Как работает RAG по продуктам

RAG-часть завязана на `ProductAgent`.

Когда LLM вызывает:

```java
ProductAgent.handle(query)
```

код идет в:

```java
productVectorRepository.searchProductsByDescription(query, 2)
```

Дальше используется `VectorStore`.

Загрузка данных происходит через:

```java
DocumentLoaderAgent.loadProductsJsonFile()
```

Он читает:

```text
src/main/resources/products-data.json
```

Потом `ProductServiceImpl.createProducts()`:

- сохраняет продукты в обычную БД;
- мапит их в `ProductVectorDto`;
- сохраняет в vector store.

В vector store создаются документы так:

```java
new Document(dto.getDescription(), dto.toMap())
```

Это значит:

- embedding строится по `description`;
- `id`, `name`, `type` кладутся в metadata;
- `price` и `stock` в vector metadata не кладутся;
- chunking вручную не выполняется.

Один продукт = один `Document`.

Пример данных:

```json
{
  "name": "Cheesecake",
  "description": "Creamy cheesecake slice",
  "price": 4.00,
  "stock": 40,
  "type": "food"
}
```

В vector store searchable text будет:

```text
Creamy cheesecake slice
```

Metadata будет примерно:

```text
id, name, type
```

Если пользователь спрашивает:

```text
Найди название чего-то сладенького
```

то в vector search пойдет не весь HTTP-запрос, а аргумент, который LLM передаст в `ProductAgent.handle(query)`. Обычно это будет исходная фраза или ее переформулировка.

После vector search:

```text
vector store -> список похожих документов -> ProductAgent -> LLM -> финальный ответ
```

LLM получает результат tool и сама формирует финальный человеческий ответ.

## Важный нюанс в vector search

В `ProductVectorRepositoryImpl.searchProductsByDescription()` создается `SearchRequest`:

```java
SearchRequest searchRequest = SearchRequest.builder()
    .query(description)
    .topK(limit)
    .similarityThreshold(0.7F)
    .build();
```

Но он не используется.

Фактически вызывается:

```java
vectorStore.similaritySearch(description);
```

Поэтому параметры `limit` и `similarityThreshold` сейчас не влияют на поиск. Чтобы они работали, нужно вызывать поиск через `SearchRequest`, если используемая версия Spring AI это поддерживает:

```java
vectorStore.similaritySearch(searchRequest);
```

Это одна из главных текущих ловушек проекта.

## ToolContext и user id

В оркестраторах создается:

```java
Map<String, Object> toolContext = Map.of(CUSTOMER_ID, userId);
```

Потом этот context передается в ChatClient:

```java
.toolContext(toolContext)
```

`OrderAgentImpl` принимает `ToolContext context` и достает оттуда customer id:

```java
Integer customerId = (Integer) map.get(CUSTOMER_ID);
```

Это правильный прием: не нужно просить LLM передавать `customerId` как обычный аргумент. Такие технические данные лучше прокидывать через `ToolContext`.

## ChatOptions и ручное управление tools

В `SimpleOrchestrator` есть экспериментальный метод:

```java
orchestrateThroughtChatOptions(...)
```

Там tools передаются не через:

```java
.tools(...)
```

а через:

```java
ChatOptions chatOptions = ToolCallingChatOptions.builder()
    .internalToolExecutionEnabled(false)
    .toolCallbacks(simpleToolSelector.getAllToolCallbackByClass(query))
    .toolContext(toolContext)
    .build();

Prompt prompt = new Prompt(query2, chatOptions);
```

`ChatOptions` - это runtime-настройки конкретного prompt.

Через них можно задать:

- доступные tools;
- tool context;
- model options;
- temperature;
- max tokens;
- режим автоматического или ручного выполнения tools.

Самый важный флаг:

```java
.internalToolExecutionEnabled(false)
```

Он означает: Spring AI не должен автоматически выполнять tool calls. Модель может вернуть tool call, но приложение само должно:

- прочитать `ChatResponse`;
- проверить `hasToolCalls()`;
- вызвать нужные tools через `ToolCallingManager`;
- отправить результаты обратно модели.

В текущем коде этот ручной цикл не реализован. Поэтому `orchestrateThroughtChatOptions()` выглядит как эксперимент, а не полноценный рабочий flow.

В обычных методах `orchestrate()` используется автоматическое выполнение tools.

## ToolExecutionEligibilityPredicate

В проекте есть свой `ToolExecutionEligibilityPredicateImpl`:

```java
return ToolCallingChatOptions.isInternalToolExecutionEnabled(promptOptions)
    && chatResponse != null
    && chatResponse.hasToolCalls();
```

Он говорит Spring AI, когда можно выполнять tool calls автоматически.

Смысл:

- если internal tool execution включен;
- и ответ модели содержит tool calls;
- тогда Spring AI может исполнить tools.

Если `internalToolExecutionEnabled(false)`, автоматическое исполнение tools отключается.

## SimpleAiChain

`SimpleAiChain` не про MCP и не про tools. Это демонстрация agentic workflow с несколькими LLM-вызовами.

Есть три режима.

### Sequential

```java
handleSequentially(mainHero)
```

История переписывается несколько раз подряд. Каждый следующий LLM-запрос получает результат предыдущего.

Flow:

```text
исходная история
 -> добавить главного героя
 -> сделать грустной
 -> перенести на вокзал
 -> добавить рыбалку
 -> итоговая история
```

Это настоящая цепочка, где каждый шаг зависит от предыдущего.

### Parallel

```java
handleParallel(mainHero)
```

Каждый факт применяется параллельно к исходной истории. Результаты не объединяются.

Flow:

```text
исходная история -> версия с героем
исходная история -> веселая версия
исходная история -> версия на елке
исходная история -> версия с пением
```

Метод возвращает список отдельных результатов.

### Repeated

```java
handleRepeated()
```

Цикл просит LLM добавлять героев, пока ответ не содержит:

```text
Получилось 5 героев
```

Есть защита:

```java
if (counter >= 12) {
    throw new RuntimeException("Запрос зациклился");
}
```

Нюанс: endpoint принимает `mainHero`, но `handleRepeated()` его не использует.

## Нюансы и хитрости проекта

### 1. Tools передаются только на конкретный запрос

LLM не знает tools постоянно. Каждый вызов `ChatClient` получает свой набор tools.

Это позволяет делать разные стратегии:

- отдать все tools;
- отдать только tools по категории;
- сначала классифицировать запрос;
- использовать tool для выбора другого tool.

### 2. Tool selection можно делать без LLM

`SimpleToolSelector.determineAllowedTools()` показывает keyword-based подход:

```java
if (q.contains("заказ") || q.contains("order") || q.contains("купить")) {
    return orderAgent, productAgent;
}
```

Такой подход дешевле и предсказуемее, но хуже понимает естественный язык.

### 3. Tool selection можно делать через LLM

`AiBasedToolSelector` делает дополнительный LLM-запрос и просит вернуть категорию.

Это гибче, но дороже и может ошибаться.

### 4. Tool selection можно делать через отдельный tool

`ToolBasedToolSelector` передает модели только `FindToolNameAgent`, а затем по его результату выбирает основной набор tools.

Это демонстрирует многошаговую оркестрацию tools.

### 5. Не отдавай опасные tools без необходимости

`DocumentLoaderAgent` изменяет БД. В `simple` orchestration он доступен модели вместе с другими tools.

Для production-подхода лучше не передавать такие tools на обычные пользовательские запросы.

### 6. Tool descriptions критичны

LLM выбирает tool по имени, описанию и параметрам. Если description плохой или двусмысленный, модель может выбрать не тот tool.

Например:

```java
@Tool(description = "возвращает описание продуктов по запросу", name = "ProductAgent")
```

Это описание помогает модели понять, что tool подходит для поиска продуктов.

### 7. Имена tools должны быть уникальными

В `OrderAgentImpl` есть комментарий:

```java
// Если имя одинаковое, то будет ошибка
```

Если два tool имеют одно имя, Spring AI/OpenAI tool-calling не сможет корректно различать их.

### 8. ToolContext лучше для технических данных

`customerId` правильно передается через `ToolContext`, а не через prompt.

Так меньше шанс, что LLM:

- забудет id;
- изменит id;
- попросит пользователя его указать;
- передаст неправильный id в tool.

### 9. Ответы classifier'ов сейчас парсятся хрупко

Код проверяет:

```java
response.contains("1")
```

Если модель вернет `1. Заказ продукта`, это сработает.

Но если модель вернет лишний текст с несколькими цифрами, можно выбрать неправильную ветку.

Для надежности лучше просить JSON или использовать enum/structured output.

### 10. FallbackAgent почти не используется

`FallBackAgent` есть, но основные selector'ы часто возвращают пустой массив, а не fallback tool.

Если хочется, чтобы LLM всегда могла вызвать fallback, его нужно явно включать в выбранный набор tools.

### 11. `SearchRequest` в vector search сейчас не работает

Код создает `SearchRequest`, но вызывает другой overload. Поэтому `topK` и threshold не применяются.

### 12. RAG здесь сделан через tool

Это не классический RAG-advisor pipeline, где documents автоматически добавляются в prompt.

Здесь RAG выглядит так:

```text
LLM -> вызывает ProductAgent tool -> tool ищет в vector store -> LLM получает результат tool -> отвечает
```

То есть retrieval - это действие tool, а не отдельный advisor.

### 13. `SimpleOrchestrator` и `AiBasedOrchestrator` отличаются главным образом набором tools

`SimpleOrchestrator`:

```text
один LLM-запрос с полным набором tools
```

`AiBasedOrchestrator`:

```text
LLM-запрос для классификации -> основной LLM-запрос с отфильтрованными tools
```

`ToolBasedOrchestrator`:

```text
LLM + findToolAgent -> основной LLM-запрос с отфильтрованными tools
```

## Что делает приложение в целом

Приложение имитирует support assistant для кафе/магазина.

Оно умеет:

- загружать продукты из JSON в БД и vector store;
- загружать покупателей из JSON в БД;
- искать продукты по смысловому описанию;
- создавать заказ для пользователя;
- отвечать через LLM;
- демонстрировать разные стратегии выбора tools;
- демонстрировать простые LLM chains;
- работать как MCP server для внешнего MCP client.

Архитектурно это учебный monolith:

```text
Controller
 -> Orchestrator / Chain
 -> Agent as Tool
 -> Service
 -> Repository
 -> Database / VectorStore
```

Главная идея проекта: показать, что tools - это не магия внутри LLM, а обычные Java methods, которые приложение регистрирует, выбирает и передает модели в конкретном запросе. LLM только решает, какой из доступных ей tools вызвать и с какими аргументами. Сам tool выполняется на стороне Java-приложения.

