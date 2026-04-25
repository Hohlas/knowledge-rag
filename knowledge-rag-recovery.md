# Knowledge-RAG Recovery Runbook

Короткий runbook для восстановления `knowledge-rag`, если в чате появляются ошибки:

- `tool call failed for knowledge-rag/reindex_documents`
- `tool call failed for knowledge-rag/search_knowledge`
- `Caused by: Transport closed`

Актуально для `knowledge-rag 3.6.2`.

## Что обычно ломается

Есть 4 основных класса проблемы:

1. MCP-сервер стартует без `KNOWLEDGE_RAG_DIR` и читает не тот config.
2. `exclude_patterns` не применяются, и парсер лезет в `.venv`, `.git`, `MT/tester`, архивы.
3. Индекс в `data_dir` битый или слишком большой.
4. В offline-среде `search_knowledge()` падает на загрузке reranker-модели.

## Ключевые факты для 3.6.2

- `exclude_patterns` читаются из `documents.exclude_patterns`.
- Не надо переносить их на top-level YAML.
- Эта версия фильтрует пути через `fnmatch`, поэтому шаблоны вида `**/.venv/**` могут работать не так, как ожидается.
- Надёжнее использовать паттерны вроде:
  - `.*`
  - `MT/tester`
  - `MT/tester/**`
  - `archive`
  - `__pycache__`

## Шаг 1. Проверь MCP config Codex

Проверь `~/.codex/config.toml`.

Ожидаемо:

```toml
[mcp_servers.knowledge-rag]
command = "/home/USER/knowledge-rag/venv/bin/knowledge-rag"
cwd = "/home/USER/knowledge-rag"
startup_timeout_sec = 120

[mcp_servers.knowledge-rag.env]
KNOWLEDGE_RAG_DIR = "/home/USER/knowledge-rag"
```

Критично:

- `KNOWLEDGE_RAG_DIR` должен быть задан.
- Без него сервер может уйти в дефолтный config и дефолтные модели.

## Шаг 2. Проверь, какой config реально загружается

```bash
cd /home/USER/knowledge-rag
./venv/bin/python - <<'PY'
from mcp_server.server import config
print('documents_dir =', config.documents_dir)
print('exclude_count =', len(config.exclude_patterns))
print('exclude_head =', config.exclude_patterns[:8])
print('embedding_model =', config.embedding_model)
print('reranker_enabled =', config.reranker_enabled)
PY
```

Ожидаемо:

- `documents_dir` указывает на корень SoSimple
- `exclude_patterns` не пустой
- модель не дефолтная случайная, а ожидаемая проектом

## Шаг 3. Исправь `config.yaml`

Открой:

- `/home/USER/knowledge-rag/config.yaml`

Минимально рабочий фрагмент:

```yaml
paths:
  documents_dir: "/home/USER/git/SoSimple"
  data_dir: "/home/USER/git/SoSimple/.knowledge-rag-data"
  models_cache_dir: "./models_cache"

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
    - "MT/MQL5"
    - "MT/MQL5/**"
    - "MT/tester"
    - "MT/tester/**"
    - "MT/MQL4/Files"
    - "MT/MQL4/Files/**"
    - "MT/MQL4/Indicators"
    - "MT/MQL4/Indicators/**"
    - "MT/MQL4/Libraries"
    - "MT/MQL4/Libraries/**"
    - "MT/MQL4/Logs"
    - "MT/MQL4/Logs/**"
    - "MT/MQL4/Profiles"
    - "MT/MQL4/Profiles/**"
    - "MT/MQL4/Scripts"
    - "MT/MQL4/Scripts/**"
    - "MT/MQL4/Trash"
    - "MT/MQL4/Trash/**"
    - ".*"
    - "node_modules"
    - "dist"
    - "build"
    - "plots"
    - "checkpoints"
    - "archive"
    - "DATA"
    - "__pycache__"
    - "*.log"
    - "*.csv"
    - "*.parquet"
    - "*.pkl"
    - "*.pt"
    - "*.pth"
    - "*.bin"
    - "*.index"
    - "*.png"
    - "*.jpg"
    - "*.jpeg"
    - "*.json"
    - "*.npy"

models:
  embedding:
    model: "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
    dimensions: 384
  reranker:
    enabled: false
    model: "Xenova/ms-marco-MiniLM-L-6-v2"
    top_k_multiplier: 3
```

Примечание:

- `reranker.enabled: false` нужен для offline-среды.
- Если сеть есть и модель уже кэширована, можно вернуть `true`.

## Шаг 4. Докажи, что исключения реально работают

```bash
cd /home/USER/knowledge-rag
./venv/bin/python - <<'PY'
from pathlib import Path
from mcp_server.ingestion import DocumentParser
from mcp_server.server import config
base = Path('/home/USER/git/SoSimple')
for path in [base/'.venv', base/'.git', base/'MT/tester', base/'docs/archive/old.md']:
    print(path, DocumentParser._should_exclude(path, base, config.exclude_patterns))
PY
```

Ожидаемо:

- для всех путей вывод `True`

Дополнительно:

```bash
cd /home/USER/knowledge-rag
./venv/bin/python - <<'PY'
from mcp_server.ingestion import DocumentParser
from pathlib import Path
parser = DocumentParser()
docs = parser.parse_directory(Path('/home/USER/git/SoSimple'))
print('docs', len(docs))
PY
```

Если документов внезапно тысячи и видно `.venv`/`.git`, сначала исправь фильтры.

## Шаг 5. Очисти старый индекс

Если индекс явно битый, неполный или занимает слишком много места:

```bash
cd /home/USER/git/SoSimple
mv .knowledge-rag-data .knowledge-rag-data.bak-$(date +%Y%m%d-%H%M%S)
mkdir -p .knowledge-rag-data
```

Если места мало, старый backup потом можно удалить.

## Шаг 6. Пересобери индекс

```bash
cd /home/USER/knowledge-rag
./venv/bin/python - <<'PY'
from mcp_server.server import reindex_documents
print(reindex_documents(force=True))
PY
```

Ожидаемо:

- `status = success`
- `operation = smart_reindex`
- ненулевые `indexed` и `chunks_added`
- `errors = 0`

## Шаг 7. Проверь поиск локально

```bash
cd /home/USER/knowledge-rag
./venv/bin/python - <<'PY'
from mcp_server.server import search_knowledge
print(search_knowledge('entry_path_v1_quantile export contract', max_results=5, hybrid_alpha=0.3))
PY
```

Если локальный вызов работает, а MCP tool всё ещё даёт `Transport closed`, проблема уже в stale transport, а не в backend.

## Шаг 8. Перезапусти MCP/чат

После ручной очистки индекса или kill старого процесса текущий чат может держать stale transport.

Сделай одно из двух:

1. Перезапусти Codex/чат.
2. Или убей процесс `knowledge-rag` и открой новый чат.

Пример:

```bash
pkill -f '/home/USER/knowledge-rag/venv/bin/knowledge-rag'
```

## Что проверить в новом чате

Попроси агента выполнить подряд:

1. `get_index_stats()`
2. `search_knowledge("entry_path_v1_quantile export contract", hybrid_alpha=0.3, max_results=5)`
3. `reindex_documents(force=True)`

Ожидаемо:

- нет `Transport closed`
- `get_index_stats()` возвращает ненулевые документы
- `search_knowledge()` даёт реальные результаты
- `reindex_documents(force=True)` завершается штатно

## Короткий чеклист

- Проверен `~/.codex/config.toml`
- Проверен `KNOWLEDGE_RAG_DIR`
- Проверен реально загруженный `config.yaml`
- Подтверждено, что `exclude_patterns` не пустой
- Подтверждено, что `.venv/.git/MT/tester/archive` исключаются
- При необходимости очищен старый `.knowledge-rag-data`
- Выполнен clean reindex
- Локальный `search_knowledge()` работает
- MCP/чат перезапущен
