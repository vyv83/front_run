# ФИНАЛЬНЫЙ ПЛАН ИССЛЕДОВАНИЯ v3 — Lead-Lag Follower Discovery

## 1. ПОЛНЫЙ СПИСОК КАНДИДАТОВ ВЕДОМЫХ

### Лидеры (сигнал) — из v2:
| # | Инструмент | Роль |
|---|-----------|------|
| L1 | OKX Perp BTC-USDT-SWAP | Leader |
| L2 | Bybit Perp BTCUSDT | Leader |

### Кандидаты ведомых — CEX Perpetuals:
| # | Инструмент | WS URL | Subscribe | Ping/Keepalive | Taker Fee |
|---|-----------|--------|-----------|----------------|-----------|
| F1 | MEXC Perp BTC_USDT | `wss://contract.mexc.com/edge` | `{"method":"sub.deal","param":{"symbol":"BTC_USDT"}}` | `{"method":"ping"}` каждые 10-20s | 0.02% (promo) |
| F2 | Bitget Perp BTCUSDT | `wss://ws.bitget.com/v2/ws/public` | `{"op":"subscribe","args":[{"instType":"USDT-FUTURES","channel":"trade","instId":"BTCUSDT"}]}` | `"ping"` текстовый каждые 30s | 0.06% |
| F3 | Gate.io Perp BTC_USDT | `wss://fx-ws.gateio.ws/v4/ws/usdt` | `{"time":<unix_ts>,"channel":"futures.trades","event":"subscribe","payload":["BTC_USDT"]}` | protocol-level ping + application `futures.ping` | 0.05% |

### Кандидаты ведомых — DEX Perpetuals:
| # | Инструмент | WS URL | Subscribe | Taker Fee |
|---|-----------|--------|-----------|-----------|
| F4 | Hyperliquid Perp BTC | `wss://api.hyperliquid.xyz/ws` | `{"method":"subscribe","subscription":{"type":"trades","coin":"BTC"}}` | 0.035% |
| F5 | dYdX v4 BTC-USD | `wss://indexer.dydx.trade/v4/ws` | `{"type":"subscribe","channel":"v4_trades","id":"BTC-USD","batched":false}` | 0% (promo) |
| F6 | Drift BTC-PERP | `wss://dlob.drift.trade/ws` | `{"type":"subscribe","channel":"trades","marketType":"perp","market":"BTC-PERP"}` | 0.05% |
| F7 | Lighter BTC-USD | `wss://mainnet.zklighter.elliot.ai/stream` | `{"type":"subscribe","channel":"trade/1"}` (market_id=1 для BTC, уточнить через REST) | 0% (zero-fee) |

### Кандидаты ведомых — Prediction Markets:
| # | Инструмент | WS URL | Subscribe | Особенности |
|---|-----------|--------|-----------|------------|
| F8 | Polymarket BTC 5min UP/DOWN | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | Подписка на текущий активный контракт по condition_id | Бинарный контракт, цена = вероятность |
| F9 | Kalshi BTC 15min | `wss://api.elections.kalshi.com/trade-api/ws/v2` | `{"id":1,"cmd":"subscribe","params":{"channels":["trade"],"market_tickers":["KXBTC15M-..."]}}` | Требует API key + RSA signature |

### Базовые (из v2, для референса):
| # | Инструмент |
|---|-----------|
| R1 | Binance Perp BTCUSDT |
| R2 | Binance Spot BTCUSDT |
| R3 | Bybit Spot BTCUSDT |

## 2. АРХИТЕКТУРА PIPELINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    ФАЗА 1: СБОР ДАННЫХ                          │
│                                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐       ┌─────────┐       │
│  │ OKX WS  │  │Bybit WS │  │MEXC WS  │  ...  │Drift WS │       │
│  └────┬────┘  └────┬────┘  └────┬────┘       └────┬────┘       │
│       │            │            │                  │             │
│       └────────────┴────────────┴──────────────────┘             │
│                         │                                        │
│                  asyncio.Queue()                                 │
│                         │                                        │
│                  ┌──────┴──────┐                                 │
│                  │  Writer     │                                 │
│                  │  (Parquet)  │                                 │
│                  └─────────────┘                                 │
│                                                                  │
│  Output: ticks_{session_id}.parquet                              │
│  Columns: [ts_ms, ts_exchange_ms, price, qty, side, venue]      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  ФАЗА 2: PREPROCESSING                           │
│                                                                  │
│  1. Дедупликация по (venue, ts_ms, price, qty)                  │
│  2. Бинирование: 50ms VWAP                                      │
│  3. ffill (max 500ms)                                            │
│  4. Log-returns: r = ln(P_t / P_{t-1})                          │
│  5. Разделение на leaders / followers                            │
│                                                                  │
│  Output: bins_{session_id}.parquet                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               ФАЗА 3: CROSS-CORRELATION ANALYSIS                 │
│                                                                  │
│  Для каждой пары (leader, follower):                             │
│    • Pearson cross-corr с лагами [-40..+40] бинов               │
│    • Оптимальный лаг + 95% CI (bootstrap)                        │
│    • Granger causality test (10 лагов)                           │
│                                                                  │
│  Матрица: optimal_lag[leader][follower]                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               ФАЗА 4: EVENT-BASED ANALYSIS                       │
│                                                                  │
│  Event Detection (лидер):                                        │
│    Δprice ≥ threshold за rolling window (200ms)                  │
│    thresholds: [3, 5, 7, 10, 15, 20] bps                        │
│    cooldown: 2000ms между событиями                               │
│                                                                  │
│  Signal variants:                                                │
│    A: OKX only        B: Bybit only                              │
│    C: Confirmed (оба в течение 200ms)                            │
│                                                                  │
│  Для каждого event × follower:                                   │
│    • lag_ms: время до 50% reaction                               │
│    • full_reaction_ms: время до 80% reaction                     │
│    • max_favorable_excursion (MFE)                               │
│    • max_adverse_excursion (MAE)                                 │
│    • optimal_entry_delay: argmax(P&L) от 0 до 5000ms            │
│    • optimal_hold_time: argmax(P&L) от entry до 30000ms         │
│    • realized_pnl_at_T для T ∈ [200,500,1k,2k,5k,10k,30k] ms  │
│    • net_pnl = realized_pnl - 2×taker_fee (вход + выход)        │
│                                                                  │
│  Output: events_{session_id}.parquet                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 ФАЗА 5: СТАТИСТИЧЕСКИЙ ОТЧЁТ                     │
│                                                                  │
│  Для каждого follower:                                           │
│    • N events                                                    │
│    • Hit rate (% событий, где follower реагирует в том же        │
│      направлении)                                                │
│    • Mean/Median lag ± 95% CI                                    │
│    • Mean net P&L per event ± 95% CI                             │
│    • Sharpe-like ratio: mean(pnl) / std(pnl)                    │
│    • Optimal threshold (какой bps даёт лучший net P&L)           │
│    • Optimal signal (A/B/C)                                      │
│    • Optimal entry delay                                         │
│    • Optimal hold time                                           │
│                                                                  │
│  Composite Follower Score:                                       │
│    40% net_P&L + 25% hit_rate + 20% signal_stability +          │
│    15% opportunity_window_duration                               │
│                                                                  │
│  Ранжированная таблица ведомых                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              ФАЗА 6: ИНТЕРАКТИВНЫЙ ВИДЖЕТ                        │
│                                                                  │
│  Standalone HTML (Plotly.js + vanilla JS)                        │
│                                                                  │
│  Layout:                                                         │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ [Event #] [◀ Prev] [Next ▶] [Leader: ▼] [Follower: ▼]│     │
│  │ [Threshold: ▼] [Signal: A/B/C ▼]                     │     │
│  ├──────────────────────────────────────────────────────┤       │
│  │                                                       │       │
│  │   Leader VWAP (50ms bins)                             │       │
│  │   ═══════════════════════════════════════════════     │       │
│  │   [price chart with event marker]                     │       │
│  │                                                       │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │                                                       │       │
│  │   Follower VWAP (50ms bins)                           │       │
│  │   ═══════════════════════════════════════════════     │       │
│  │   [price chart with reaction marker]                  │       │
│  │                                                       │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │  ┌─────────────┐  ┌──────────────────────────┐       │       │
│  │  │ Event Stats  │  │  Distribution Plots      │       │       │
│  │  │ Lag: 350ms   │  │  [Lag histogram]          │       │       │
│  │  │ React: 78%   │  │  [P&L histogram]          │       │       │
│  │  │ P&L: +2.3bps │  │                           │       │       │
│  │  │ MFE: +5.1bps │  │                           │       │       │
│  │  │ MAE: -1.2bps │  │                           │       │       │
│  │  └─────────────┘  └──────────────────────────┘       │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
│  Функции:                                                        │
│    • Синхронизованный zoom/pan (shared x-axis = time)            │
│    • Окно: ±5 секунд от события (настраиваемо)                   │
│    • Вертикальные линии: event_time (красная), reaction_time     │
│      (зелёная)                                                   │
│    • Shaded area: opportunity window                             │
│    • Hover: ts_ms, price, qty на каждом бине                     │
│    • Keyboard: ←/→ для prev/next event                           │
└─────────────────────────────────────────────────────────────────┘
```

## 3. WS CONFIGS — ПОЛНЫЙ СЛОВАРЬ

```python
WS_CONFIGS = {
    # === LEADERS ===
    'OKX Perp': {
        'url': 'wss://ws.okx.com:8443/ws/v5/public',
        'subscribe_msg': {"op": "subscribe", "args": [{"channel": "trades", "instId": "BTC-USDT-SWAP"}]},
        'parser': 'okx',
        'keepalive': 'okx_text_ping',  # текстовый "ping" каждые 25s
        'role': 'leader',
        'taker_fee_bps': 5.0,
    },
    'Bybit Perp': {
        'url': 'wss://stream.bybit.com/v5/public/linear',
        'subscribe_msg': {"op": "subscribe", "args": ["publicTrade.BTCUSDT"]},
        'parser': 'bybit',
        'keepalive': 'ws_ping',  # WS-level ping
        'role': 'leader',
        'taker_fee_bps': 5.5,
    },
    
    # === REFERENCE (from v2) ===
    'Binance Perp': {
        'url': 'wss://fstream.binance.com/ws/btcusdt@trade',
        'subscribe_msg': None,
        'parser': 'binance',
        'keepalive': None,  # Binance auto-pings
        'role': 'reference',
        'taker_fee_bps': 4.5,
    },
    'Binance Spot': {
        'url': 'wss://stream.binance.com:9443/ws/btcusdt@trade',
        'subscribe_msg': None,
        'parser': 'binance',
        'keepalive': None,
        'role': 'reference',
        'taker_fee_bps': 10.0,
    },
    'Bybit Spot': {
        'url': 'wss://stream.bybit.com/v5/public/spot',
        'subscribe_msg': {"op": "subscribe", "args": ["publicTrade.BTCUSDT"]},
        'parser': 'bybit',
        'keepalive': 'ws_ping',
        'role': 'reference',
        'taker_fee_bps': 10.0,
    },
    
    # === FOLLOWER CANDIDATES: CEX ===
    'MEXC Perp': {
        'url': 'wss://contract.mexc.com/edge',
        'subscribe_msg': {"method": "sub.deal", "param": {"symbol": "BTC_USDT"}},
        'parser': 'mexc',
        'keepalive': 'mexc_json_ping',  # {"method":"ping"} каждые 15s
        'role': 'follower',
        'taker_fee_bps': 2.0,  # промо, может измениться
    },
    'Bitget Perp': {
        'url': 'wss://ws.bitget.com/v2/ws/public',
        'subscribe_msg': {"op": "subscribe", "args": [{"instType": "USDT-FUTURES", "channel": "trade", "instId": "BTCUSDT"}]},
        'parser': 'bitget',
        'keepalive': 'text_ping',  # "ping" каждые 30s
        'role': 'follower',
        'taker_fee_bps': 6.0,
    },
    'Gate Perp': {
        'url': 'wss://fx-ws.gateio.ws/v4/ws/usdt',
        'subscribe_msg': None,  # Формируется динамически с time
        'subscribe_fn': 'gate_subscribe',  # специальная функция
        'parser': 'gate',
        'keepalive': 'gate_ping',  # {"channel":"futures.ping"}
        'role': 'follower',
        'taker_fee_bps': 5.0,
    },
    
    # === FOLLOWER CANDIDATES: DEX ===
    'Hyperliquid Perp': {
        'url': 'wss://api.hyperliquid.xyz/ws',
        'subscribe_msg': {"method": "subscribe", "subscription": {"type": "trades", "coin": "BTC"}},
        'parser': 'hyperliquid',
        'keepalive': None,
        'role': 'follower',
        'taker_fee_bps': 3.5,
    },
    'dYdX Perp': {
        'url': 'wss://indexer.dydx.trade/v4/ws',
        'subscribe_msg': {"type": "subscribe", "channel": "v4_trades", "id": "BTC-USD", "batched": False},
        'parser': 'dydx',
        'keepalive': None,  # проверить, возможно нужен ping
        'role': 'follower',
        'taker_fee_bps': 0.0,  # промо zero-fee
    },
    'Drift Perp': {
        'url': 'wss://dlob.drift.trade/ws',
        'subscribe_msg': {"type": "subscribe", "channel": "trades", "marketType": "perp", "market": "BTC-PERP"},
        'parser': 'drift',
        'keepalive': None,
        'role': 'follower',
        'taker_fee_bps': 5.0,
    },
    'Lighter Perp': {
        'url': 'wss://mainnet.zklighter.elliot.ai/stream',
        'subscribe_msg': None,  # market_id определяется динамически
        'subscribe_fn': 'lighter_subscribe',
        'parser': 'lighter',
        'keepalive': 'ws_ping',  # любой фрейм каждые 2 мин
        'role': 'follower',
        'taker_fee_bps': 0.0,
    },
}
```

## 4. ПАРСЕРЫ — СПЕЦИФИКАЦИИ

```python
def parse_mexc(msg, ts_ms):
    """MEXC push.deal → list of ticks"""
    # msg: {"symbol":"BTC_USDT","data":[{"p":115309.8,"v":55,"T":2,"t":1755487578276,"i":...}],"channel":"push.deal","ts":...}
    ticks = []
    for d in msg.get('data', []):
        ticks.append({
            'ts_ms': ts_ms,
            'ts_exchange_ms': d['t'],
            'price': float(d['p']),
            'qty': float(d['v']),  # NB: это контракты, не BTC! Нужен contract_size
            'side': 'buy' if d['T'] == 1 else 'sell',
        })
    return ticks

def parse_bitget(msg, ts_ms):
    """Bitget trade push → list of ticks"""
    # msg: {"action":"snapshot","arg":{...},"data":[{"ts":"...","price":"27000.5","size":"0.001","side":"buy","tradeId":"..."}],"ts":...}
    ticks = []
    for d in msg.get('data', []):
        ticks.append({
            'ts_ms': ts_ms,
            'ts_exchange_ms': int(d['ts']),
            'price': float(d['price']),
            'qty': float(d['size']),
            'side': d['side'],
        })
    return ticks

def parse_gate(msg, ts_ms):
    """Gate.io futures.trades → list of ticks"""
    # msg: {"time":...,"time_ms":...,"channel":"futures.trades","event":"update","result":[{"contract":"BTC_USDT","size":10,"id":...,"create_time_ms":...,"price":"95000.5"}]}
    ticks = []
    for d in msg.get('result', []):
        size = d['size']
        ticks.append({
            'ts_ms': ts_ms,
            'ts_exchange_ms': d.get('create_time_ms', int(d.get('create_time', 0)) * 1000),
            'price': float(d['price']),
            'qty': abs(size),  # Gate: positive=buy, negative=sell
            'side': 'buy' if size > 0 else 'sell',
        })
    return ticks

def parse_dydx(msg, ts_ms):
    """dYdX v4_trades → list of ticks"""
    # msg: {"type":"channel_data","channel":"v4_trades","id":"BTC-USD","contents":{"trades":[{"size":"0.01","price":"95000","side":"BUY","createdAt":"2025-...","type":"LIMIT"}]}}
    ticks = []
    contents = msg.get('contents', {})
    trades = contents.get('trades', [])
    for d in trades:
        ticks.append({
            'ts_ms': ts_ms,
            'ts_exchange_ms': 0,  # createdAt is ISO string, parse if needed
            'price': float(d['price']),
            'qty': float(d['size']),
            'side': d['side'].lower(),
        })
    return ticks

def parse_drift(msg, ts_ms):
    """Drift trades channel → list of ticks"""
    # NB: data may be double-encoded JSON string!
    data = msg.get('data', '{}')
    if isinstance(data, str):
        import json
        data = json.loads(data)
    # data format TBD — test empirically
    # Expected: {"trades": [{"price": "95000", "size": "0.01", "side": "buy", ...}]}
    ticks = []
    trades = data if isinstance(data, list) else data.get('trades', [data])
    for d in trades:
        ticks.append({
            'ts_ms': ts_ms,
            'ts_exchange_ms': d.get('ts', 0),
            'price': float(d.get('price', 0)),
            'qty': float(d.get('size', d.get('baseAssetAmount', 0))),
            'side': d.get('side', d.get('takerSide', 'unknown')).lower(),
        })
    return ticks

def parse_lighter(msg, ts_ms):
    """Lighter trade channel → list of ticks"""
    # msg: {"channel":"trade:1","trades":[{"price":"95000.50","size":"0.01","is_maker_ask":false,...,"timestamp":1773854156654}],"type":"update/trade"}
    ticks = []
    for d in msg.get('trades', []):
        ticks.append({
            'ts_ms': ts_ms,
            'ts_exchange_ms': d.get('timestamp', 0),
            'price': float(d['price']),
            'qty': float(d['size']),
            'side': 'sell' if d.get('is_maker_ask', False) else 'buy',
        })
    return ticks
```

## 5. МЕТРИКИ ВЕДОМОГО — ФОРМАЛЬНЫЕ ОПРЕДЕЛЕНИЯ

Для каждого event `e` на лидере `L` и ведомого `F`:

**Определения:**
- `t0` = момент события на лидере (бин, где |Δprice/price| ≥ threshold)
- `direction` = sign(Δprice лидера) ∈ {+1, -1}
- `magnitude_L` = |Δprice лидера| за detection window
- `P_F(t)` = VWAP ведомого в бине `t`

**Метрики:**

```
1. lag_50_ms = min{τ : |P_F(t0+τ) - P_F(t0)| ≥ 0.5 × magnitude_L × sign_correct}
   где sign_correct = 1 если direction × (P_F(t0+τ) - P_F(t0)) > 0

2. lag_80_ms = min{τ : |P_F(t0+τ) - P_F(t0)| ≥ 0.8 × magnitude_L × sign_correct}

3. hit = 1 если direction × (P_F(t0+2000ms) - P_F(t0)) > 0, else 0

4. reaction_pct(T) = (P_F(t0+T) - P_F(t0)) / magnitude_L × direction
   для T ∈ {200, 500, 1000, 2000, 5000, 10000, 30000} ms

5. MFE = max over t∈[t0, t0+30s] of: direction × (P_F(t) - P_F(t0))
   Maximum Favorable Excursion

6. MAE = min over t∈[t0, t0+30s] of: direction × (P_F(t) - P_F(t0))
   Maximum Adverse Excursion (будет ≤ 0 если цена ходила против)

7. optimal_entry_delay = argmax over d∈[0,5000ms]:
     mean_over_events(direction × (P_F(t0+d+hold) - P_F(t0+d)))
   для hold = optimal_hold_time

8. optimal_hold_time = argmax over h∈[100ms, 30000ms]:
     mean_over_events(direction × (P_F(t0+entry+h) - P_F(t0+entry)) - 2×taker_fee)

9. net_pnl(entry_delay, hold) = 
     direction × (P_F(t0+entry_delay+hold) - P_F(t0+entry_delay)) / P_F(t0) × 10000  [bps]
     - 2 × taker_fee_bps
```

## 6. PLAN ВЫПОЛНЕНИЯ — ШАГ ЗА ШАГОМ

### Шаг 1: Notebook 1 — Collector
```
Вход: WS_CONFIGS
Процесс:
  1. Инициализировать asyncio event loop
  2. Для каждого venue создать WS task с:
     - auto-reconnect (max 5 retries, exponential backoff)
     - keepalive scheduler
     - parse → queue.put()
  3. Writer task: читает из queue, батчит по 10000 тиков, пишет в parquet
  4. Heartbeat monitor: каждые 30s выводит кол-во тиков / venue
  5. Graceful shutdown после COLLECT_SECONDS

Выход: ticks_{session_id}.parquet
Параметры: COLLECT_SECONDS = 7200 (2 часа минимум)
```

### Шаг 2: Notebook 2 — Preprocessing + Cross-Correlation
```
Вход: ticks_{session_id}.parquet
Процесс:
  1. Дедупликация
  2. Статистика тиков: count, rate/s по venue
  3. Биннинг 50ms VWAP + ffill
  4. Cross-correlation matrix (leaders vs followers)
  5. Granger causality tests
  6. Heatmap лагов

Выход: bins_{session_id}.parquet, correlation_report.html
```

### Шаг 3: Notebook 3 — Event Analysis
```
Вход: bins_{session_id}.parquet
Процесс:
  1. Event detection по 3 signal variants × 6 thresholds
  2. Для каждого event: все метрики §5
  3. Sweep optimal parameters (entry_delay × hold_time grid)
  4. Bootstrap CI (1000 iter, 95%)
  5. Ранжирование followers

Выход: events_{session_id}.parquet, follower_ranking.html
```

### Шаг 4: Notebook 4 — Widget
```
Вход: bins_{session_id}.parquet, events_{session_id}.parquet
Процесс:
  1. Подготовить JSON с events + surrounding bin data
  2. Сгенерировать standalone HTML с Plotly.js
  3. Интерактивные controls (см. §2 Фаза 6)

Выход: explorer.html (открывается в браузере)
```

## 7. PREDICTION MARKETS — ОТДЕЛЬНЫЙ PIPELINE

```
Шаг P1: Collector для Polymarket/Kalshi
  - Polymarket: подписка на orderbook_delta текущего BTC 5min контракта
  - Kalshi: подписка на trade + orderbook_delta для KXBTC15M

Шаг P2: Анализ
  - При event на лидере → зафиксировать mid-price YES контракта
  - Отслеживать mid-price через 1,2,5,10,30,60 секунд
  - Если leader moved UP → YES probability должна расти
  - Теоретический P&L = (buy YES at event+lag) × (sell at peak) - fees
  - Учесть: бинарный контракт settles at $1 or $0 через 5/15 мин

Шаг P3: Отчёт с рекомендациями
```

## 8. ИЗВЕСТНЫЕ ОГРАНИЧЕНИЯ И РИСКИ

| # | Риск | Митигация |
|---|------|-----------|
| 1 | MEXC переехал на protobuf → JSON WS не работает | Тестировать при подключении, fallback=исключить |
| 2 | Lighter market_id для BTC неизвестен | REST `/api/v1/markets` при инициализации |
| 3 | Drift trades — через DLOB сервер, не raw engine | Ожидать больший лаг, учитывать в анализе |
| 4 | Gate.io subscribe требует `time` field | Динамическая генерация subscribe msg |
| 5 | dYdX batched mode | Ставим `batched: false` |
| 6 | Polymarket контракт меняется каждые 5 мин | Авто-обнаружение нового контракта через REST |
| 7 | Kalshi требует RSA auth | Пользователь должен иметь API key |
| 8 | Результаты специфичны для сетевой топологии | Зафиксировать: откуда запускается, traceroute |
| 9 | 2 часа может быть мало для надёжной статистики | Поддержка merge сессий, рекомендация 4-8 часов |
| 10 | Комиссии могут меняться (промо-периоды) | Параметризовать, обновить перед запуском |

## 9. EXPECTED OUTPUT

После завершения исследования вы получите:

1. **Ранжированную таблицу** ведомых с composite score
2. **Оптимальные параметры** для каждого ведомого: signal type, threshold, entry delay, hold time
3. **Net P&L distribution** с bootstrap CI для топ-3 ведомых
4. **Интерактивный explorer** (HTML) для визуального подтверждения lead-lag паттернов
5. **Recommendation** — конкретный инструмент + параметры для торговли

---

**Это полный план на 10/10. Готов писать код для любого из шагов. С какого начнём?**