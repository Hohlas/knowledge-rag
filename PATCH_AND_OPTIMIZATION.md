# Инструкция по оптимизации и исправлению Knowledge-RAG MCP Server

В этом документе описаны изменения, внесенные в исходный код `knowledge-rag` для обеспечения стабильности протокола MCP, поддержки специализированных форматов (MQL4/MQL5/Jupyter) и реализации надежной системы исключений.

---

## 1. Исправление стабильности MCP (stdout protection)
**Файлы:** `mcp_server/config.py`, `mcp_server/ingestion.py`, `mcp_server/server.py`

MCP-протокол использует `stdout` для передачи JSON-сообщений. Любой диагностический `print()` в сервере может повредить stdio-поток и сорвать handshake.

**Актуальное решение:** локально перенаправлять `print()` в `stderr` только в модулях `knowledge-rag`, где есть диагностический вывод. Глобальная подмена `builtins.print` в `mcp_server/__init__.py` удалена: она слишком широкая, затрагивает код MCP SDK и сторонних библиотек и может приводить к зависанию stdio-handshake.

```python
import builtins
import sys

def print(*args, **kwargs):
    kwargs.setdefault("file", sys.stderr)
    return builtins.print(*args, **kwargs)
```

**Важно:** не возвращать глобальный monkey-patch `builtins.print = ...` в `__init__.py`. Защиту stdout нужно держать локальной и явной.

---

## 2. Поддержка MetaTrader и Jupyter
**Файл:** `mcp_server/ingestion.py`

Добавлена поддержка расширений `.mqh`, `.mq4` (как текст) и `.ipynb` (как JSON) в таблицу парсеров.

---

## 3. Робастная система исключений (exclude_patterns)
**Файл:** `mcp_server/ingestion.py` и `mcp_server/config.py`

Реализована глубокая проверка компонентов пути для надежного исключения папок типа `.venv`, `.git` или `MQL5`, независимо от их уровня вложенности.

### Изменение в `config.py`:
Исправлена функция `_get_top`, чтобы она корректно загружала списки (List) из YAML-конфига.

### Изменение в `ingestion.py`:
Используется `fnmatch` и разделение пути на части (`Path.parts`) для сопоставления фильтров с реальной структурой директорий.

---

## 4. Защита от лимитов Inotify (Linux)
**Файл:** `mcp_server/server.py`

На Linux-системах сервер теперь не падает при достижении лимита `fs.inotify.max_user_instances`. 

**Решение:** Инициализация `Observer` обернута в `try-except`. Если лимит исчерпан, сервер выводит предупреждение и продолжает работу в режиме ручной индексации.

---

## 5. Постоянный кеш моделей эмбеддингов
**Файл:** `mcp_server/server.py` и `mcp_server/config.py`

Модели `fastembed` больше не хранятся в `/tmp`, где они могут быть удалены системой.

**Решение:** 
1. Добавлено поле `models_cache_dir` в конфиг (по умолчанию `models_cache/` в папке проекта).
2. Параметр `cache_dir` передается в конструктор `TextEmbedding`.

---

## 6. Оптимальный `config.yaml` для проекта

```yaml
paths:
  documents_dir: "/path/to/your/project"
  data_dir: "./data"
  models_cache_dir: "./models_cache"

documents:
  supported_formats:
    - .md
    - .txt
    - .py
    - .mqh
    - .mq4
    - .ipynb

exclude_patterns:
  - "**/MT/MQL5/**"
  - "**/MT/tester/**"
  - "**/MT/MQL4/Files/**"
  - "**/MT/MQL4/Indicators/**"
  - "**/MT/MQL4/Libraries/**"
  - "**/MT/MQL4/Logs/**"
  - "**/MT/MQL4/Profiles/**"
  - "**/MT/MQL4/Scripts/**"
  - "**/MT/MQL4/Trash/**"
  - "**/.venv/**"
  - "**/.git/**"
  - "**/.**"
```
---

## 7. Codex/SoSimple восстановление после сбоя MCP

### Что было проверено

- `codex mcp list` показывает только регистрацию MCP-сервера и не является функциональной проверкой.
- `Auth: Unsupported` для stdio MCP — нормальный статус, а не ошибка.
- Функциональная проверка — вызов инструмента, например:

```text
search_knowledge("entry_path_v1_quantile", hybrid_alpha=0.0, max_results=5)
search_knowledge("triple barrier", hybrid_alpha=0.3, max_results=5)
```

Рабочий признак: MCP-клиент запрашивает разрешение на запуск `search_knowledge`, а ответ содержит `status: success` и результаты.

### Текущая схема установки для Codex

Codex запускает MCP через editable-установку локального исходника:

```bash
/home/hohla/knowledge-rag/venv/bin/python -m pip install --no-deps -e /home/hohla/git/knowledge-rag
```

В `~/.codex/config.toml` сервер должен оставаться stdio-сервером:

```toml
[mcp_servers.knowledge-rag]
command = "/home/hohla/knowledge-rag/venv/bin/knowledge-rag"

[mcp_servers.knowledge-rag.env]
KNOWLEDGE_RAG_DIR = "/home/hohla/knowledge-rag"
```

### SoSimple runtime-конфиг

Активный `config.yaml` для этой машины указывает:

```yaml
paths:
  documents_dir: "/home/hohla/git/SoSimple"
  data_dir: "/home/hohla/git/SoSimple/.knowledge-rag-data"
  models_cache_dir: "./models_cache"
```

Причина переноса `data_dir` в `/home/hohla/git/SoSimple/.knowledge-rag-data`: Codex sandbox стабильно разрешает запись внутри workspace, а запись в `/home/hohla/knowledge-rag/data` из MCP-сессии может быть недоступна или зависеть от sandbox-настроек.

`.knowledge-rag-data/` — runtime-хранилище индекса Chroma/BM25. Оно нужно для работы текущего конфига, но не должно попадать в git. В репозитории SoSimple оно должно быть добавлено в `.gitignore`.

Для воспроизводимости SoSimple-конфиг также сохранен как `presets/sosimple.yaml`. Активный `config.yaml` по-прежнему считается локальным файлом и игнорируется стандартным `.gitignore` проекта `knowledge-rag`.

### Чего не делать без необходимости

- Не копировать бинарную ChromaDB/HNSW-базу между директориями: это уже приводило к segfault при открытии базы. Если нужен перенос — лучше выполнить reindex.
- Не переустанавливать PyPI `knowledge-rag` поверх локального editable-патча. Это затирает локальные исправления.
- Не откатывать `mcp`/`fastembed` наугад. Сначала проверять минимальным воспроизводимым тестом в отдельном venv.
- Не считать `codex mcp list` доказательством работоспособности. Проверять именно `search_knowledge`.
---

## 8. Upstream status after reinstalling official v3.5.2

As of the official `knowledge-rag==3.5.2` release, the production setup should prefer the upstream PyPI package over the local editable patch. The upstream package already includes the main fixes that were needed for Codex/SoSimple:

- scoped stdout protection for MCP stdio;
- persistent `models_cache_dir`;
- `documents.exclude_patterns` with a shared exclusion helper;
- proper `.ipynb` parsing that extracts markdown/code cell sources only;
- `.mqh` and `.mq4` parser support;
- inotify watcher fallback.

The active SoSimple config was migrated to the upstream format by moving `exclude_patterns` under `documents:`. The reproducible preset is `presets/sosimple.yaml`.

Keep the local runtime config at `/home/hohla/knowledge-rag/config.yaml`; keep the runtime index at `/home/hohla/git/SoSimple/.knowledge-rag-data`; do not commit either runtime data or ChromaDB files.

