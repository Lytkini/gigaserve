# GigaServe 🦜️🏓 = LangServe + GigaChat

[![Release Notes](https://img.shields.io/github/release/langchain-ai/langserve)](https://github.com/ai-forever/gigaserve/releases)
[![Скачивания](https://static.pepy.tech/badge/langserve/month)](https://www.pepy.tech/projects/gigaserve)

🚩 We will be releasing a hosted version of LangServe for one-click deployments of LangChain applications. [Sign up here](https://airtable.com/app0hN6sd93QcKubv/shrAjst60xXa6quV2) to get on the waitlist.

## О GigaServe

`GigaServe = LangServe + GigaChain` helps developers deploy `GigaChain` [runnables and chains](https://python.langchain.com/docs/expression_language/) as a REST API.

С помощью библиотеки GigaServe вы можете реализовать REST API для предоставления доступа к runnable-интерфейсам и цепочкам GigaChain.

Библиотека GigaServe интегрирована с [FastAPI](https://fastapi.tiangolo.com/) и использует для валидации данных [Pydantic](https://docs.pydantic.dev/latest/).

## Особенности библиотеки

- Схемы ввода и вывода автоматически определяются на основе объекта GigaChain, применяются для каждого запроса к API и обеспечивают подробные сообщения об ошибках.
- Страница API-документации с JSONSchema и Swagger. 
- Эффективные эндпоинты `/invoke`, `/batch` и `/stream` с поддержкой множества одновременных запросов на одном сервере.
- Эндпоинт `/stream_log` для потоковой передачи всех или выбранных промежуточных шагов работы цепочки/агента.
- Интерактивная страница `/playground` с потоковым выводом и демонстрацией промежуточных шагов.
- Использование проверенных open-source библиотек Python таких, как FastAPI, Pydantic, uvloop и asyncio.
- Use the client SDK to call a LangServe server as if it was a Runnable running locally (or call the HTTP API directly)
- Возможность обращения к серверу GigaServe с помощью клиентского SDK, как если бы это был локальный runnable-интерфейс (или возможность обратиться напрямую к HTTP API).

### Ограничения

- Обратные вызовы клиента пока не поддерживаются для событий, происходящих на сервере
- При использовании Pydantic V2 документация OpenAPI не генерируется. Это связанно с тем, что Fast API не поддерживает [смешивание пространств имен pydantic v1 и v2](https://github.com/tiangolo/fastapi/issues/10360). Подробнее в разделе ниже.

## Безопасность

Уязвимость в версиях 0.0.13 - 0.0.15 — Интерактивная страница, доступная по адресу `/playground`, позволяет получить доступ к произвольным файлам на сервере. [Устранено в версии 0.0.16](https://github.com/langchain-ai/langserve/pull/98).

## GigaChain CLI 🛠️

Для быстрой настройки проекта GigaChain используйте актуальную версию GigaChain CLI:

```sh
gigachain app new ../path/to/directory
```


Вы можете установить актуальную версию CLI с помощью менеджера пакетов pip:

```sh
pip install -U gigachain-cli
```

## Примеры

Для быстрого старта GigaServe используйте [шаблоны GigaChain](https://github.com/ai-forever/gigachain/blob/master/templates/README.md).

Больше примеров шаблонов вы найдете в [соответствующей директории](https://github.com/ai-forever/gigaserve/tree/main/examples).

### Сервер

Пример ниже разворачивает модель чата OpenAI, модель чата Anthropic, и цепочку, которая генерирует шутку по заданной теме (`topic`) с помощью модели Anthropic.

```python
#!/usr/bin/env python
from fastapi import FastAPI
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import ChatAnthropic, ChatOpenAI
from langserve import add_routes


app = FastAPI(
  title="GigaChain Server",
  version="1.0",
  description="Простой API-сервер, использующий runnable-интерфейсы GigaChain",
)

add_routes(
    app,
    ChatOpenAI(),
    path="/openai",
)

add_routes(
    app,
    ChatAnthropic(),
    path="/anthropic",
)

model = ChatAnthropic()
prompt = ChatPromptTemplate.from_template("расскажи шутку о {topic}")
add_routes(
    app,
    prompt | model,
    path="/chain",
)

if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="localhost", port=8000)
```

### Документация

Сгенерированная OpenAPI-документация к серверу, развернутому с помощью предыдущего примера, доступна по адресу:

```sh
curl localhost:8000/docs
```

> [!NOTE]
> Обращение к адресу `localhost:8000` будет возвращать ошибку 404, пока вы не определите `@app.get("/")`.

### Клиент

Пример клиента на основе Python SDK:

```python
from langchain.schema import SystemMessage, HumanMessage
from langchain.prompts import ChatPromptTemplate
from langchain.schema.runnable import RunnableMap
from langserve import RemoteRunnable

openai = RemoteRunnable("http://localhost:8000/openai/")
anthropic = RemoteRunnable("http://localhost:8000/anthropic/")
joke_chain = RemoteRunnable("http://localhost:8000/chain/")

# Синхронный вызов

joke_chain.invoke({"topic": "попугаи"})

# Асинхронный вызов
await joke_chain.ainvoke({"topic": "попугаии"})

prompt = [
    SystemMessage(content='Веди себя как кошка или попугай.'),
    HumanMessage(content='Привет!')
]

# Поддержка astream
async for msg in anthropic.astream(prompt):
    print(msg, end="", flush=True)

prompt = ChatPromptTemplate.from_messages(
    [("system", "Расскажи мне длинную историю о {topic}")]
)

# Определение собственных цепочек
chain = prompt | RunnableMap({
    "openai": openai,
    "anthropic": anthropic,
})

chain.batch([{ "topic": "попугаи" }, { "topic": "кошки" }])
```

Пример клиента на TypeScript (для работы клиента требуется LangChain.js версии 0.0.166 или выше):

```typescript
import { RemoteRunnable } from "langchain/runnables/remote";

const chain = new RemoteRunnable({
  url: `http://localhost:8000/chain/invoke/`,
});
const result = await chain.invoke({
  topic: "кошки",
});
```

Клиент, использующий Python-библиотеку `requests`:

```python
import requests
response = requests.post(
    "http://localhost:8000/chain/invoke/",
    json={'input': {'topic': 'кошки'}}
)
response.json()
```

Использование cURL:

```sh
curl --location --request POST 'http://localhost:8000/chain/invoke/' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "input": {
            "topic": "кошки"
        }
    }'
```

## Эндпоинты

С помощью примера ниже вы можете добавить на сервер заранее подготовленные эндпоинты для работы с runnable-интерфейсами:

```python
...
add_routes(
  app,
  runnable,
  path="/my_runnable",
)
```

Список эндпоинтов:

- `POST /my_runnable/invoke` - вызвать runnable-интерфейс для единичных входных данных;
- `POST /my_runnable/batch` - вызвать runnable-интерфейс для набора входных данных;
- `POST /my_runnable/stream` - вызвать для единичных входных данных с потоковым выводом;
- `POST /my_runnable/stream_log` - вызвать для единичных входных данных с потоковым выводом, включая вывод промежуточных шагов по ходу генерации;
- `GET /my_runnable/input_schema` - получить json-схему входных данных runnable-интерфейса;
- `GET /my_runnable/output_schema` - получить json-схему выходных данных runnable-интерфейса;
- `GET /my_runnable/config_schema` - получить json-схему параметров конфигурации runnable-интерфейса;

## Интерактивная страница

Интерактивная страница доступна по адресу `/my_runnable/playground`. На ней представлен простой интерфейс, который позволяет настроить параметры runnable-интерфейса и сделать запрос к нему с потоковым выводом и демонстрацией промежуточных шагов.

## Установка

Для одновременной установки сервера и клиента используйте команду:

```sh
pip install "gigaserve[all]"
```

Для отдельной установки сервера используйте команду:

```sh
pip install "gigaserve[server]"
```

Для отдельной установки клиента используйте команду:

```sh
pip install "gigaserve[client]"
```

## Работа со устаревшими цепочками

GigaServe работает как с runnable-интерфейсами(написанным с помощью constructed via [LangChain Expression Language](https://python.langchain.com/docs/expression_language/)), так и с устаревшими цепочками (посредством наследования от `Chain`). Но следует учиывать, что некоторые входные схемы для устаревших цепочек могут быть некорректными или неполными и могут вызывать ошибки. Это можно предотвратить, если обновить аттрибут `input_schema` таких цепочек в LangChain.

## Добавление аутентификации

О том как добавить аутентификацию на свой сервер GigaServe вы может узнать в разделах документации FastAPI, посвященных [безопасности](https://fastapi.tiangolo.com/tutorial/security/) и [использованию связующего ПО](https://fastapi.tiangolo.com/tutorial/middleware/).

## Развертывание

### Развертывание на GCP

Для развертывания на GCP Cloud Run используйте команду:

```
gcloud run deploy [your-service-name] --source . --port 8001 --allow-unauthenticated --region us-central1 --set-env-vars=GIGACHAT_API_KEY=your_key
```

## Работа с Pydantic

GigaServe поддерживает Pydantic 2 с некоторыми ограничениями:

- При использовании Pydantic V2 документация OpenAPI не генерируется. Это связанно с тем, что Fast API не поддерживает [смешивание пространств имен pydantic v1 и v2](https://github.com/tiangolo/fastapi/issues/10360).
- GigaChain использует пространство имен версии v1 в Pydantic v2.

За исключением указанных ограничений эндпоинты API, интерактивная страница и другие функции должны работать корректно.

## Дополнительные возможности

### Работа с файлами

Обработка файлов это типичная задача для больших языковых моделей.
Существуют различные архитектурные подходы для решения этой задачи:

- Файл может быть загружен на сервер с помощью выделенного эндпоинта и обработан с помощью отдельного эндпоинта
- Файл может быть представлен как в виде бинарного значения, так и в виде ссылки, например, на содержимое файла, размещенное на хранилище s3.
- Эндпоинт может блокирующим или неблокирующим.
- Сложную обработку можно выделить в отдельный пул процессов. 

Выбирайте подход в соответсвии со своими задачами.

GigaServe пока не поддерживает тип `multipart/form-data`.
Для загрузки бинарного значения файла в runnable-интерфейс используйте кодировку base64.

[Пример загрузки файла закодированного с помощью base64](https://github.com/ai-forever/gigaserve/tree/main/examples/file_processing).

Вы также можете загружать файлы с помощью ссылок (например, на хранилище s3) или загружать их на отдельный эндпоинт как `multipart/form-data`

### Настраиваемые типы входных и выходных данных

Типы входных и выходных данных определяются для всех runnable-интерфейсов. Они доступны в аттрибутах `input_schema` и `output_schema`. GigaServe использует эти типы для валидации данных и генерации документации.

Вы можете переопределить наследованные типы с помощью метода `with_types`.

Общий пример работы с типами:

```python
from typing import Any

from fastapi import FastAPI
from langchain.schema.runnable import RunnableLambda

app = FastAPI()


def func(x: Any) -> int:
    """Ошибочно заданная функция, которая принимает любые данные, хотя должна принимать int."""
    return x + 1


runnable = RunnableLambda(func).with_types(
    input_schema=int,
)

add_routes(app, runnable)
```

### Пользовательские типы

Для десериализации данных в виде pydantic-модели, а не `dict`, унаследуйтесь от `CustomUserType`. При наследовании от этого типа сервер будет не будет преобразовывать данные в `dict`, а будет сохранять их как pydantic-модель.

В настоящее время этот тип работает только на стороне сервера и определяет поведение при декодировании данных.

```python
from fastapi import FastAPI
from langchain.schema.runnable import RunnableLambda

from langserve import add_routes
from langserve.schema import CustomUserType

app = FastAPI()


class Foo(CustomUserType):
    bar: int


def func(foo: Foo) -> int:
    """Пример функции, которая ожидает тип Foo, представленный в виде моде pydantic model"""
    assert isinstance(foo, Foo)
    return foo.bar

# Обратите внимание, что входные и выходные типы наследуются автоматически!
# Вам не нужно их указывать
# runnable = RunnableLambda(func).with_types( # <-- Не нужно в данном случае
#     input_schema=Foo,
#     output_schema=int,
# 
add_routes(app, RunnableLambda(func), path="/foo")
```

### Виджеты интерактивной страницы

На интерактивной странице вы можете создавать различные виджеты, демонстрирующие работу runnable-интерфейсов вашего бекенда.

- Виджет задается на уровне поля и поставляется как часть JSON-схемы вводного типа.
- Виджет должен содержать ключ `type`, значением которого является один из известного списка виджетов
- Другие ключи виджета будут связаны со значениями, описывающими пути в JSON-объекте

Общая схема:

```typescript
type JsonPath = number | string | (number | string)[];
type NameSpacedPath = { title: string; path: JsonPath }; // title используется для имитации json-схемы,но можно использовать namespace
type OneOfPath = { oneOf: JsonPath[] };

type Widget = {
    type: string // Какой-то хорошо известный тип, например, base64file, chat и др.
    [key: string]: JsonPath | NameSpacedPath | OneOfPath;
};
```


#### Виджет загрузки файла

Виджет позволяет загружать файлы в интерфейсе интерактивной страницы. Работает для файлов, представленных в виде строки, закодированной в base64.

Фрагмент примера:

```python
try:
    from pydantic.v1 import Field
except ImportError:
    from pydantic import Field

from langserve import CustomUserType


# ВНИМАНИЕ: Наследуйтесь от CustomUserType, а не от BaseModel. В противном случае
#            сервер декодирует данные в dict, а не модель pydantic.
class FileProcessingRequest(CustomUserType):
    """Request including a base64 encoded file."""

    # Дополнительное поле используется, чтобы задать виджет в интерфейсе интерактивной страницы.
    file: str = Field(..., extra={"widget": {"type": "base64file"}})
    num_chars: int = 100

```

> [!NOTE]
> [Подробный пример загрузки файла](https://github.com/ai-forever/gigaserve/tree/main/examples/file_processing).