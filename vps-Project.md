Теперь у меня полная картина. Пишу финальный документ.

---

# LEADLAG TRADING SYSTEM — ПОЛНАЯ СПЕЦИФИКАЦИЯ ПРОЕКТА

## 0. КОНТЕКСТ

### Что уже есть
Три Jupyter ноутбука с рабочим кодом: коллектор (12 бирж, WebSocket, parquet), анализ (VWAP бинирование, детекция событий, grid search, bootstrap CI), визуализатор (HTML explorer). 12 часов собранных данных, 3.2M тиков. Результат: Lighter Perp — лучший follower, Signal C, +5.94 bps, 81% hit rate.

### Сервер
Hetzner CAX11: ARM64 (aarch64), 2 vCPU, 4 GB RAM, 40 GB SSD, Ubuntu. IP: 46.62.144.105. Подключение через VS Code Remote SSH с Mac M4.

### Цель
Превратить ноутбуки в надёжную серверную систему: сбор данных 24/7, веб-интерфейс для мониторинга и анализа, удобные эксперименты в Jupyter, расширяемость до paper/live trading.

### Принципы
- **Notebook-first**: стратегия и анализ разрабатываются в Jupyter, production-код генерируется из ноутбука
- **Простота**: минимум абстракций, минимум зависимостей, один человек должен понимать всё
- **Профессиональный стек**: Tornado + Vanilla JS + Chart.js (проверено, есть опыт), не Streamlit/Gradio
- **Parquet + DuckDB**: append-only parquet для записи, DuckDB для аналитики (0 RAM overhead на БД-сервер)
- **4 отдельных процесса**: collector, dashboard, jupyter, trader — через systemd

---

## 1. СТРУКТУРА ПРОЕКТА

```
/home/leadlag/leadlag/
│
├── config/
│   ├── venues.py                 # VenueConfig для всех бирж
│   ├── settings.py               # Пути, порты, интервалы
│   └── fees.py                   # Taker fees по биржам
│
├── parsers/                      # Один файл на биржу
│   ├── __init__.py               # re-export всех парсеров
│   ├── okx.py                    # parse_okx_trade, parse_okx_bbo
│   ├── bybit.py
│   ├── binance.py
│   ├── mexc.py
│   ├── bitget.py
│   ├── gate.py
│   ├── hyperliquid.py
│   ├── lighter.py
│   ├── edgex.py
│   └── aster.py
│
├── collector/                    # Процесс 1: сбор данных 24/7
│   ├── __init__.py
│   ├── engine.py                 # ws_venue_task, keepalive, reconnect
│   ├── writer.py                 # parquet writer с ротацией 30 мин
│   ├── status.py                 # Генерация status.json каждые 5 сек
│   └── main.py                   # Entry point: asyncio.run(), SIGTERM handler
│
├── core/                         # Утилиты (стабильный код, меняется редко)
│   ├── __init__.py
│   ├── data.py                   # DuckDB: load_ticks(), load_bbo(), venue_stats()
│   └── types.py                  # Dataclasses: Tick, BBO, VenueConfig
│
├── dashboard/                    # Процесс 2: веб-интерфейс
│   ├── server.py                 # Tornado application, routes, WebSocket handlers
│   ├── handlers.py               # HTTP handlers для каждой страницы
│   ├── api.py                    # JSON API endpoints
│   ├── ws_handler.py             # WebSocket handler для real-time обновлений
│   ├── templates/                # HTML шаблоны (Tornado template engine)
│   │   ├── base.html             # Layout: sidebar + main content
│   │   ├── overview.html         # Страница 1: главная
│   │   ├── venue_detail.html     # Страница 2: детали биржи
│   │   ├── events.html           # Страница 3: детектированные события
│   │   ├── backtest.html         # Страница 4: результаты бэктеста
│   │   ├── paper.html            # Страница 5: paper trading
│   │   ├── data_files.html       # Страница 6: управление данными
│   │   └── settings.html         # Страница 7: настройки
│   └── static/
│       ├── css/
│       │   └── app.css           # Стили (dark theme, таблицы, карточки)
│       ├── js/
│       │   ├── app.js            # Общая логика: навигация, WebSocket клиент
│       │   ├── charts.js         # Chart.js обёртки для графиков
│       │   └── tables.js         # Динамические таблицы, сортировка, фильтры
│       └── lib/                  # Внешние библиотеки (CDN fallback)
│           ├── chart.js          # Chart.js 4.4 (chart.umd.min.js)
│           ├── chartjs-adapter-date-fns.js
│           └── lightweight-charts.js  # TradingView Lightweight Charts (для price charts)
│
├── strategy/
│   └── current_strategy.py       # Генерируется из Jupyter через %%writefile
│
├── trader/                       # Процесс 4: paper/live trading
│   └── paper_main.py             # Paper trader с hot-reload стратегии
│
├── notebooks/                    # Jupyter — рабочее пространство
│   ├── research.ipynb            # Главный рабочий ноутбук
│   ├── data_check.ipynb          # Проверка качества данных
│   └── explorer.ipynb            # Генерация explorer.html
│
├── systemd/                      # Unit-файлы для systemd
│   ├── leadlag-collector.service
│   ├── leadlag-dashboard.service
│   ├── leadlag-jupyter.service
│   └── leadlag-trader.service
│
├── scripts/
│   ├── install.sh                # Полная установка с нуля
│   └── backup.sh                 # Бэкап данных
│
├── data/                         # Данные (в .gitignore)
│   ├── ticks_*.parquet           # Тики, ротация 30 мин
│   ├── bbo_*.parquet             # BBO, ротация 30 мин
│   ├── status.json               # Текущий статус коллектора
│   ├── backtest_results.json     # Последние результаты бэктеста
│   ├── paper_trades/             # Логи paper trading (JSONL)
│   └── explorer.html             # Сгенерированный визуализатор
│
├── requirements.txt
└── README.md
```

---

## 2. КОНФИГУРАЦИЯ

### 2.1 config/settings.py

```python
"""Центральная конфигурация проекта. Все пути, порты, интервалы."""
from pathlib import Path

# ─── Пути ───
PROJECT_DIR = Path("/home/leadlag/leadlag")
DATA_DIR = PROJECT_DIR / "data"
PAPER_TRADES_DIR = DATA_DIR / "paper_trades"

# ─── Коллектор ───
PARQUET_ROTATION_SECONDS = 1800          # 30 минут
STATUS_UPDATE_SECONDS = 5                # обновление status.json
PARQUET_COMPRESSION = "zstd"
QUEUE_MAXSIZE = 500_000

# ─── Dashboard ───
DASHBOARD_PORT = 8080
DASHBOARD_HOST = "0.0.0.0"
WS_UPDATE_INTERVAL_MS = 2000             # real-time обновление через WebSocket

# ─── Jupyter ───
JUPYTER_PORT = 8888

# ─── Бинирование ───
BIN_SIZE_MS = 50                         # 50ms bins для VWAP

# ─── DuckDB ───
DUCKDB_MEMORY_LIMIT = "1GB"              # из 4GB RAM оставляем запас
DUCKDB_THREADS = 2                       # из 2 vCPU
```

### 2.2 config/venues.py

```python
"""Конфигурация всех бирж. Добавление биржи = добавить блок + парсер."""
from core.types import VenueConfig
from parsers import (
    okx, bybit, binance, mexc, bitget,
    gate, hyperliquid, lighter, edgex, aster
)

VENUES = {
    # ═══ ЛИДЕРЫ ═══
    'OKX Perp': VenueConfig(
        name='OKX Perp', role='leader',
        ws_url='wss://ws.okx.com:8443/ws/v5/public',
        subscribe_msg={"op": "subscribe", "args": [{"channel": "trades", "instId": "BTC-USDT-SWAP"}]},
        parser=okx.parse_trade,
        bbo_subscribe_msg={"op": "subscribe", "args": [{"channel": "bbo-tbt", "instId": "BTC-USDT-SWAP"}]},
        bbo_parser=okx.parse_bbo,
        keepalive_type='text_ping', keepalive_interval=25,
        taker_fee_bps=5.0,
    ),
    'Bybit Perp': VenueConfig(
        name='Bybit Perp', role='leader',
        ws_url='wss://stream.bybit.com/v5/public/linear',
        subscribe_msg={"op": "subscribe", "args": ["publicTrade.BTCUSDT"]},
        parser=bybit.parse_trade,
        bbo_subscribe_msg={"op": "subscribe", "args": ["orderbook.1.BTCUSDT"]},
        bbo_parser=bybit.parse_bbo,
        keepalive_type='ws_ping', keepalive_interval=20,
        taker_fee_bps=5.5,
    ),

    # ═══ FOLLOWERS: CEX ═══
    'Binance Perp': VenueConfig(
        name='Binance Perp', role='follower',
        ws_url='wss://fstream.binance.com/stream?streams=btcusdt@trade/btcusdt@bookTicker',
        subscribe_msg=None,
        parser=binance.parse_trade,
        bbo_parser=binance.parse_bbo,
        taker_fee_bps=4.5,
    ),
    'Binance Spot': VenueConfig(
        name='Binance Spot', role='follower',
        ws_url='wss://stream.binance.com:9443/stream?streams=btcusdt@trade/btcusdt@bookTicker',
        subscribe_msg=None,
        parser=binance.parse_trade,
        bbo_parser=binance.parse_bbo,
        taker_fee_bps=10.0,
    ),
    'Bybit Spot': VenueConfig(
        name='Bybit Spot', role='follower',
        ws_url='wss://stream.bybit.com/v5/public/spot',
        subscribe_msg={"op": "subscribe", "args": ["publicTrade.BTCUSDT"]},
        parser=bybit.parse_trade,
        bbo_subscribe_msg={"op": "subscribe", "args": ["orderbook.1.BTCUSDT"]},
        bbo_parser=bybit.parse_bbo,
        keepalive_type='ws_ping', keepalive_interval=20,
        taker_fee_bps=10.0,
    ),
    'MEXC Perp': VenueConfig(
        name='MEXC Perp', role='follower',
        ws_url='wss://contract.mexc.com/edge',
        subscribe_msg={"method": "sub.deal", "param": {"symbol": "BTC_USDT"}},
        parser=mexc.parse_trade,
        keepalive_type='json_ping',
        keepalive_msg={"method": "ping"},
        keepalive_interval=15,
        taker_fee_bps=2.0,
        contract_size=0.0001,
    ),
    'Bitget Perp': VenueConfig(
        name='Bitget Perp', role='follower',
        ws_url='wss://ws.bitget.com/v2/ws/public',
        subscribe_msg={"op": "subscribe", "args": [{"instType": "USDT-FUTURES", "channel": "trade", "instId": "BTCUSDT"}]},
        parser=bitget.parse_trade,
        bbo_subscribe_msg={"op": "subscribe", "args": [{"instType": "USDT-FUTURES", "channel": "books1", "instId": "BTCUSDT"}]},
        bbo_parser=bitget.parse_bbo,
        keepalive_type='text_ping', keepalive_interval=30,
        taker_fee_bps=6.0,
    ),
    'Gate Perp': VenueConfig(
        name='Gate Perp', role='follower',
        ws_url='wss://fx-ws.gateio.ws/v4/ws/usdt',
        subscribe_msg='DYNAMIC',
        subscribe_factory=gate.make_subscribe,
        parser=gate.parse_trade,
        keepalive_type='json_ping',
        keepalive_msg={"channel": "futures.ping"},
        keepalive_interval=15,
        taker_fee_bps=5.0,
        contract_size=0.0001,
    ),

    # ═══ FOLLOWERS: DEX ═══
    'Hyperliquid Perp': VenueConfig(
        name='Hyperliquid Perp', role='follower',
        ws_url='wss://api.hyperliquid.xyz/ws',
        subscribe_msg={"method": "subscribe", "subscription": {"type": "trades", "coin": "BTC"}},
        parser=hyperliquid.parse_trade,
        taker_fee_bps=3.5,
    ),
    'Lighter Perp': VenueConfig(
        name='Lighter Perp', role='follower',
        ws_url='wss://mainnet.zklighter.elliot.ai/stream',
        subscribe_msg='DYNAMIC',
        subscribe_factory=lighter.make_subscribe,
        parser=lighter.parse_trade,
        bbo_parser=lighter.parse_bbo,
        keepalive_type='ws_ping', keepalive_interval=60,
        taker_fee_bps=0.0,
    ),
    'edgeX Perp': VenueConfig(
        name='edgeX Perp', role='follower',
        ws_url='wss://quote.edgex.exchange/api/v1/public/ws',
        subscribe_msg={"type": "subscribe", "channel": "trades.10000001"},
        parser=edgex.parse_trade,
        bbo_subscribe_msg={"type": "subscribe", "channel": "bookTicker.all.1s"},
        bbo_parser=edgex.parse_bbo,
        keepalive_type='edgex_pong',
        taker_fee_bps=2.6,
    ),
    'Aster Perp': VenueConfig(
        name='Aster Perp', role='follower',
        ws_url='wss://fstream.asterdex.com/ws/btcusdt@aggTrade/btcusdt@bookTicker',
        subscribe_msg=None,
        parser=aster.parse_trade,
        bbo_parser=aster.parse_bbo,
        taker_fee_bps=2.0,
    ),
}

LEADERS = [n for n, v in VENUES.items() if v.role == 'leader']
FOLLOWERS = [n for n, v in VENUES.items() if v.role == 'follower']
FEES = {n: v.taker_fee_bps for n, v in VENUES.items()}
```

### 2.3 config/fees.py

```python
"""Taker fees. Вынесены отдельно для удобного обновления."""
FEES = {
    'OKX Perp': 5.0,
    'Bybit Perp': 5.5,
    'Binance Perp': 4.5,
    'Binance Spot': 10.0,
    'Bybit Spot': 10.0,
    'MEXC Perp': 2.0,
    'Bitget Perp': 6.0,
    'Gate Perp': 5.0,
    'Hyperliquid Perp': 3.5,
    'Lighter Perp': 0.0,
    'edgeX Perp': 2.6,
    'Aster Perp': 2.0,
}
```

---

## 3. CORE — ТИПЫ И ДОСТУП К ДАННЫМ

### 3.1 core/types.py

```python
"""Базовые типы данных для всего проекта."""
from dataclasses import dataclass, field
from typing import Any, Callable, Optional

@dataclass
class VenueConfig:
    name: str
    role: str                          # 'leader' | 'follower'
    ws_url: str
    subscribe_msg: Any = None          # dict | None | 'DYNAMIC'
    subscribe_factory: Callable = None
    parser: Callable = None
    bbo_subscribe_msg: Any = None
    bbo_parser: Callable = None
    keepalive_type: str = None         # 'text_ping' | 'json_ping' | 'ws_ping' | 'edgex_pong' | None
    keepalive_msg: Any = None
    keepalive_interval: int = 0
    taker_fee_bps: float = 0.0
    contract_size: float = 1.0
    enabled: bool = True

@dataclass
class Tick:
    ts_ms: int
    ts_exchange_ms: int
    price: float
    qty: float
    side: str
    venue: str

@dataclass
class BBO:
    ts_ms: int
    bid_price: float
    bid_qty: float
    ask_price: float
    ask_qty: float
    venue: str
```

### 3.2 core/data.py

```python
"""DuckDB-powered доступ к parquet файлам. Никакого pandas на этапе загрузки."""
import duckdb
import pandas as pd
import pyarrow.parquet as pq
from pathlib import Path
from config.settings import DATA_DIR, DUCKDB_MEMORY_LIMIT, DUCKDB_THREADS

def _con():
    """Создать DuckDB connection с ограничениями по RAM."""
    con = duckdb.connect()
    con.execute(f"SET memory_limit='{DUCKDB_MEMORY_LIMIT}'")
    con.execute(f"SET threads={DUCKDB_THREADS}")
    return con

def load_ticks(date_from: str = None, date_to: str = None,
               venues: list[str] = None, limit: int = None) -> pd.DataFrame:
    """
    Загрузка тиков. DuckDB делает predicate pushdown в parquet —
    загружает только нужные row groups.
    
    Примеры:
        load_ticks()                                    # всё
        load_ticks("2026-04-11")                        # с даты
        load_ticks("2026-04-11", "2026-04-12")         # диапазон
        load_ticks(venues=["OKX Perp", "Lighter Perp"]) # фильтр бирж
        load_ticks(limit=10000)                         # первые N
    """
    glob = str(DATA_DIR / "ticks_*.parquet")
    filters = []
    if date_from:
        ts = int(pd.Timestamp(date_from, tz='UTC').timestamp() * 1000)
        filters.append(f"ts_ms >= {ts}")
    if date_to:
        ts = int(pd.Timestamp(date_to, tz='UTC').timestamp() * 1000)
        filters.append(f"ts_ms < {ts}")
    if venues:
        vl = ",".join(f"'{v}'" for v in venues)
        filters.append(f"venue IN ({vl})")
    where = " WHERE " + " AND ".join(filters) if filters else ""
    lim = f" LIMIT {limit}" if limit else ""
    return _con().execute(
        f"SELECT * FROM read_parquet('{glob}'){where} ORDER BY ts_ms{lim}"
    ).fetchdf()

def load_bbo(date_from: str = None, date_to: str = None,
             venues: list[str] = None) -> pd.DataFrame:
    """Аналогично для BBO."""
    glob = str(DATA_DIR / "bbo_*.parquet")
    filters = []
    if date_from:
        ts = int(pd.Timestamp(date_from, tz='UTC').timestamp() * 1000)
        filters.append(f"ts_ms >= {ts}")
    if date_to:
        ts = int(pd.Timestamp(date_to, tz='UTC').timestamp() * 1000)
        filters.append(f"ts_ms < {ts}")
    if venues:
        vl = ",".join(f"'{v}'" for v in venues)
        filters.append(f"venue IN ({vl})")
    where = " WHERE " + " AND ".join(filters) if filters else ""
    return _con().execute(
        f"SELECT * FROM read_parquet('{glob}'){where} ORDER BY ts_ms"
    ).fetchdf()

def venue_stats() -> pd.DataFrame:
    """Быстрая сводка по биржам без загрузки всех данных."""
    glob = str(DATA_DIR / "ticks_*.parquet")
    return _con().execute(f"""
        SELECT venue,
               COUNT(*) as ticks,
               MIN(ts_ms) as first_ts,
               MAX(ts_ms) as last_ts,
               APPROX_QUANTILE(price, 0.5) as median_price,
               COUNT(*) * 1000.0 / NULLIF(MAX(ts_ms) - MIN(ts_ms), 0) as avg_rate_s
        FROM read_parquet('{glob}')
        GROUP BY venue ORDER BY ticks DESC
    """).fetchdf()

def file_inventory() -> list[dict]:
    """Список parquet файлов с метаданными."""
    files = []
    for f in sorted(DATA_DIR.glob("*.parquet")):
        try:
            meta = pq.read_metadata(str(f))
            files.append({
                "name": f.name,
                "type": "ticks" if "ticks" in f.name else "bbo",
                "size_mb": round(f.stat().st_size / 1024 / 1024, 1),
                "rows": meta.num_rows,
                "modified": f.stat().st_mtime,
            })
        except Exception:
            files.append({"name": f.name, "type": "unknown", "size_mb": 0, "rows": 0, "modified": 0})
    return files

def recent_ticks(venue: str, n: int = 50) -> list[dict]:
    """Последние N тиков для venue (для dashboard)."""
    glob = str(DATA_DIR / "ticks_*.parquet")
    return _con().execute(f"""
        SELECT ts_ms, price, qty, side FROM read_parquet('{glob}')
        WHERE venue = '{venue}'
        ORDER BY ts_ms DESC LIMIT {n}
    """).fetchdf().to_dict('records')

def tick_rate_history(venue: str, minutes: int = 30) -> list[dict]:
    """Tick rate по минутам для графика."""
    glob = str(DATA_DIR / "ticks_*.parquet")
    import time
    cutoff = int((time.time() - minutes * 60) * 1000)
    return _con().execute(f"""
        SELECT (ts_ms // 60000) * 60000 as minute_ts,
               COUNT(*) / 60.0 as rate
        FROM read_parquet('{glob}')
        WHERE venue = '{venue}' AND ts_ms > {cutoff}
        GROUP BY 1 ORDER BY 1
    """).fetchdf().to_dict('records')
```

---

## 4. ПАРСЕРЫ

Каждый файл в `parsers/` содержит `parse_trade(msg, ts) → list[dict]` и опционально `parse_bbo(msg, ts) → dict|None`. Код идентичен тому что в ваших ноутбуках — только разнесён по файлам.

### 4.1 parsers/okx.py (пример)

```python
"""OKX Perp: trades + bbo-tbt."""

def parse_trade(msg, ts):
    if msg.get('arg', {}).get('channel') != 'trades' or 'data' not in msg:
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d['ts']),
             'price': float(d['px']), 'qty': float(d['sz']),
             'side': d['side']} for d in msg['data']]

def parse_bbo(msg, ts):
    if msg.get('arg', {}).get('channel') != 'bbo-tbt' or 'data' not in msg:
        return None
    d = msg['data'][0]
    return {'ts_ms': ts,
            'bid_price': float(d['bids'][0][0]), 'bid_qty': float(d['bids'][0][1]),
            'ask_price': float(d['asks'][0][0]), 'ask_qty': float(d['asks'][0][1])}
```

### 4.2 parsers/__init__.py

```python
"""Re-export для удобного импорта: from parsers import okx."""
from parsers import (okx, bybit, binance, mexc, bitget,
                     gate, hyperliquid, lighter, edgex, aster)
```

Остальные парсеры (bybit, binance, mexc, bitget, gate, hyperliquid, lighter, edgex, aster) — прямой перенос из ячейки 2 Notebook 1 текущего кода. Каждый в отдельном файле. Включая фабрики `make_subscribe()` для Gate и Lighter.

---

## 5. КОЛЛЕКТОР (collector/)

### 5.1 collector/engine.py

Перенос из ячейки 4 Notebook 1. Изменения:
- `print()` → `logging.info()/warning()/error()`
- Убрать `tqdm`, `ipywidgets`, `nest_asyncio`
- Добавить параметр `stop_event` вместо `COLLECT_SECONDS`
- Добавить вызов `status.write_status()` в heartbeat

```python
"""Async engine: WebSocket tasks, writers, heartbeat."""
import asyncio
import json
import time
import random
import logging
import websockets
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path
from datetime import datetime, timezone

from config.settings import (DATA_DIR, PARQUET_ROTATION_SECONDS,
                              PARQUET_COMPRESSION, QUEUE_MAXSIZE, STATUS_UPDATE_SECONDS)
from collector.status import write_status

logger = logging.getLogger('collector')

# ─── Parquet schemas ───
TICK_SCHEMA = pa.schema([
    ('ts_ms', pa.int64()), ('ts_exchange_ms', pa.int64()),
    ('price', pa.float64()), ('qty', pa.float64()),
    ('side', pa.string()), ('venue', pa.string()),
])
BBO_SCHEMA = pa.schema([
    ('ts_ms', pa.int64()),
    ('bid_price', pa.float64()), ('bid_qty', pa.float64()),
    ('ask_price', pa.float64()), ('ask_qty', pa.float64()),
    ('venue', pa.string()),
])

# ─── Глобальные счётчики (читаются из status.py и dashboard) ───
tick_counts = {}
bbo_counts = {}
venue_status = {}
venue_errors = {}
venue_reconnects = {}
errors_log = []  # последние 100 ошибок


async def keepalive_task(ws, cfg):
    """Отправляет ping по расписанию."""
    try:
        while True:
            await asyncio.sleep(cfg.keepalive_interval)
            if cfg.keepalive_type == 'text_ping':
                await ws.send("ping")
            elif cfg.keepalive_type == 'json_ping':
                await ws.send(json.dumps(cfg.keepalive_msg))
    except (asyncio.CancelledError, Exception):
        pass


async def ws_venue_task(cfg, trades_queue, bbo_queue, stop_event):
    """Один WebSocket connection для одной биржи. Auto-reconnect."""
    venue_status[cfg.name] = 'connecting'
    tick_counts.setdefault(cfg.name, 0)
    bbo_counts.setdefault(cfg.name, 0)
    venue_reconnects.setdefault(cfg.name, 0)
    attempt = 0

    while not stop_event.is_set():
        try:
            ws_kwargs = dict(close_timeout=5, max_size=10_000_000, open_timeout=15)
            if cfg.keepalive_type == 'ws_ping':
                ws_kwargs['ping_interval'] = cfg.keepalive_interval
                ws_kwargs['ping_timeout'] = 10
            else:
                ws_kwargs['ping_interval'] = None
                ws_kwargs['ping_timeout'] = None

            venue_status[cfg.name] = 'connecting'
            async with websockets.connect(cfg.ws_url, **ws_kwargs) as ws:
                venue_status[cfg.name] = 'ok'
                if attempt > 0:
                    venue_reconnects[cfg.name] += 1
                    logger.info(f"{cfg.name}: reconnected (attempt {attempt})")
                else:
                    logger.info(f"{cfg.name}: connected")

                # Subscribe
                if cfg.subscribe_msg == 'DYNAMIC' and cfg.subscribe_factory:
                    msgs = cfg.subscribe_factory()
                    for m in (msgs if isinstance(msgs, list) else [msgs]):
                        await ws.send(json.dumps(m))
                elif cfg.subscribe_msg is not None:
                    await ws.send(json.dumps(cfg.subscribe_msg))

                if cfg.bbo_subscribe_msg:
                    await ws.send(json.dumps(cfg.bbo_subscribe_msg))

                # Keepalive task
                ka = None
                if cfg.keepalive_type in ('text_ping', 'json_ping'):
                    ka = asyncio.create_task(keepalive_task(ws, cfg))

                try:
                    connect_time = time.time()
                    async for raw in ws:
                        if stop_event.is_set():
                            break
                        ts = int(time.time_ns() // 1_000_000)

                        # Фильтрация служебных
                        if isinstance(raw, bytes):
                            continue
                        if raw in ('pong', '"pong"', 'ping'):
                            continue
                        try:
                            msg = json.loads(raw)
                        except (json.JSONDecodeError, TypeError):
                            continue
                        if not isinstance(msg, dict):
                            continue

                        # Skip confirmations
                        if msg.get('event') in ('subscribe', 'subscribed', 'info', 'pong'):
                            continue
                        if msg.get('op') == 'pong' or msg.get('ret_msg') == 'pong':
                            continue
                        if msg.get('type') in ('connected', 'pong', 'subscribed'):
                            continue
                        if msg.get('channel', '').endswith(('.ping', '.pong')):
                            continue
                        if msg.get('data') == 'pong' or msg.get('method') == 'pong':
                            continue

                        # edgeX ping → pong
                        if msg.get('type') == 'ping':
                            await ws.send(json.dumps({"type": "pong", "time": msg.get("time", "")}))
                            continue

                        # Parse trades
                        try:
                            ticks = cfg.parser(msg, ts)
                            for t in ticks:
                                t['venue'] = cfg.name
                                await trades_queue.put(t)
                            tick_counts[cfg.name] += len(ticks)
                        except Exception as e:
                            _log_error(cfg.name, f"parse_trade: {e}")

                        # Parse BBO
                        if cfg.bbo_parser:
                            try:
                                bbo = cfg.bbo_parser(msg, ts)
                                if bbo:
                                    bbo['venue'] = cfg.name
                                    await bbo_queue.put(bbo)
                                    bbo_counts[cfg.name] += 1
                            except Exception as e:
                                _log_error(cfg.name, f"parse_bbo: {e}")

                    # Если WS закрылся нормально — сбросить attempt если работал долго
                    if time.time() - connect_time > 60:
                        attempt = 0

                finally:
                    if ka:
                        ka.cancel()

        except asyncio.CancelledError:
            return
        except Exception as e:
            venue_status[cfg.name] = 'reconnecting'
            _log_error(cfg.name, str(e))
            if stop_event.is_set():
                return
            delay = min(1.0 * (2 ** attempt), 60.0)
            delay += delay * 0.25 * (2 * random.random() - 1)
            try:
                await asyncio.wait_for(stop_event.wait(), timeout=delay)
                return
            except asyncio.TimeoutError:
                pass
            attempt += 1


def _log_error(venue, msg):
    """Записать ошибку в лог и в буфер для dashboard."""
    logger.warning(f"{venue}: {msg}")
    venue_errors[venue] = msg
    errors_log.append({
        'ts': int(time.time() * 1000),
        'venue': venue,
        'error': msg
    })
    if len(errors_log) > 100:
        errors_log.pop(0)


async def writer_task(queue, stop_event, prefix, schema):
    """Пишет данные из очереди в parquet с ротацией."""
    buffer = []
    last_rotate = time.time()

    def flush():
        nonlocal buffer, last_rotate
        if not buffer:
            return
        ts_str = datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")
        path = DATA_DIR / f"{prefix}_{ts_str}.parquet"
        cols = {name: [r.get(name, 0) for r in buffer] for name in schema.names}
        pq.write_table(
            pa.table(cols, schema=schema), str(path),
            compression=PARQUET_COMPRESSION
        )
        logger.info(f"[WRITER] {prefix}: {len(buffer):,} rows → {path.name}")
        buffer = []
        last_rotate = time.time()

    try:
        while not (stop_event.is_set() and queue.empty()):
            try:
                item = await asyncio.wait_for(queue.get(), timeout=1.0)
                buffer.append(item)
            except asyncio.TimeoutError:
                pass
            if time.time() - last_rotate >= PARQUET_ROTATION_SECONDS:
                flush()
    finally:
        flush()


async def heartbeat_task(stop_event, start_time, venues_dict):
    """Логирование + обновление status.json."""
    while not stop_event.is_set():
        await asyncio.sleep(STATUS_UPDATE_SECONDS)
        elapsed = time.time() - start_time

        # Статус в лог
        active = sum(1 for s in venue_status.values() if s == 'ok')
        total_ticks = sum(tick_counts.values())
        logger.info(
            f"Uptime: {elapsed/60:.0f}m | "
            f"Venues: {active}/{len(venues_dict)} | "
            f"Ticks: {total_ticks:,} | "
            f"Rate: {total_ticks/elapsed:.0f}/s"
        )

        # status.json для dashboard
        write_status(
            tick_counts=tick_counts,
            bbo_counts=bbo_counts,
            venue_status=venue_status,
            venue_errors=venue_errors,
            venue_reconnects=venue_reconnects,
            errors_log=errors_log,
            start_time=start_time,
        )


async def run_collector(stop_event, venues):
    """Main entry: запускает все tasks."""
    DATA_DIR.mkdir(parents=True, exist_ok=True)

    trades_queue = asyncio.Queue(maxsize=QUEUE_MAXSIZE)
    bbo_queue = asyncio.Queue(maxsize=QUEUE_MAXSIZE)
    start_time = time.time()

    active = {n: c for n, c in venues.items() if c.enabled}
    logger.info(f"Starting {len(active)} venues: {list(active.keys())}")

    ws_tasks = [
        asyncio.create_task(ws_venue_task(cfg, trades_queue, bbo_queue, stop_event))
        for cfg in active.values()
    ]
    t_writer = asyncio.create_task(writer_task(trades_queue, stop_event, "ticks", TICK_SCHEMA))
    b_writer = asyncio.create_task(writer_task(bbo_queue, stop_event, "bbo", BBO_SCHEMA))
    hb = asyncio.create_task(heartbeat_task(stop_event, start_time, active))

    # Ждём stop_event (от SIGTERM)
    await stop_event.wait()
    logger.info("Stop signal received, shutting down...")

    # Graceful shutdown
    for t in ws_tasks:
        t.cancel()
    await asyncio.gather(t_writer, b_writer, return_exceptions=True)
    hb.cancel()
    logger.info("Collector stopped")
```

### 5.2 collector/status.py

```python
"""Генерация status.json для dashboard."""
import json
import os
import time
import psutil
from pathlib import Path
from config.settings import DATA_DIR

STATUS_PATH = DATA_DIR / "status.json"

def write_status(tick_counts, bbo_counts, venue_status, venue_errors,
                 venue_reconnects, errors_log, start_time):
    """Atomic write status.json."""
    now = time.time()
    elapsed = now - start_time

    venues = {}
    for name in set(list(tick_counts.keys()) + list(venue_status.keys())):
        tc = tick_counts.get(name, 0)
        venues[name] = {
            "ticks": tc,
            "rate": round(tc / elapsed, 1) if elapsed > 0 else 0,
            "bbo": bbo_counts.get(name, 0),
            "status": venue_status.get(name, "unknown"),
            "reconnects": venue_reconnects.get(name, 0),
            "last_error": venue_errors.get(name, ""),
        }

    tick_files = list(DATA_DIR.glob("ticks_*.parquet"))
    bbo_files = list(DATA_DIR.glob("bbo_*.parquet"))
    total_bytes = sum(f.stat().st_size for f in tick_files + bbo_files)

    status = {
        "ts": int(now * 1000),
        "uptime_s": int(elapsed),
        "venues": venues,
        "files": {
            "ticks_count": len(tick_files),
            "bbo_count": len(bbo_files),
            "total_mb": round(total_bytes / 1024 / 1024, 1),
        },
        "system": {
            "cpu_percent": psutil.cpu_percent(interval=None),
            "ram_used_mb": round(psutil.virtual_memory().used / 1024 / 1024),
            "ram_total_mb": round(psutil.virtual_memory().total / 1024 / 1024),
            "disk_free_gb": round(psutil.disk_usage('/').free / 1024 / 1024 / 1024, 1),
        },
        "errors_log": errors_log[-50:],  # последние 50
    }

    tmp = str(STATUS_PATH) + ".tmp"
    with open(tmp, 'w') as f:
        json.dump(status, f)
    os.replace(tmp, str(STATUS_PATH))
```

### 5.3 collector/main.py

```python
#!/usr/bin/env python3
"""LeadLag Collector — systemd entry point."""
import asyncio
import signal
import logging
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from collector.engine import run_collector
from config.venues import VENUES

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
    handlers=[logging.StreamHandler()]
)

async def main():
    stop_event = asyncio.Event()
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, stop_event.set)

    await run_collector(stop_event=stop_event, venues=VENUES)

if __name__ == '__main__':
    asyncio.run(main())
```

---

## 6. DASHBOARD (Tornado + Vanilla JS + Chart.js)

### 6.1 Архитектура dashboard

```
Браузер                         Tornado Server (:8080)
  │                                     │
  │  GET /                              │ → overview.html
  │  GET /venues/Lighter%20Perp         │ → venue_detail.html
  │  GET /events                        │ → events.html
  │  GET /backtest                      │ → backtest.html
  │  GET /paper                         │ → paper.html
  │  GET /files                         │ → data_files.html
  │  GET /settings                      │ → settings.html
  │                                     │
  │  WS /ws/live ◄─────────────────────►│ WebSocket: real-time updates
  │    (каждые 2s сервер пушит:         │   читает status.json
  │     venue_table, summary_cards,     │   + DuckDB запросы для графиков
  │     errors, sidebar_status)         │
  │                                     │
  │  GET /api/venue/{name}/ticks?n=50   │ → JSON: last N ticks
  │  GET /api/venue/{name}/rate?min=30  │ → JSON: tick rate history
  │  GET /api/backtest/results          │ → JSON: ci_df
  │  GET /api/backtest/grid?f=X&s=Y     │ → JSON: grid heatmap data
  │  GET /api/paper/status              │ → JSON: paper trading state
  │  GET /api/paper/trades              │ → JSON: trade log
  │  GET /api/files                     │ → JSON: file inventory
  │  GET /api/system                    │ → JSON: CPU, RAM, disk
  │                                     │
  │  POST /api/collector/restart        │ → systemctl restart
  │  POST /api/collector/stop           │ → systemctl stop
  │  POST /api/settings/venues          │ → save enabled/disabled
```

### 6.2 dashboard/server.py

```python
"""Tornado web server для dashboard."""
import os
import sys
import json
import logging
import tornado.ioloop
import tornado.web
import tornado.websocket
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from config.settings import DASHBOARD_PORT, DASHBOARD_HOST, DATA_DIR, WS_UPDATE_INTERVAL_MS
from dashboard.handlers import (
    OverviewHandler, VenueDetailHandler, EventsHandler,
    BacktestHandler, PaperHandler, FilesHandler, SettingsHandler,
)
from dashboard.api import (
    VenueTicksAPI, VenueRateAPI, BacktestResultsAPI, BacktestGridAPI,
    PaperStatusAPI, PaperTradesAPI, FilesAPI, SystemAPI,
    CollectorRestartAPI, CollectorStopAPI, SettingsVenuesAPI,
)
from dashboard.ws_handler import LiveWSHandler

logger = logging.getLogger('dashboard')

def make_app():
    base = Path(__file__).parent
    return tornado.web.Application(
        [
            # ─── Pages ───
            (r"/", OverviewHandler),
            (r"/venues/(.+)", VenueDetailHandler),
            (r"/events", EventsHandler),
            (r"/backtest", BacktestHandler),
            (r"/paper", PaperHandler),
            (r"/files", FilesHandler),
            (r"/settings", SettingsHandler),

            # ─── WebSocket ───
            (r"/ws/live", LiveWSHandler),

            # ─── JSON API ───
            (r"/api/venue/(.+)/ticks", VenueTicksAPI),
            (r"/api/venue/(.+)/rate", VenueRateAPI),
            (r"/api/backtest/results", BacktestResultsAPI),
            (r"/api/backtest/grid", BacktestGridAPI),
            (r"/api/paper/status", PaperStatusAPI),
            (r"/api/paper/trades", PaperTradesAPI),
            (r"/api/files", FilesAPI),
            (r"/api/system", SystemAPI),

            # ─── Actions ───
            (r"/api/collector/restart", CollectorRestartAPI),
            (r"/api/collector/stop", CollectorStopAPI),
            (r"/api/settings/venues", SettingsVenuesAPI),
        ],
        template_path=str(base / "templates"),
        static_path=str(base / "static"),
        debug=False,
    )

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO,
                        format='%(asctime)s [%(levelname)s] %(name)s: %(message)s')
    app = make_app()
    app.listen(DASHBOARD_PORT, DASHBOARD_HOST)
    logger.info(f"Dashboard running on http://{DASHBOARD_HOST}:{DASHBOARD_PORT}")
    tornado.ioloop.IOLoop.current().start()
```

### 6.3 dashboard/ws_handler.py

```python
"""WebSocket handler: пушит обновления клиентам каждые 2 секунды."""
import json
import time
import tornado.websocket
import tornado.ioloop
import logging
from pathlib import Path
from config.settings import DATA_DIR, WS_UPDATE_INTERVAL_MS

logger = logging.getLogger('dashboard.ws')

# Все подключённые клиенты
clients = set()

class LiveWSHandler(tornado.websocket.WebSocketHandler):
    def check_origin(self, origin):
        return True  # разрешить все origins (сервер за firewall)

    def open(self):
        clients.add(self)
        logger.info(f"WS client connected ({len(clients)} total)")

    def on_close(self):
        clients.discard(self)
        logger.info(f"WS client disconnected ({len(clients)} total)")

    def on_message(self, message):
        """Клиент может запросить конкретные данные."""
        try:
            msg = json.loads(message)
            if msg.get('type') == 'ping':
                self.write_message(json.dumps({'type': 'pong'}))
        except Exception:
            pass


def read_status():
    """Прочитать status.json от коллектора."""
    path = DATA_DIR / "status.json"
    try:
        with open(path) as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {"ts": 0, "uptime_s": 0, "venues": {}, "files": {}, "system": {}, "errors_log": []}


def broadcast():
    """Отправить обновление всем клиентам."""
    if not clients:
        return
    status = read_status()
    payload = json.dumps({
        'type': 'status_update',
        'data': status,
        'server_ts': int(time.time() * 1000),
    })
    dead = set()
    for client in clients:
        try:
            client.write_message(payload)
        except Exception:
            dead.add(client)
    clients.difference_update(dead)


def start_broadcast_loop():
    """Запустить периодическую рассылку."""
    cb = tornado.ioloop.PeriodicCallback(broadcast, WS_UPDATE_INTERVAL_MS)
    cb.start()
    return cb
```

### 6.4 dashboard/handlers.py

```python
"""HTTP handlers для HTML страниц."""
import tornado.web
from config.settings import DATA_DIR
from dashboard.ws_handler import read_status

class BaseHandler(tornado.web.RequestHandler):
    def get_template_namespace(self):
        ns = super().get_template_namespace()
        ns['current_path'] = self.request.path
        return ns

class OverviewHandler(BaseHandler):
    def get(self):
        status = read_status()
        self.render("overview.html", status=status)

class VenueDetailHandler(BaseHandler):
    def get(self, venue_name):
        venue_name = tornado.escape.url_unescape(venue_name)
        status = read_status()
        venue = status.get('venues', {}).get(venue_name, {})
        self.render("venue_detail.html", venue_name=venue_name, venue=venue, status=status)

class EventsHandler(BaseHandler):
    def get(self):
        self.render("events.html")

class BacktestHandler(BaseHandler):
    def get(self):
        # Читать backtest_results.json если есть
        import json
        results_path = DATA_DIR / "backtest_results.json"
        results = {}
        try:
            with open(results_path) as f:
                results = json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            pass
        self.render("backtest.html", results=results)

class PaperHandler(BaseHandler):
    def get(self):
        self.render("paper.html")

class FilesHandler(BaseHandler):
    def get(self):
        self.render("data_files.html")

class SettingsHandler(BaseHandler):
    def get(self):
        from config.venues import VENUES
        self.render("settings.html", venues=VENUES)
```

### 6.5 dashboard/api.py

```python
"""JSON API endpoints."""
import json
import subprocess
import tornado.web
from config.settings import DATA_DIR
from core.data import recent_ticks, tick_rate_history, file_inventory

class JSONHandler(tornado.web.RequestHandler):
    def set_default_headers(self):
        self.set_header("Content-Type", "application/json")
    def write_json(self, data):
        self.write(json.dumps(data, default=str))

class VenueTicksAPI(JSONHandler):
    def get(self, venue_name):
        venue_name = tornado.escape.url_unescape(venue_name)
        n = int(self.get_argument("n", 50))
        self.write_json(recent_ticks(venue_name, n))

class VenueRateAPI(JSONHandler):
    def get(self, venue_name):
        venue_name = tornado.escape.url_unescape(venue_name)
        minutes = int(self.get_argument("min", 30))
        self.write_json(tick_rate_history(venue_name, minutes))

class BacktestResultsAPI(JSONHandler):
    def get(self):
        path = DATA_DIR / "backtest_results.json"
        try:
            with open(path) as f:
                self.write_json(json.load(f))
        except FileNotFoundError:
            self.write_json({"error": "no results yet"})

class BacktestGridAPI(JSONHandler):
    def get(self):
        # grid data загружается из backtest_results.json
        path = DATA_DIR / "backtest_results.json"
        try:
            with open(path) as f:
                data = json.load(f)
            follower = self.get_argument("follower", "")
            signal = self.get_argument("signal", "")
            grid = [r for r in data.get("grid", [])
                    if r.get("follower") == follower and r.get("signal") == signal]
            self.write_json(grid)
        except FileNotFoundError:
            self.write_json([])

class PaperStatusAPI(JSONHandler):
    def get(self):
        path = DATA_DIR / "paper_status.json"
        try:
            with open(path) as f:
                self.write_json(json.load(f))
        except FileNotFoundError:
            self.write_json({"running": False})

class PaperTradesAPI(JSONHandler):
    def get(self):
        # Читаем последний JSONL файл
        trade_dir = DATA_DIR / "paper_trades"
        files = sorted(trade_dir.glob("*.jsonl")) if trade_dir.exists() else []
        if not files:
            self.write_json([])
            return
        trades = []
        for line in open(files[-1]):
            try:
                trades.append(json.loads(line))
            except json.JSONDecodeError:
                pass
        self.write_json(trades[-100:])  # последние 100

class FilesAPI(JSONHandler):
    def get(self):
        self.write_json(file_inventory())

class SystemAPI(JSONHandler):
    def get(self):
        import psutil
        self.write_json({
            "cpu_percent": psutil.cpu_percent(interval=0.1),
            "ram_used_mb": round(psutil.virtual_memory().used / 1024 / 1024),
            "ram_total_mb": round(psutil.virtual_memory().total / 1024 / 1024),
            "disk_free_gb": round(psutil.disk_usage('/').free / 1024 / 1024 / 1024, 1),
            "disk_total_gb": round(psutil.disk_usage('/').total / 1024 / 1024 / 1024, 1),
        })

class CollectorRestartAPI(JSONHandler):
    def post(self):
        result = subprocess.run(
            ["sudo", "systemctl", "restart", "leadlag-collector"],
            capture_output=True, text=True, timeout=10
        )
        self.write_json({"ok": result.returncode == 0, "output": result.stdout or result.stderr})

class CollectorStopAPI(JSONHandler):
    def post(self):
        result = subprocess.run(
            ["sudo", "systemctl", "stop", "leadlag-collector"],
            capture_output=True, text=True, timeout=10
        )
        self.write_json({"ok": result.returncode == 0, "output": result.stdout or result.stderr})

class SettingsVenuesAPI(JSONHandler):
    def post(self):
        # Сохранить enabled/disabled venues
        body = json.loads(self.request.body)
        # TODO: записать в config file, перезапустить collector
        self.write_json({"ok": True})
```

---

## 7. DASHBOARD — ЭКРАНЫ (полная спецификация для ИИ)

### 7.1 Цветовая схема и стили

```css
/* dashboard/static/css/app.css */

:root {
    --bg:          #0f1117;
    --card:        #1a1d27;
    --card-hover:  #22252f;
    --border:      #2a2d3a;
    --text:        #e0e0e0;
    --text-dim:    #888;
    --accent:      #4a6cf7;
    --positive:    #00c853;
    --negative:    #ff5252;
    --warning:     #ffa726;
}

/* Вся страница — dark theme. Шрифт: system sans-serif, размер 14px base. */
/* Sidebar: 224px fixed left, цвет var(--card), border-right 1px var(--border). */
/* Main content: margin-left 224px, padding 24px. */
/* Карточки: background var(--card), border-radius 8px, padding 16px. */
/* Таблицы: без border, строки разделены border-bottom 1px var(--border), hover var(--card-hover). */
/* Числа: font-variant-numeric tabular-nums (моноширинный для чисел). */
```

### 7.2 base.html — Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ ┌── Sidebar (224px, fixed) ──┐  ┌── Main ──────────────────┐   │
│ │                             │  │                           │   │
│ │  ▣ LeadLag                  │  │  {% block content %}      │   │
│ │  ─────────────────────      │  │  {% endblock %}           │   │
│ │  📊 Overview     ← active  │  │                           │   │
│ │  📡 Venues                  │  │                           │   │
│ │  ⚡ Events                  │  │                           │   │
│ │  📈 Backtest                │  │                           │   │
│ │  💹 Paper Trading           │  │                           │   │
│ │  📁 Data Files              │  │                           │   │
│ │  ⚙️  Settings                │  │                           │   │
│ │                             │  │                           │   │
│ │  ── Collector ──            │  │                           │   │
│ │  🟢 Running                 │  │                           │   │
│ │  ⏱ 14h 32m                 │  │                           │   │
│ │  💾 847 MB                  │  │                           │   │
│ │  📊 3.2M ticks              │  │                           │   │
│ └─────────────────────────────┘  └───────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

Sidebar внизу (секция "Collector") обновляется через WebSocket: JS-клиент получает `status_update`, обновляет элементы по id.

### 7.3 Страница Overview (`/`)

```
┌── 4 Summary Cards (grid, 4 колонки) ────────────────────────────┐
│ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ │
│ │ TOTAL TICKS    │ │ ACTIVE VENUES  │ │ EVENTS TODAY   │ │ BEST P&L       │ │
│ │ 3,244,939      │ │ 10 / 12       │ │ 47             │ │ +5.94 bps      │ │
│ │ ↗ 12.3/s avg  │ │ 🔴 2 errors   │ │ 14 signal C   │ │ Lighter/C      │ │
│ └────────────────┘ └────────────────┘ └────────────────┘ └────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘

┌── Venue Table ──────────────────────────────────────────────────┐
│  ●  Venue              Role     Status  Ticks      Rate/s  BBO │
│  ── ──────────────── ──────── ──────── ──────── ──────── ───── │
│  🟢 OKX Perp          leader   OK      721,098    16.7    ✓   │ ← кликабельная строка
│  🟢 Bybit Perp        leader   OK      585,170    13.5    ✓   │
│  🔴 Binance Perp      follower LOW      11,455     0.3    ✓   │
│  🟢 MEXC Perp         follower OK      397,718     9.2    –   │
│  🟢 Lighter Perp      follower OK      273,543     6.3    ✓   │ ← best follower подсвечен
│  ...                                                           │
│  Клик → /venues/{name}                                         │
└─────────────────────────────────────────────────────────────────┘

┌── Tick Rate (Chart.js line chart) ──────────────────────────────┐
│  X: время (последние 30 мин, 1 точка = 1 мин)                  │
│  Y: ticks/sec                                                    │
│  Линии: одна на venue (12 линий), палитра из 12 цветов          │
│  Legend: справа, кликабельная (показать/скрыть)                  │
│  Обновление: каждые 10s через WebSocket                         │
│  При первой загрузке: GET /api/venue/{name}/rate?min=30         │
│  Затем: append новую точку из ws status_update                  │
└─────────────────────────────────────────────────────────────────┘

┌── Recent Errors (последние 20) ─────────────────────────────────┐
│  14:32:05  Lighter Perp   ConnectionClosed (reconnect #3)       │
│  14:28:11  edgeX Perp     Timeout (reconnect #7)                │
│  Обновление: из ws status_update → errors_log                   │
└─────────────────────────────────────────────────────────────────┘
```

**Логика обновления:**
1. При загрузке страницы: Tornado рендерит HTML с начальными данными из status.json
2. JS устанавливает WebSocket на `/ws/live`
3. Каждые 2 секунды приходит `status_update`
4. JS обновляет: summary cards (по id), venue table (пересобрать tbody), sidebar status, errors list
5. Tick rate chart: каждые 10 секунд — добавить новую точку, сдвинуть окно

### 7.4 Страница Venue Detail (`/venues/{name}`)

```
┌── Header ───────────────────────────────────────────────────────┐
│  ← Back to Overview                                             │
│  Lighter Perp                                                    │
│  Role: follower  │  Fee: 0.0 bps  │  Status: 🟢 OK             │
└─────────────────────────────────────────────────────────────────┘

┌── Stats Cards (6 в ряд) ───────────────────────────────────────┐
│  Ticks    │ BBO     │ Rate/s │ Reconnects │ Last Tick  │ Uptime │
│  273,543  │ 218,000 │ 6.3    │ 4          │ 2s ago     │ 99.7%  │
└─────────────────────────────────────────────────────────────────┘

┌── Tick Rate Chart (Chart.js, 1 час) ────────────────────────────┐
│  1 линия, данные: GET /api/venue/{name}/rate?min=60             │
│  Обновление: append из ws                                        │
└─────────────────────────────────────────────────────────────────┘

┌── Price Chart (Lightweight Charts, 5 мин) ──────────────────────┐
│  TradingView Lightweight Charts: line series                     │
│  Данные: GET /api/venue/{name}/ticks?n=500                      │
│  Агрегировать в 1s bins client-side                              │
│  Обновление: через ws (если venue = текущий)                     │
└─────────────────────────────────────────────────────────────────┘

┌── Last 50 Ticks (таблица, auto-scroll) ─────────────────────────┐
│  Time          │ Price     │ Qty     │ Side                      │
│  14:32:05.123  │ 73,067.5  │ 0.0100  │ ■ buy (green)            │
│  14:32:05.089  │ 73,067.0  │ 0.0050  │ ■ sell (red)             │
│  Обновление: GET /api/venue/{name}/ticks?n=50 каждые 5s         │
└─────────────────────────────────────────────────────────────────┘
```

### 7.5 Страница Events (`/events`)

```
┌── Фильтры ──────────────────────────────────────────────────────┐
│  Signal: [ All ▼]  Min σ: [2.0 ____]  [Apply]                  │
│  (фильтры работают client-side — данные из backtest_results.json)│
└─────────────────────────────────────────────────────────────────┘

┌── Events Table ─────────────────────────────────────────────────┐
│  #   │ Time     │ Signal │ Dir │ Mag σ │ Leader     │ Laggers  │
│  1   │ 14:32:05 │  C     │ ↑   │ 3.89  │ confirmed  │ 7       │
│  2   │ 14:28:11 │  A     │ ↓   │ 2.51  │ OKX Perp   │ 4       │
│  Клик → раскрывающаяся панель с деталями                        │
│  Кнопка: [Open Explorer] → data/explorer.html#event=N           │
└─────────────────────────────────────────────────────────────────┘

┌── Раскрытая панель для события #1 ──────────────────────────────┐
│  Lagging followers: MEXC Perp, Lighter Perp, edgeX Perp, ...   │
│  Confirmer: OKX Perp, lag 50ms                                   │
│  Best P&L: +8.03 bps (Lighter, delay=0, hold=30s)              │
│  [Open in Explorer]                                              │
└─────────────────────────────────────────────────────────────────┘

Данные: из data/backtest_results.json (генерируется в Jupyter)
```

### 7.6 Страница Backtest (`/backtest`)

```
┌── Info Bar ─────────────────────────────────────────────────────┐
│  Last run: 2026-04-12 05:30 UTC                                 │
│  Dataset: 12h (519 events: A=135, B=155, C=229)                │
│  [▶ Run Backtest] — запускает jupyter nbconvert (async)         │
│  [📥 Download CSV]                                               │
└─────────────────────────────────────────────────────────────────┘

┌── Results Table (из ci_df) ─────────────────────────────────────┐
│  🟢🟡🔴 │ Follower     │Sig│ Delay│ Hold │ N │ Net    │ 95% CI  │
│  🟢     │ Lighter Perp │ C │   0  │ 30s  │73 │ +5.94  │[+3.7,+8]│
│  🟢     │ Lighter Perp │ B │   0  │ 30s  │54 │ +3.99  │[+1.7,+7]│
│  🟢     │ MEXC Perp    │ C │   0  │ 30s  │70 │ +3.22  │[+1.0,+6]│
│  🟡     │ edgeX Perp   │ C │   0  │ 30s  │96 │ +1.18  │[-0.6,+3]│
│  🔴     │ ...          │   │      │      │   │        │         │
│  Сортировка: по Net P&L descending                               │
└─────────────────────────────────────────────────────────────────┘

┌── Grid Heatmap ─────────────────────────────────────────────────┐
│  Dropdown: [Lighter Perp ▼]  [Signal C ▼]                      │
│  Chart.js matrix/heatmap: X=delay_ms, Y=hold_ms, color=net_pnl │
│  Зелёный = profit, красный = loss, белый = 0                    │
│  Данные: GET /api/backtest/grid?follower=Lighter+Perp&signal=C  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.7 Страница Paper Trading (`/paper`)

```
┌── Status Bar ───────────────────────────────────────────────────┐
│  Strategy: current_strategy.py                                   │
│  Description: "EMA deviation, signal C, Lighter Perp, hold=30s" │
│  Loaded: 2026-04-12 09:15 UTC (2h ago)                         │
│  Status: 🟢 RUNNING  │  [⏹ Stop]  [▶ Start]                   │
└─────────────────────────────────────────────────────────────────┘

┌── P&L Cards (4) ───────────────────────────────────────────────┐
│  Net P&L    │ Trades   │ Hit Rate  │ Avg per trade              │
│  +18.3 bps  │ 7 (5W/2L)│ 71.4%     │ +2.61 bps                │
│  ($1.34)    │          │           │                            │
└─────────────────────────────────────────────────────────────────┘

┌── Equity Curve (Chart.js line, cumulative bps) ─────────────────┐
│  X: время trades, Y: cumulative P&L bps                         │
│  Одна зелёная линия, точки = trades                              │
└─────────────────────────────────────────────────────────────────┘

┌── Trade Log ────────────────────────────────────────────────────┐
│  Time     │ Signal│ Side│ Entry    │ Exit     │ P&L    │ Hold   │
│  14:32:05 │  C    │ BUY │ 73,067.5 │ 73,072.3 │ +6.57  │ 30.0s │
│  14:15:33 │  C    │ SELL│ 73,120.0 │ 73,115.1 │ +6.70  │ 30.0s │
│  13:58:22 │  C    │ BUY │ 73,045.0 │ 73,043.2 │ -2.47  │ 30.0s │
│  Данные: GET /api/paper/trades, обновление каждые 5s            │
└─────────────────────────────────────────────────────────────────┘

┌── Strategy History ─────────────────────────────────────────────┐
│  09:15 - v3: "EMA dev, signal C, hold=30s" — 5 trades, +18 bps │
│  Yesterday 14:30 - v2: "signal C, hold=10s" — 12 trades, +3 bps│
└─────────────────────────────────────────────────────────────────┘
```

### 7.8 Страница Data Files (`/files`)

```
┌── Storage Stats ────────────────────────────────────────────────┐
│  Total: 847 MB  │  Ticks: 612 MB (24 files)  │  BBO: 235 MB    │
│  Disk free: 14.2 / 40 GB  │  Oldest: 2026-04-11               │
└─────────────────────────────────────────────────────────────────┘

┌── Files Table ──────────────────────────────────────────────────┐
│  ☐ │ Type  │ File Name                    │ Size  │ Rows     │  │
│  ☐ │ ticks │ ticks_20260411_171417.parquet │ 28 MB │ 122,775  │  │
│  ☐ │ bbo   │ bbo_20260411_171417.parquet   │ 12 MB │ 358,246  │  │
│  ...                                                            │
│  [Select All]  [Delete Selected]  [Refresh]                     │
│  Данные: GET /api/files                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 7.9 Страница Settings (`/settings`)

```
┌── Collector Control ────────────────────────────────────────────┐
│  Status: 🟢 Running                                             │
│  [Restart Collector]  [Stop Collector]                          │
│  (POST /api/collector/restart, /api/collector/stop)             │
└─────────────────────────────────────────────────────────────────┘

┌── Venues ───────────────────────────────────────────────────────┐
│  ☑ OKX Perp (leader, 5.0 bps)                                  │
│  ☑ Bybit Perp (leader, 5.5 bps)                                │
│  ☑ Binance Perp (follower, 4.5 bps)                            │
│  ☐ Gate Perp (follower, 5.0 bps) — disabled                    │
│  ...                                                             │
│  [Save & Restart Collector]                                      │
└─────────────────────────────────────────────────────────────────┘

┌── System ───────────────────────────────────────────────────────┐
│  Server: 46.62.144.105  │  OS: Ubuntu (ARM64)                   │
│  CPU: 23%  │  RAM: 1.8 / 4.0 GB  │  Disk: 14.2 / 40 GB        │
│  Python: 3.11  │  Uptime: 3d 14h                                │
│  Обновление: GET /api/system каждые 10s                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. JAVASCRIPT — CLIENT SIDE

### 8.1 static/js/app.js

```javascript
/**
 * Главный JS файл dashboard.
 * WebSocket клиент + обновление DOM элементов.
 */

let ws = null;
let charts = {};  // name → Chart.js instance

function connectWS() {
    const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
    ws = new WebSocket(`${proto}//${location.host}/ws/live`);

    ws.onopen = () => {
        document.getElementById('ws-status').textContent = '●';
        document.getElementById('ws-status').style.color = 'var(--positive)';
    };

    ws.onclose = () => {
        document.getElementById('ws-status').textContent = '●';
        document.getElementById('ws-status').style.color = 'var(--negative)';
        setTimeout(connectWS, 3000);  // auto-reconnect
    };

    ws.onmessage = (event) => {
        const msg = JSON.parse(event.data);
        if (msg.type === 'status_update') {
            updateDashboard(msg.data);
        }
    };
}

function updateDashboard(status) {
    // 1. Sidebar
    updateSidebar(status);

    // 2. Если на странице overview — обновить таблицу и карточки
    if (document.getElementById('venue-table')) {
        updateVenueTable(status.venues);
        updateSummaryCards(status);
        updateErrorsList(status.errors_log);
    }
}

function updateSidebar(status) {
    const el = document.getElementById('sidebar-status');
    if (!el) return;
    const uptime = formatDuration(status.uptime_s);
    const totalTicks = Object.values(status.venues).reduce((s, v) => s + v.ticks, 0);
    const active = Object.values(status.venues).filter(v => v.status === 'ok').length;
    const total = Object.keys(status.venues).length;

    el.innerHTML = `
        <div class="status-item">
            <span class="status-dot ${active === total ? 'green' : 'yellow'}"></span>
            Collector: ${active}/${total}
        </div>
        <div class="status-item">⏱ ${uptime}</div>
        <div class="status-item">💾 ${status.files.total_mb} MB</div>
        <div class="status-item">📊 ${formatNumber(totalTicks)} ticks</div>
    `;
}

function updateVenueTable(venues) {
    const tbody = document.getElementById('venue-tbody');
    if (!tbody) return;

    const rows = Object.entries(venues)
        .sort((a, b) => b[1].ticks - a[1].ticks)
        .map(([name, v]) => {
            const icon = statusIcon(v.status, v.rate);
            const bbo = v.bbo > 0 ? '✓' : '–';
            const rateClass = v.rate < 1 && v.rate > 0 ? 'text-warning' : '';
            return `<tr class="venue-row" onclick="location.href='/venues/${encodeURIComponent(name)}'">
                <td>${icon}</td>
                <td>${name}</td>
                <td class="text-dim">${v.status === 'ok' ? 'leader' : 'follower'}</td>
                <td>${statusLabel(v.status, v.rate)}</td>
                <td class="num">${formatNumber(v.ticks)}</td>
                <td class="num ${rateClass}">${v.rate.toFixed(1)}</td>
                <td>${bbo}</td>
            </tr>`;
        }).join('');

    tbody.innerHTML = rows;
}

function statusIcon(status, rate) {
    if (status === 'ok' && rate > 1) return '<span class="dot green">●</span>';
    if (status === 'ok' && rate > 0) return '<span class="dot orange">●</span>';
    if (status === 'ok') return '<span class="dot gray">●</span>';
    if (status === 'reconnecting') return '<span class="dot yellow">●</span>';
    return '<span class="dot red">●</span>';
}

function statusLabel(status, rate) {
    if (status === 'ok' && rate > 1) return '<span class="text-positive">OK</span>';
    if (status === 'ok' && rate > 0) return '<span class="text-warning">LOW</span>';
    if (status === 'ok') return '<span class="text-dim">IDLE</span>';
    if (status === 'reconnecting') return '<span class="text-warning">REC</span>';
    if (status === 'connecting') return '<span class="text-dim">CONN</span>';
    return '<span class="text-negative">ERR</span>';
}

function formatNumber(n) {
    return n.toLocaleString('en-US');
}

function formatDuration(seconds) {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    if (h > 24) return `${Math.floor(h/24)}d ${h%24}h`;
    if (h > 0) return `${h}h ${m}m`;
    return `${m}m`;
}

// Start
document.addEventListener('DOMContentLoaded', connectWS);
```

### 8.2 static/js/charts.js

```javascript
/**
 * Chart.js обёртки для стандартных графиков dashboard.
 */

function createTickRateChart(canvasId, initialData) {
    /**
     * Line chart: tick rate per venue за последние 30 мин.
     * initialData: { "OKX Perp": [{minute_ts, rate}, ...], ... }
     */
    const ctx = document.getElementById(canvasId).getContext('2d');

    const colors = [
        '#4a6cf7', '#f7a04a', '#00c853', '#ff5252', '#ffa726',
        '#ab47bc', '#26c6da', '#8d6e63', '#78909c', '#d4e157',
        '#ec407a', '#7e57c2'
    ];

    const datasets = Object.entries(initialData).map(([venue, points], i) => ({
        label: venue,
        data: points.map(p => ({ x: p.minute_ts, y: p.rate })),
        borderColor: colors[i % colors.length],
        borderWidth: 1.5,
        pointRadius: 0,
        tension: 0.3,
        fill: false,
    }));

    return new Chart(ctx, {
        type: 'line',
        data: { datasets },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            animation: false,
            scales: {
                x: {
                    type: 'time',
                    time: { unit: 'minute', displayFormats: { minute: 'HH:mm' } },
                    grid: { color: '#222' },
                    ticks: { color: '#888', maxTicksLimit: 10 },
                },
                y: {
                    title: { display: true, text: 'ticks/sec', color: '#888' },
                    grid: { color: '#222' },
                    ticks: { color: '#888' },
                    beginAtZero: true,
                }
            },
            plugins: {
                legend: {
                    position: 'right',
                    labels: { color: '#ccc', boxWidth: 12, padding: 8, font: { size: 11 } },
                },
                tooltip: { mode: 'index', intersect: false },
            },
        }
    });
}

function createEquityCurve(canvasId, trades) {
    /**
     * Cumulative P&L line chart for paper trading.
     * trades: [{ ts, pnl_bps }, ...]
     */
    const ctx = document.getElementById(canvasId).getContext('2d');
    let cum = 0;
    const data = trades.map(t => {
        cum += t.pnl_bps;
        return { x: t.ts, y: cum };
    });

    return new Chart(ctx, {
        type: 'line',
        data: {
            datasets: [{
                label: 'Cumulative P&L (bps)',
                data: data,
                borderColor: '#00c853',
                borderWidth: 2,
                pointRadius: 4,
                pointBackgroundColor: data.map(d =>
                    d.y >= 0 ? '#00c853' : '#ff5252'),
                fill: false,
            }]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            animation: false,
            scales: {
                x: {
                    type: 'time',
                    grid: { color: '#222' },
                    ticks: { color: '#888' },
                },
                y: {
                    title: { display: true, text: 'bps', color: '#888' },
                    grid: { color: '#222' },
                    ticks: { color: '#888' },
                }
            },
            plugins: { legend: { display: false } },
        }
    });
}
```

---

## 9. JUPYTER WORKFLOW

### 9.1 notebooks/research.ipynb — структура

```python
# ═══ Ячейка 1: Setup ═══
import sys
sys.path.insert(0, '/home/leadlag/leadlag')
from core.data import load_ticks, load_bbo, venue_stats
import pandas as pd
import numpy as np

# Проверка что данные есть
stats = venue_stats()
print(stats)

# ═══ Ячейка 2: Загрузка данных ═══
ticks = load_ticks()  # или load_ticks("2026-04-12")
bbo = load_bbo()
print(f"Тиков: {len(ticks):,}, BBO: {len(bbo):,}")

# ═══ Ячейка 3: Бинирование (VWAP 50ms) ═══
# ... ваш код бинирования
# Результат: vwap_df — DataFrame columns=venues, index=bin_idx

# ═══ Ячейка 4: Детекция сигналов ═══
# ... EMA baseline, deviation, threshold crossing, кластеризация
# Результат: all_events — list of dicts

# ═══ Ячейка 5: Метрики ═══
# ... lag, hit, MFE/MAE для каждого follower

# ═══ Ячейка 6: Grid Search ═══
# ... delay × hold → P&L matrix
# ... bootstrap CI

# ═══ Ячейка 7: Визуализация ═══
# ... графики, heatmaps

# ═══ Ячейка 8: Сохранение результатов для Dashboard ═══
import json
results = {
    "updated": pd.Timestamp.now(tz='UTC').isoformat(),
    "dataset_hours": ...,
    "events_count": len(all_events),
    "signals": {"A": ..., "B": ..., "C": ...},
    "ci_table": ci_df.to_dict('records'),
    "grid": grid_df[grid_df['count'] >= 5].to_dict('records'),  # только значимые
}
with open('/home/leadlag/leadlag/data/backtest_results.json', 'w') as f:
    json.dump(results, f, default=str)
print("✅ Saved to data/backtest_results.json — dashboard обновится автоматически")

# ═══ Ячейка 9: Экспорт стратегии для Paper Trading ═══
# Запускается ТОЛЬКО когда довольны результатом

%%writefile /home/leadlag/leadlag/strategy/current_strategy.py
"""
Auto-exported from research.ipynb
Date: 2026-04-12
Strategy: EMA deviation, signal C, Lighter Perp
Backtest: +5.94 bps, 81% hit, N=73, CI=[+3.69, +8.03]
Hold: 30s, Delay: 0ms
"""
import numpy as np
from collections import deque

class LiveStrategy:
    # ... (полный класс стратегии — см. предыдущее описание)
    pass
```

### 9.2 Как Jupyter работает с dashboard

Jupyter записывает `data/backtest_results.json` → Dashboard читает этот файл для страниц Events и Backtest. Без API вызовов, без IPC — просто общий файл на диске.

Jupyter записывает `strategy/current_strategy.py` → Paper trader обнаруживает изменение через `stat().st_mtime`, перезагружает стратегию.

### 9.3 Доступ к Jupyter

VS Code Remote SSH уже настроен. Jupyter запущен как systemd service на порту 8888, привязан к localhost. VS Code подключается через SSH tunnel — порт 8888 автоматически форвардится. Jupyter доступен в браузере VS Code или в отдельной вкладке через `localhost:8888`.

---

## 10. PAPER TRADER

### 10.1 trader/paper_main.py

Полный код — см. описание в предыдущем разговоре. Ключевые моменты:

Отдельный процесс (systemd). Свои WebSocket подключения к биржам (не зависит от коллектора). Hot-reload стратегии: проверяет `mtime` файла `strategy/current_strategy.py` каждые 10 секунд. При изменении — закрывает открытую позицию, reload модуля, создаёт новый экземпляр. Логирует trades в `data/paper_trades/*.jsonl`. Пишет `data/paper_status.json` каждые 10 секунд (для dashboard).

### 10.2 Формат paper_status.json

```json
{
    "running": true,
    "strategy_file": "strategy/current_strategy.py",
    "strategy_loaded": "2026-04-12T09:15:00Z",
    "strategy_description": "EMA deviation, signal C, Lighter Perp, hold=30s",
    "total_trades": 7,
    "wins": 5,
    "losses": 2,
    "total_pnl_bps": 18.3,
    "avg_pnl_bps": 2.61,
    "hit_rate": 0.714,
    "open_position": null,
    "last_trade_ts": 1744483925000
}
```

---

## 11. SYSTEMD UNITS

### 11.1 leadlag-collector.service

```ini
[Unit]
Description=LeadLag Market Data Collector
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=leadlag
Group=leadlag
WorkingDirectory=/home/leadlag/leadlag
ExecStart=/home/leadlag/leadlag/.venv/bin/python -m collector.main
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=leadlag-collector
MemoryMax=2G
CPUQuota=80%
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

### 11.2 leadlag-dashboard.service

```ini
[Unit]
Description=LeadLag Dashboard
After=network.target leadlag-collector.service

[Service]
Type=simple
User=leadlag
WorkingDirectory=/home/leadlag/leadlag
ExecStart=/home/leadlag/leadlag/.venv/bin/python -m dashboard.server
Restart=always
RestartSec=5
SyslogIdentifier=leadlag-dashboard

[Install]
WantedBy=multi-user.target
```

### 11.3 leadlag-jupyter.service

```ini
[Unit]
Description=LeadLag JupyterLab
After=network.target

[Service]
Type=simple
User=leadlag
WorkingDirectory=/home/leadlag/leadlag
ExecStart=/home/leadlag/leadlag/.venv/bin/jupyter lab \
    --no-browser --port=8888 --ip=127.0.0.1 \
    --NotebookApp.token='' \
    --notebook-dir=/home/leadlag/leadlag/notebooks
Restart=always
RestartSec=5
SyslogIdentifier=leadlag-jupyter

[Install]
WantedBy=multi-user.target
```

### 11.4 leadlag-trader.service

```ini
[Unit]
Description=LeadLag Paper Trader
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=leadlag
WorkingDirectory=/home/leadlag/leadlag
ExecStart=/home/leadlag/leadlag/.venv/bin/python -m trader.paper_main
Restart=always
RestartSec=10
SyslogIdentifier=leadlag-trader

[Install]
WantedBy=multi-user.target
```

---

## 12. УСТАНОВКА

### 12.1 scripts/install.sh

```bash
#!/bin/bash
set -e
echo "═══ LeadLag System Installation ═══"

# Пользователь
sudo adduser --disabled-password --gecos "" leadlag 2>/dev/null || true
PROJECT=/home/leadlag/leadlag

# Системные пакеты
sudo apt update
sudo apt install -y python3 python3-venv python3-pip ufw

# Структура проекта
sudo -u leadlag mkdir -p $PROJECT/{config,parsers,collector,core,strategy,trader}
sudo -u leadlag mkdir -p $PROJECT/{dashboard/{templates,static/{css,js,lib}},notebooks,systemd,scripts}
sudo -u leadlag mkdir -p $PROJECT/data/paper_trades

# Python venv
sudo -u leadlag python3 -m venv $PROJECT/.venv
sudo -u leadlag $PROJECT/.venv/bin/pip install --upgrade pip
sudo -u leadlag $PROJECT/.venv/bin/pip install \
    websockets pyarrow pandas numpy duckdb psutil \
    tornado \
    jupyterlab ipywidgets tqdm plotly

# Systemd
sudo cp $PROJECT/systemd/*.service /etc/systemd/system/
sudo systemctl daemon-reload

# Sudoers для dashboard → collector restart
echo "leadlag ALL=(ALL) NOPASSWD: /bin/systemctl restart leadlag-collector, /bin/systemctl stop leadlag-collector, /bin/systemctl start leadlag-collector, /bin/systemctl restart leadlag-trader, /bin/systemctl stop leadlag-trader, /bin/systemctl start leadlag-trader" \
    | sudo tee /etc/sudoers.d/leadlag

# Firewall
sudo ufw default deny incoming
sudo ufw allow ssh
# ВАЖНО: замените YOUR_IP на ваш реальный IP
# sudo ufw allow from YOUR_IP to any port 8080
# sudo ufw allow from YOUR_IP to any port 8888
sudo ufw --force enable

# Запуск
sudo systemctl enable leadlag-collector leadlag-dashboard leadlag-jupyter
sudo systemctl start leadlag-collector leadlag-dashboard leadlag-jupyter

echo ""
echo "═══ Установка завершена ═══"
echo "Dashboard: http://46.62.144.105:8080"
echo "Jupyter:   через VS Code SSH tunnel (порт 8888)"
echo "Логи:      journalctl -u leadlag-collector -f"
echo "           journalctl -u leadlag-dashboard -f"
```

### 12.2 requirements.txt

```
websockets>=12.0
pyarrow>=15.0
pandas>=2.0
numpy>=1.26
duckdb>=0.10
psutil>=5.9
tornado>=6.4
jupyterlab>=4.0
ipywidgets>=8.0
tqdm>=4.65
plotly>=5.18
```

---

## 13. КАК ДОБАВИТЬ НОВУЮ БИРЖУ

```bash
# 1. Создать парсер
cat > parsers/kraken.py << 'EOF'
def parse_trade(msg, ts):
    if msg.get('feed') != 'trade': return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d.get('time', 0) * 1000),
             'price': float(d['price']), 'qty': float(d['qty']),
             'side': d['side']} for d in msg.get('trades', [])]

def parse_bbo(msg, ts):
    return None  # если нет BBO
EOF

# 2. Добавить в parsers/__init__.py
echo "from parsers import kraken" >> parsers/__init__.py

# 3. Добавить в config/venues.py
# 'Kraken Perp': VenueConfig(
#     name='Kraken Perp', role='follower',
#     ws_url='wss://futures.kraken.com/ws/v1',
#     subscribe_msg={"event":"subscribe","feed":"trade","product_ids":["PI_XBTUSD"]},
#     parser=kraken.parse_trade,
#     taker_fee_bps=5.0,
# ),

# 4. Перезапустить
sudo systemctl restart leadlag-collector
```

---

## 14. ПОРЯДОК РАЗРАБОТКИ — ЧТО ДАВАТЬ ИИ

### Промпт 1: Фундамент (Collector)

Дать ИИ: всю секцию 1 (структура), секцию 2 (config), секцию 3 (core), секцию 4 (parsers), секцию 5 (collector), секцию 11.1 (systemd unit), секцию 12 (install).

Задача: создать рабочий проект, который запускается через systemd и собирает данные в parquet.

Приложить: текущий рабочий код из ячеек 1-4 Notebook 1 (коллектор).

### Промпт 2: Dashboard

Дать ИИ: секцию 6 (dashboard backend), секцию 7 (экраны), секцию 8 (JavaScript), секцию 11.2 (systemd).

Задача: создать Tornado dashboard со всеми 7 страницами, WebSocket обновлениями, Chart.js графиками.

### Промпт 3: Jupyter + Strategy

Дать ИИ: секцию 9 (Jupyter workflow), секцию 10 (paper trader), секцию 11.3-11.4 (systemd).

Задача: создать шаблон research.ipynb, paper_main.py с hot-reload.

Приложить: текущий рабочий код из Notebook 2 (анализ) + Notebook 3 (визуализатор).

---

## 15. ЕЖЕДНЕВНАЯ РАБОТА — КАК ЭТИМ ПОЛЬЗОВАТЬСЯ

```
Утро:
  Открыть http://46.62.144.105:8080
  Посмотреть: все биржи зелёные? Тики идут? Ошибки?
  Посмотреть Paper Trading P&L за ночь.

Работа (в VS Code):
  Открыть notebooks/research.ipynb
  from core.data import load_ticks
  ticks = load_ticks()
  ... экспериментировать ...
  Сохранить backtest_results.json (ячейка 8)
  → Dashboard страница Backtest обновилась
  Если нашли улучшение → запустить %%writefile (ячейка 9)
  → Paper trader подхватит через 10 секунд

Проверка:
  Dashboard → Paper Trading → видите новые trades
  Если P&L стабильно > 0 за неделю → думать о live

Обслуживание:
  Dashboard → Data Files → удалить старые файлы
  Dashboard → Settings → добавить/убрать биржу → Restart Collector
  journalctl -u leadlag-collector -f  ← если нужны подробные логи
```

---

Это полный документ. Каждый файл, каждый экран, каждый endpoint описан. Можно давать ИИ по частям (3 промпта) и получить работающую систему.