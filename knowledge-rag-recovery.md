# Knowledge-RAG Recovery Runbook

Runbook для агента, который должен восстановить `knowledge-rag`, если в чате появляются ошибки вида:

- `tool call failed for knowledge-rag/reindex_documents`
- `tool call failed for knowledge-rag/search_knowledge`
- `Caused by: Transport closed`

Этот документ рассчитан на новую машину и новый чат. Выполняй шаги по порядку.

## Что обычно ломается

В SoSimple было подтверждено два разных класса проблемы:

1. MCP запускает `knowledge-rag`, но сервер падает ещё до обработки tool call.
2. `knowledge-rag` читает неправильный config и индексирует весь репозиторий, включая `.venv`, после чего:
   - reindex идёт слишком долго;
   - процесс может умереть;
   - индекс остаётся частично записанным;
   - следующий запуск показывает `Transport closed`.

Главный root cause из этого кейса:

- в `config.yaml` список `exclude_patterns` был вложен в `documents:`
- версия `knowledge-rag` `3.6.x` читает `exclude_patterns` только с top-level YAML
- из-за этого исключения фактически были пустыми
- сервер начинал индексировать `.venv/lib*/site-packages`, `.git`, большие скрытые каталоги и ломал себе индекс

Побочный эффект:

- на диске могли остаться чанки ChromaDB
- но `index_metadata.json` отсутствовал или не соответствовал Chroma
- из-за этого сервер мог видеть "chunks exist, docs metadata missing"

## Симптомы, которые подтверждают именно этот сценарий

Проверь по возможности:

1. В логах клиента есть `Transport closed`.
2. `knowledge-rag` стартует, но reindex/search иногда падают без нормального JSON-ответа.
3. В stderr/logs встречаются пути из `.venv/lib.../site-packages/...` во время индексации.
4. В каталоге индекса есть `chroma.sqlite3`, но:
   - нет `index_metadata.json`, или
   - metadata явно не соответствует ожидаемому числу документов.

## Что должно быть в исправленном состоянии

После фикса:

1. `exclude_patterns` реально загружаются.
2. Парсер не видит `.venv` и `.git`.
3. Индекс пересобран с нуля.
4. В data dir есть:
   - `chroma_db/chroma.sqlite3`
   - `index_metadata.json`
5. Новый чат или перезапущенный MCP-сервер отвечает на:
   - `get_index_stats`
   - `search_knowledge`
   - `reindex_documents(force=True)`

## Пошаговое восстановление

### Шаг 1. Найди текущий MCP config Codex

На этой машине у Codex MCP был в:

- `~/.codex/config.toml`

Проверь, что для `knowledge-rag` прописан server entry:

```toml
[mcp_servers.knowledge-rag]
command = "/home/USER/knowledge-rag/venv/bin/knowledge-rag"
cwd = "/home/USER/knowledge-rag"
startup_timeout_sec = 120

[mcp_servers.knowledge-rag.env]
KNOWLEDGE_RAG_DIR = "/home/USER/knowledge-rag"
```

Критично:

- `KNOWLEDGE_RAG_DIR` должен быть задан
- без него `knowledge-rag` может не прочитать нужный `config.yaml`
- тогда он уйдёт в дефолтный config и попытается скачать/поднять другую модель

Если путь к `knowledge-rag` другой, подставь реальный.

### Шаг 2. Проверь, какой config реально загружается

Запусти:

```bash
cd /home/USER/knowledge-rag
./venv/bin/python -c 'from mcp_server.config import config; print(config.documents_dir); print(len(config.exclude_patterns)); print(config.exclude_patterns[:5])'
```

Ожидаемо:

- `documents_dir` указывает на корень SoSimple
- `exclude_patterns` не пустой

Если `exclude_patterns == 0`, значит config структурно неверный или не тот файл читается.

### Шаг 3. Исправь `config.yaml`

Открой:

- `/home/USER/knowledge-rag/config.yaml`

Критическая проверка:

- `exclude_patterns:` должен быть на верхнем уровне YAML
- не внутри блока `documents:`

Правильно:

```yaml
paths:
  documents_dir: "/home/USER/git/SoSimple"
  data_dir: "/home/USER/git/SoSimple/.knowledge-rag-data"

documents:
  supported_formats:
    - .md
    - .txt
    - .py
    - .mqh
    - .mq4
    - .ipynb
  chunking:
    chunk_size: 1000
    chunk_overlap: 200

exclude_patterns:
  - "**/.git/**"
  - "**/.venv/**"
  - "**/.**"
  - "**/node_modules/**"
  - "**/archive/**"
  - "**/DATA/**"
  - "**/*.csv"
  - "**/*.pt"
  - "**/*.json"
```

Неправильно:

```yaml
documents:
  ...
  exclude_patterns:
    - "**/.venv/**"
```

После правки повтори проверку из шага 2.

### Шаг 4. Убедись, что парсер больше не лезет в `.venv`

Запусти:

```bash
cd /home/USER/knowledge-rag
./venv/bin/python -c 'from mcp_server.ingestion import DocumentParser; p=DocumentParser(); docs=p.parse_directory(); print("docs", len(docs)); print("venv_hits", sum(1 for d in docs if "/.venv/" in str(d.source))); print("git_hits", sum(1 for d in docs if "/.git/" in str(d.source)))'
```

Ожидаемо:

- `venv_hits = 0`
- `git_hits = 0`

Если не ноль, не переходи дальше. Сначала добейся корректного исключения каталогов.

### Шаг 5. Очисти битый индекс

Если `data_dir` указывает на `SoSimple/.knowledge-rag-data`, то:

```bash
cd /home/USER/git/SoSimple
rm -rf .knowledge-rag-data
mkdir -p .knowledge-rag-data
```

Это допустимо только после подтверждения, что конфиг уже исправлен.

### Шаг 6. Пересобери индекс clean rebuild

Используй локальный Python из `knowledge-rag`, запуская из каталога `knowledge-rag`:

```bash
cd /home/USER/knowledge-rag
./venv/bin/python - <<'PY'
from mcp_server.server import KnowledgeOrchestrator
orch = KnowledgeOrchestrator()
stats = orch.nuclear_rebuild()
print(stats)
PY
```

Ожидаемо:

- rebuild завершается без exceptions
- есть ненулевые `indexed` и `chunks_added`
- `errors = 0`

### Шаг 7. Проверь артефакты на диске

```bash
cd /home/USER/git/SoSimple
find .knowledge-rag-data -maxdepth 2 -type f | sort
python3 - <<'PY'
import json
from pathlib import Path
p = Path('.knowledge-rag-data/index_metadata.json')
data = json.loads(p.read_text())
print('documents', len(data))
PY
```

Ожидаемо:

- есть `chroma_db/chroma.sqlite3`
- есть `index_metadata.json`
- число документов похоже на реальное состояние репозитория

### Шаг 8. Перезапусти MCP-процесс / чат

Важно: после ручной очистки индекса и принудительного kill старый чат может держать stale transport.

Сделай одно из двух:

1. Полностью перезапусти Codex/чат.
2. Или убей процесс `knowledge-rag` и затем открой новый чат.

Пример:

```bash
pkill -f '/home/USER/knowledge-rag/venv/bin/knowledge-rag'
```

После этого новый чат должен поднять сервер заново.

## Что проверить в новом чате

Попроси агента выполнить подряд:

1. `get_index_stats()`
2. `search_knowledge("entry_path_v1_quantile export contract", hybrid_alpha=0.3, max_results=5)`
3. `reindex_documents(force=True)`

Ожидаемо:

- нет `Transport closed`
- `get_index_stats()` возвращает ненулевые `total_documents`
- `search_knowledge()` даёт реальные результаты по проекту
- `reindex_documents(force=True)` завершается штатно

## Готовый текст для другого AI-агента

Скопируй агенту задачу примерно в таком виде:

```text
Нужно восстановить knowledge-rag в проекте SoSimple.

Прочитай docs/knowledge-rag-recovery.md и выполни runbook полностью.
Цель:
- исправить config knowledge-rag
- убедиться, что exclude_patterns реально загружаются
- исключить индексацию .venv/.git
- очистить битый индекс
- выполнить clean rebuild
- перезапустить MCP/server state
- проверить get_index_stats, search_knowledge и reindex_documents(force=True)

Ничего не рефакторь заодно. Нужен только recovery knowledge-rag.
В конце дай краткий отчёт:
- какой был root cause
- что именно исправлено
- какие команды проверки выполнены
- какие результаты получены
```

## Короткий чеклист для агента

- Проверен `~/.codex/config.toml`
- Проверен `KNOWLEDGE_RAG_DIR`
- Проверен загруженный `config.yaml`
- Подтверждено, что `exclude_patterns` не пустой
- Подтверждено, что `venv_hits = 0`
- Удалён старый `.knowledge-rag-data`
- Выполнен `nuclear_rebuild()`
- На диске есть `index_metadata.json`
- MCP/чат перезапущен
- Tool calls работают без `Transport closed`

## Известный нюанс

Если агент запускает `venv/bin/knowledge-rag` вручную без `KNOWLEDGE_RAG_DIR`, сервер может:

- не подхватить проектный config
- уйти в дефолтную модель
- попытаться скачать модель из сети
- упасть по `Temporary failure in name resolution`

Это уже другая проблема, но внешне она тоже может выглядеть как `Transport closed`.
Поэтому `KNOWLEDGE_RAG_DIR` надо проверять всегда.

## Ещё один практический нюанс: reranker и offline-среда

Если `get_index_stats()` работает, а `search_knowledge()` падает с ошибками вида:

- `Temporary failure in name resolution`
- `httpx.ConnectError`
- попытка скачать `Xenova/ms-marco-MiniLM-L-6-v2`

то проблема уже не в битом индексе, а в том, что reranker-модель ещё не закэширована на этой машине.

Есть два варианта:

1. Один раз дать `knowledge-rag` скачать reranker при наличии сети.
2. Или временно отключить reranker в `config.yaml`:

```yaml
models:
  reranker:
    enabled: false
```

Это снижает качество ранжирования, но делает `search_knowledge()` рабочим даже в среде без доступа к HuggingFace.
