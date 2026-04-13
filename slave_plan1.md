

Окей. Полный самодостаточный план, который можно дать ИИ как единственный документ.

---

# ПЛАН ИССЛЕДОВАНИЯ: Lead-Lag Follower Discovery v5

## 0. ПРЕДЫСТОРИЯ И КОНТЕКСТ

### Что уже сделано
Проведено исследование first mover на 6 биржах. Собраны тики через WebSocket, сделано бинирование 50ms VWAP, адаптивная детекция событий (σ-based), кластеризация. Результат — таблица first mover:

```
OKX Perp           29.4%   ← ЛИДЕР
Bybit Perp         28.7%   ← ЛИДЕР
Binance Perp       21.0%
Binance Spot       12.6%
Hyperliquid Perp    4.9%
Bybit Spot          3.5%
```

Параметры из first mover анализа (использовать как baseline):
```
σ за бин (50ms):           0.000014
σ за 10 бинов (500ms):     0.000044 (0.44 bps)
Порог детекции:            1.5σ = 0.0066%
Окно детекции:             500ms (10 бинов)
Кластерный зазор:          3000ms
Результат:                 143 кластера (событий)
```

### Что нужно сделать
Подключить биржи-кандидаты на роль follower. Собрать данные одновременно с лидерами. Измерить lag, hit rate, P&L. Найти лучшего follower для торговли.

### Цель
Конкретная рекомендация: "Торгуйте [инструмент X] по сигналу [A/B/C] с входом через [D ms] и выходом через [H ms]. Ожидаемый P&L = [N bps/event], hit rate = [M%], 95% CI = [...]"

---

## 1. СОСТАВ БИРЖ

### 1.1 Лидеры (источник сигнала)

| ID | Venue | Инструмент | WS URL | Subscribe | Taker Fee |
|----|-------|-----------|--------|-----------|-----------|
| L1 | OKX Perp | BTC-USDT-SWAP | `wss://ws.okx.com:8443/ws/v5/public` | `{"op":"subscribe","args":[{"channel":"trades","instId":"BTC-USDT-SWAP"}]}` | 5.0 bps |
| L2 | Bybit Perp | BTCUSDT | `wss://stream.bybit.com/v5/public/linear` | `{"op":"subscribe","args":["publicTrade.BTCUSDT"]}` | 5.5 bps |

### 1.2 Кандидаты followers

| ID | Venue | Тип | WS URL | Subscribe (trades) | Taker Fee | Статус | Ожидаемая причина отставания |
|----|-------|-----|--------|--------------------|-----------|--------|------------------------------|
| F1 | Binance Perp | CEX | `wss://fstream.binance.com/stream?streams=btcusdt@trade/btcusdt@bookTicker` | Не нужен (combined stream) | 4.5 bps | ✅ Работает, 33/s | 79% случаев не first mover; глубокая ликвидность замедляет ценовую реакцию |
| F2 | Binance Spot | CEX | `wss://stream.binance.com:9443/stream?streams=btcusdt@trade/btcusdt@bookTicker` | Не нужен (combined stream) | 10.0 bps | ✅ Работает, 21/s | Спот всегда отстаёт от фьючерсов (фьючерсы = леверидж) |
| F3 | Bybit Spot | CEX | `wss://stream.bybit.com/v5/public/spot` | `{"op":"subscribe","args":["publicTrade.BTCUSDT"]}` | 10.0 bps | ✅ Работает, 8/s | Спот vs фьючерсы |
| F4 | MEXC Perp | CEX | `wss://contract.mexc.com/edge` | `{"method":"sub.deal","param":{"symbol":"BTC_USDT"}}` | 2.0 bps | ✅ Работает, 9/s | Второй эшелон CEX, мало маркет-мейкеров, медленнее арбитраж |
| F5 | Bitget Perp | CEX | `wss://ws.bitget.com/v2/ws/public` | `{"op":"subscribe","args":[{"instType":"USDT-FUTURES","channel":"trade","instId":"BTCUSDT"}]}` | 6.0 bps | ✅ Работает, 12/s | Второй эшелон CEX |
| F6 | Gate Perp | CEX | `wss://fx-ws.gateio.ws/v4/ws/usdt` | ДИНАМИЧЕСКИЙ (см. §3.3) | 5.0 bps | ✅ Работает, 4/s | Третий эшелон, малый объём |
| F7 | Hyperliquid Perp | DEX | `wss://api.hyperliquid.xyz/ws` | `{"method":"subscribe","subscription":{"type":"trades","coin":"BTC"}}` | 3.5 bps | ✅ Работает, 14/s | DEX, on-chain matching latency |
| F8 | Lighter Perp | DEX | `wss://mainnet.zklighter.elliot.ai/stream` | ДИНАМИЧЕСКИЙ (market_id через REST, см. §3.3) | 0.0 bps | 🔸 НЕ тестирован | DEX, zkRollup proof latency |
| F9 | edgeX Perp | DEX | `wss://quote.edgex.exchange/api/v1/public/ws` | `{"type":"subscribe","channel":"trades.10000001"}` | 2.6 bps | 🔸 НЕ тестирован | DEX, offchain matching + zk settlement |
| F10 | Aster Perp | DEX | `wss://fstream.asterdex.com/ws/btcusdt@aggTrade/btcusdt@bookTicker` | Не нужен (combined stream, Binance-fork) | 2.0 bps | 🔸 НЕ тестирован | DEX на BSC, блокчейн latency; aggTrade агрегируется каждые 100ms |

### 1.3 BBO (Best Bid/Offer) стримы — где доступны

BBO нужен для реалистичного P&L (вход по ask, выход по bid). Собирать на том же WS где trades, если биржа поддерживает.

| Venue | BBO channel | Subscribe | Парсер |
|-------|------------|-----------|--------|
| OKX Perp | bbo-tbt | `{"op":"subscribe","args":[{"channel":"bbo-tbt","instId":"BTC-USDT-SWAP"}]}` | parse_okx_bbo |
| Bybit Perp | orderbook.1 | `{"op":"subscribe","args":["orderbook.1.BTCUSDT"]}` | parse_bybit_bbo |
| Binance Perp | bookTicker | В combined stream URL | parse_binance_bbo |
| Binance Spot | bookTicker | В combined stream URL | parse_binance_bbo |
| Bybit Spot | orderbook.1 | `{"op":"subscribe","args":["orderbook.1.BTCUSDT"]}` | parse_bybit_bbo |
| Bitget Perp | books1 | `{"op":"subscribe","args":[{"instType":"USDT-FUTURES","channel":"books1","instId":"BTCUSDT"}]}` | parse_bitget_bbo |
| Lighter Perp | ticker/{MKT_ID} | ДИНАМИЧЕСКИЙ | parse_lighter_bbo |
| edgeX Perp | bookTicker.all.1s | `{"type":"subscribe","channel":"bookTicker.all.1s"}` | parse_edgex_bbo |
| Aster Perp | bookTicker | В combined stream URL | parse_aster_bbo |
| MEXC Perp | — | Нет простого BBO | — |
| Gate Perp | — | Нет простого BBO | — |
| Hyperliquid | — | Нет BBO stream | — |

Для бирж без BBO: в анализе использовать VWAP как proxy для entry price. P&L будет оптимистичным — отметить в отчёте.

### 1.4 Убраны из исследования

| Venue | Причина |
|-------|---------|
| Drift (Solana) | 0 тиков за 30s теста — мёртвая ликвидность BTC-PERP |
| dYdX v4 | 13 тиков за 2 мин (0.1/s) — слишком мало для 50ms бинов; stale snapshot при подписке |
| HTX (Huobi) | gzip-compressed WS — лишняя сложность; ликвидность ≈ Bitget — избыточен |
| BingX | Нестабильный API, документация хаотичная, эндпоинт менялся |
| Paradex | Требует bearer auth для public trades |
| GRVT | Требует API key для WS |

---

## 2. АРХИТЕКТУРА КОЛЛЕКТОРА

### 2.1 VenueConfig — единая структура для любой биржи

```python
@dataclass
class VenueConfig:
    name: str                          # уникальное имя, напр. 'OKX Perp'
    role: str                          # 'leader' | 'follower'
    
    # --- Trades WebSocket ---
    ws_url: str                        # wss://...
    subscribe_msg: Any                 # dict | None | 'DYNAMIC'
    subscribe_factory: Callable = None # вызывается если subscribe_msg == 'DYNAMIC'
    parser: Callable = None            # fn(msg_dict, ts_local_ms) → list[tick_dict]
    
    # --- BBO (опционально, тот же WS) ---
    bbo_subscribe_msg: Any = None      # None = нет BBO для этой биржи
    bbo_parser: Callable = None        # fn(msg_dict, ts_local_ms) → bbo_dict | None
    
    # --- Keepalive ---
    keepalive_type: str = None         # 'text_ping' | 'json_ping' | 'ws_ping' | 'edgex_pong' | None
    keepalive_msg: Any = None          # payload для json_ping
    keepalive_interval: int = 0        # секунды, 0 = нет keepalive
    
    # --- Метаданные ---
    taker_fee_bps: float = 0.0
    contract_size: float = 1.0         # множитель qty для пересчёта в BTC
```

### 2.2 Registry — добавление биржи

```python
VENUES: dict[str, VenueConfig] = {}

def register_venue(cfg: VenueConfig):
    VENUES[cfg.name] = cfg
```

Для добавления новой биржи: написать парсер, вызвать `register_venue()`, перезапустить. Никаких изменений в engine/writer/analysis.

### 2.3 Потоки данных

```
12 venues × 1 WS каждый
         │
         │ каждый WS task парсит ОБА типа сообщений:
         │   trades → parser() → trades_queue
         │   BBO    → bbo_parser() → bbo_queue
         │
    ┌────┴─────┐        ┌────┴─────┐
    │ trades_q │        │  bbo_q   │
    └────┬─────┘        └────┬─────┘
         │                    │
    ┌────┴─────┐        ┌────┴─────┐
    │ Writer 1 │        │ Writer 2 │
    │ flush    │        │ flush    │
    │ 5s/5000  │        │ 5s/5000  │
    └────┬─────┘        └────┬─────┘
         │                    │
  ticks_{id}.parquet   bbo_{id}.parquet
```

### 2.4 Parquet Schemas

**ticks_{session_id}.parquet** (trades):
```
ts_ms           INT64     — local timestamp, time.time_ns()//1_000_000
ts_exchange_ms  INT64     — exchange-provided timestamp (0 если нет)
price           FLOAT64   — trade price
qty             FLOAT64   — trade size в BTC (после contract_size пересчёта)
side            STRING    — 'buy' | 'sell'
venue           STRING    — venue name из VenueConfig.name
```

**bbo_{session_id}.parquet** (best bid/offer):
```
ts_ms           INT64     — local timestamp
bid_price       FLOAT64
bid_qty         FLOAT64
ask_price       FLOAT64
ask_qty         FLOAT64
venue           STRING
```

---

## 3. ПАРСЕРЫ — ПОЛНЫЕ СПЕЦИФИКАЦИИ

### 3.1 Trades парсеры

Каждый парсер: `fn(msg: dict, ts: int) → list[dict]` где dict содержит ключи: `ts_ms, ts_exchange_ms, price, qty, side`.

```python
# ═══════════════ ЛИДЕРЫ ═══════════════

def parse_okx_trade(msg, ts):
    """OKX: {"arg":{"channel":"trades"},"data":[{"px":"72980","sz":"0.02","side":"sell","ts":"1775843339639"}]}"""
    if msg.get('arg',{}).get('channel') != 'trades' or 'data' not in msg:
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d['ts']),
             'price': float(d['px']), 'qty': float(d['sz']),
             'side': d['side']} for d in msg['data']]

def parse_bybit_trade(msg, ts):
    """Bybit: {"topic":"publicTrade.BTCUSDT","data":[{"T":1775843338932,"p":"72976.9","v":"0.001","S":"Sell"}]}"""
    if not msg.get('topic','').startswith('publicTrade') or 'data' not in msg:
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d['T']),
             'price': float(d['p']), 'qty': float(d['v']),
             'side': d['S'].lower()} for d in msg['data']]

# ═══════════════ FOLLOWERS: CEX ═══════════════

def parse_binance_trade(msg, ts):
    """Binance (combined stream): {"stream":"btcusdt@trade","data":{"e":"trade","T":...,"p":"72961.6","q":"0.009","m":true}}
    ФИКС: фильтр price=0, qty=0 (мусорные сообщения обнаружены при тесте)"""
    data = msg.get('data', msg)
    if data.get('e') != 'trade':
        return []
    p, q = float(data['p']), float(data['q'])
    if p <= 0 or q <= 0:
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(data['T']),
             'price': p, 'qty': q,
             'side': 'sell' if data.get('m') else 'buy'}]

def parse_mexc_trade(msg, ts):
    """MEXC: {"channel":"push.deal","data":[{"p":72961.7,"v":8500,"T":1,"t":1775843338704}]}
    ФИКС: qty в контрактах! 1 контракт = 0.0001 BTC. Тест показал qty=8500 (= 0.85 BTC)"""
    if msg.get('channel') != 'push.deal':
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d['t']),
             'price': float(d['p']),
             'qty': float(d['v']) * 0.0001,  # контракты → BTC
             'side': 'buy' if d.get('T') == 1 else 'sell'}
            for d in msg.get('data', [])]

def parse_bitget_trade(msg, ts):
    """Bitget: {"arg":{"channel":"trade"},"data":[{"ts":"...","price":"72958.0","size":"0.0001","side":"buy"}]}"""
    if msg.get('arg',{}).get('channel') != 'trade' or 'data' not in msg:
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d['ts']),
             'price': float(d['price']), 'qty': float(d['size']),
             'side': d['side'].lower()} for d in msg['data']]

def parse_gate_trade(msg, ts):
    """Gate.io: {"channel":"futures.trades","event":"update","result":[{"price":"95000.5","size":10,"create_time_ms":...}]}
    Gate: size>0 = buy, size<0 = sell. 1 контракт BTC_USDT = 0.0001 BTC"""
    if msg.get('channel') != 'futures.trades' or msg.get('event') != 'update':
        return []
    ticks = []
    for d in msg.get('result', []):
        size = d.get('size', 0)
        ts_ex = d.get('create_time_ms', int(d.get('create_time', 0)) * 1000)
        ticks.append({'ts_ms': ts, 'ts_exchange_ms': int(ts_ex),
                      'price': float(d['price']),
                      'qty': abs(size) * 0.0001,
                      'side': 'buy' if size > 0 else 'sell'})
    return ticks

# ═══════════════ FOLLOWERS: DEX ═══════════════

def parse_hyperliquid_trade(msg, ts):
    """Hyperliquid: {"channel":"trades","data":[{"px":"72979.0","sz":"0.00144","side":"b","time":1775843328581}]}
    ФИКС: side может быть 'b'/'a' вместо 'buy'/'sell' (обнаружено при тесте)"""
    if msg.get('channel') != 'trades':
        return []
    ticks = []
    for d in msg.get('data', []):
        raw = str(d.get('side', '')).lower()
        if raw in ('b', 'buy', 'long'):
            side = 'buy'
        elif raw in ('a', 'sell', 'short'):
            side = 'sell'
        else:
            side = 'unknown'
        ticks.append({'ts_ms': ts, 'ts_exchange_ms': int(d.get('time', 0)),
                      'price': float(d['px']), 'qty': float(d['sz']),
                      'side': side})
    return ticks

def parse_lighter_trade(msg, ts):
    """Lighter: {"type":"update/trade","channel":"trade:1","trades":[{"price":"95000.50","size":"0.01","is_maker_ask":false,"timestamp":1773854156654}]}
    is_maker_ask=false → taker купил (buy). is_maker_ask=true → taker продал (sell)."""
    if msg.get('type') != 'update/trade':
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d.get('timestamp', 0)),
             'price': float(d['price']), 'qty': float(d['size']),
             'side': 'sell' if d.get('is_maker_ask') else 'buy'}
            for d in msg.get('trades', [])]

def parse_edgex_trade(msg, ts):
    """edgeX: {"type":"quote-event","channel":"trades.10000001","content":{"data":[{"price":"30065.12","size":"0.01","time":"1688365544504","isBuyerMaker":false}]}}
    isBuyerMaker=false → taker купил. contractId 10000001 = BTCUSD."""
    if msg.get('type') != 'quote-event':
        return []
    content = msg.get('content', {})
    if not content.get('channel', '').startswith('trades.'):
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d.get('time', 0)),
             'price': float(d['price']), 'qty': float(d['size']),
             'side': 'sell' if d.get('isBuyerMaker') else 'buy'}
            for d in content.get('data', [])]

def parse_aster_trade(msg, ts):
    """Aster (Binance fork): {"stream":"btcusdt@aggTrade","data":{"e":"aggTrade","T":...,"p":"72961.6","q":"0.009","m":true}}
    ВАЖНО: aggTrade агрегируется каждые 100ms. Минимальный наблюдаемый lag = 100ms."""
    data = msg.get('data', msg)
    if data.get('e') != 'aggTrade':
        return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(data['T']),
             'price': float(data['p']), 'qty': float(data['q']),
             'side': 'sell' if data.get('m') else 'buy'}]
```

### 3.2 BBO парсеры

Каждый парсер: `fn(msg: dict, ts: int) → dict | None` где dict содержит: `ts_ms, bid_price, bid_qty, ask_price, ask_qty`. Возвращает None если сообщение не BBO.

```python
def parse_okx_bbo(msg, ts):
    """OKX bbo-tbt: {"arg":{"channel":"bbo-tbt"},"data":[{"bids":[["72980","1.2",...]],"asks":[["72981","0.5",...]],"ts":"..."}]}"""
    if msg.get('arg',{}).get('channel') != 'bbo-tbt' or 'data' not in msg:
        return None
    d = msg['data'][0]
    return {'ts_ms': ts,
            'bid_price': float(d['bids'][0][0]), 'bid_qty': float(d['bids'][0][1]),
            'ask_price': float(d['asks'][0][0]), 'ask_qty': float(d['asks'][0][1])}

def parse_bybit_bbo(msg, ts):
    """Bybit orderbook.1: {"topic":"orderbook.1.BTCUSDT","data":{"b":[["72976","0.5"]],"a":[["72977","0.3"]]}}"""
    if not msg.get('topic','').startswith('orderbook.1') or 'data' not in msg:
        return None
    d = msg['data']
    b, a = d.get('b', []), d.get('a', [])
    if not b or not a:
        return None
    return {'ts_ms': ts,
            'bid_price': float(b[0][0]), 'bid_qty': float(b[0][1]),
            'ask_price': float(a[0][0]), 'ask_qty': float(a[0][1])}

def parse_binance_bbo(msg, ts):
    """Binance bookTicker (combined): {"stream":"btcusdt@bookTicker","data":{"b":"72960","B":"1.5","a":"72961","A":"0.8"}}
    Отличаем от trade: у bookTicker нет поля 'e'='trade', но есть 'b' и 'a'."""
    data = msg.get('data', msg)
    if data.get('e') in ('trade', 'aggTrade') or 'b' not in data or 'a' not in data:
        return None
    return {'ts_ms': ts,
            'bid_price': float(data['b']), 'bid_qty': float(data['B']),
            'ask_price': float(data['a']), 'ask_qty': float(data['A'])}

def parse_bitget_bbo(msg, ts):
    """Bitget books1: {"arg":{"channel":"books1"},"data":[{"bids":[["72958","0.5"]],"asks":[["72959","0.3"]]}]}"""
    if msg.get('arg',{}).get('channel') != 'books1' or 'data' not in msg:
        return None
    d = msg['data'][0]
    b, a = d.get('bids', []), d.get('asks', [])
    if not b or not a:
        return None
    return {'ts_ms': ts,
            'bid_price': float(b[0][0]), 'bid_qty': float(b[0][1]),
            'ask_price': float(a[0][0]), 'ask_qty': float(a[0][1])}

def parse_lighter_bbo(msg, ts):
    """Lighter ticker: {"type":"update/ticker","ticker":{"s":"BTC","a":{"price":"95000","size":"0.5"},"b":{"price":"94999","size":"0.8"}}}"""
    if msg.get('type') != 'update/ticker':
        return None
    t = msg.get('ticker', {})
    b, a = t.get('b', {}), t.get('a', {})
    if not b.get('price') or not a.get('price'):
        return None
    return {'ts_ms': ts,
            'bid_price': float(b['price']), 'bid_qty': float(b['size']),
            'ask_price': float(a['price']), 'ask_qty': float(a['size'])}

def parse_edgex_bbo(msg, ts):
    """edgeX bookTicker: {"type":"quote-event","channel":"bookTicker.all.1s","content":{"data":[{"contractId":"10000001","bestBidPrice":"30000","bestBidSize":"2.5","bestAskPrice":"30001","bestAskSize":"1.8"}]}}
    Фильтруем только contractId=10000001 (BTCUSD)."""
    if msg.get('type') != 'quote-event':
        return None
    content = msg.get('content', {})
    if not content.get('channel', '').startswith('bookTicker'):
        return None
    for d in content.get('data', []):
        if str(d.get('contractId')) == '10000001':
            bp, ap = float(d.get('bestBidPrice',0)), float(d.get('bestAskPrice',0))
            if bp > 0 and ap > 0:
                return {'ts_ms': ts,
                        'bid_price': bp, 'bid_qty': float(d.get('bestBidSize',0)),
                        'ask_price': ap, 'ask_qty': float(d.get('bestAskSize',0))}
    return None

def parse_aster_bbo(msg, ts):
    """Aster bookTicker (Binance fork): идентичен parse_binance_bbo."""
    return parse_binance_bbo(msg, ts)
```

### 3.3 Динамические subscribe фабрики

```python
def make_gate_subscribe():
    """Gate.io: subscribe требует поле time с текущим unix timestamp."""
    return {"time": int(time.time()), "channel": "futures.trades",
            "event": "subscribe", "payload": ["BTC_USDT"]}

def make_lighter_subscribe():
    """Lighter: market_id для BTC определяется через REST /api/v1/markets.
    Fallback: market_id=1 (по документации 0=ETH, предположительно 1=BTC).
    Возвращает СПИСОК subscribe сообщений (trades + ticker BBO)."""
    import urllib.request
    btc_id = 1  # fallback
    try:
        resp = urllib.request.urlopen(
            'https://mainnet.zklighter.elliot.ai/api/v1/markets', timeout=10)
        markets = json.loads(resp.read())
        for m in markets:
            if 'BTC' in m.get('symbol', '').upper():
                btc_id = m.get('market_index', m.get('market_id', 1))
                break
    except Exception:
        pass
    return [
        {"type": "subscribe", "channel": f"trade/{btc_id}"},      # trades
        {"type": "subscribe", "channel": f"ticker/{btc_id}"},     # BBO
    ]
```

### 3.4 Keepalive спецификации

| Тип | Что делать | Интервал | Биржи |
|-----|-----------|----------|-------|
| `text_ping` | `await ws.send("ping")` | OKX: 25s, Bitget: 30s | OKX, Bitget |
| `json_ping` | `await ws.send(json.dumps(msg))` | MEXC: 15s, Gate: 15s | MEXC (msg=`{"method":"ping"}`), Gate (msg=`{"channel":"futures.ping"}`) |
| `ws_ping` | Через параметр `ping_interval` websockets | Bybit: 20s, Lighter: 60s | Bybit Perp, Bybit Spot, Lighter |
| `edgex_pong` | При получении `{"type":"ping","time":"..."}` ответить `{"type":"pong","time":"..."}` | Реактивный (не по таймеру) | edgeX |
| `None` | Ничего, биржа сама пингует | — | Binance Perp/Spot, Hyperliquid, Aster |

### 3.5 Фильтрация служебных сообщений

В основном цикле ws_task перед вызовом парсеров — пропускать:
```python
# Пропуск bytes
if isinstance(raw, bytes): continue

# Пропуск текстовых pong/ping
if raw in ('pong', '"pong"', 'ping'): continue

# После json.loads(raw) → msg:
if isinstance(msg, dict):
    # Subscribe confirmations
    if msg.get('event') in ('subscribe', 'subscribed', 'info', 'pong'): continue
    if msg.get('op') == 'pong' or msg.get('ret_msg') == 'pong': continue
    if msg.get('type') in ('connected', 'pong', 'subscribed'): continue
    # Ping channels
    if msg.get('channel', '').endswith(('.ping', '.pong')): continue
    # MEXC pong
    if msg.get('data') == 'pong' or msg.get('method') == 'pong': continue

    # edgeX: application-level ping → отвечаем pong
    if msg.get('type') == 'ping':
        await ws.send(json.dumps({"type": "pong", "time": msg.get("time", "")}))
        continue
```

---

## 4. ASYNC ENGINE

### 4.1 ws_venue_task (один на venue)

```
Параметры: cfg: VenueConfig, trades_queue, bbo_queue, stop_event
Retry: max 5, exponential backoff (2^n секунд, max 30)

Алгоритм:
1. Connect:
   - ping_interval=20 если keepalive_type='ws_ping', иначе None
   - ping_timeout=10, close_timeout=5, max_size=10MB, open_timeout=15

2. Subscribe:
   - Если subscribe_msg == 'DYNAMIC': вызвать subscribe_factory()
     - Может вернуть dict (один subscribe) или list[dict] (несколько)
   - Если subscribe_msg == dict: послать json.dumps(subscribe_msg)
   - Если subscribe_msg == None: ничего (combined stream)
   
3. BBO subscribe (если bbo_subscribe_msg задан):
   - Аналогично trades subscribe
   - Lighter: уже включён в результат make_lighter_subscribe()

4. Start keepalive task (если keepalive_type не None и не 'ws_ping' и не 'edgex_pong')

5. Message loop:
   a. raw = await ws.recv()
   b. ts_local = int(time.time_ns() // 1_000_000)
   c. Фильтрация служебных (§3.5)
   d. msg = json.loads(raw)
   e. trades = cfg.parser(msg, ts_local) → каждый tick добавить venue=cfg.name → trades_queue
   f. if cfg.bbo_parser: bbo = cfg.bbo_parser(msg, ts_local) → если не None, добавить venue → bbo_queue
   g. Обновить tick_counts[cfg.name]

6. On disconnect: retry
```

### 4.2 Writer task

```
Параметры: queue, stop_event, parquet_path, schema
Буфер: list
Flush условие: len(buffer) >= 5000 ИЛИ time_since_last_flush > 5s ИЛИ (stop_event.is_set() AND buffer)
При flush: pyarrow.Table.from_pydict() → ParquetWriter.write_table()
Compression: zstd
```

### 4.3 Heartbeat task

```
Каждые 5 секунд:
  - Обновить прогресс-бар (tqdm): n=elapsed, postfix=total ticks, written, active venues

Каждые 30 секунд:
  - Полный отчёт по venue: count, rate/s, status, errors
  - Маркеры: ✅ (ok + ticks>0), 🔄 (ok + ticks=0), ❌ (ошибка)
```

### 4.4 run_collector (main)

```
1. Сброс счётчиков
2. Создать trades_queue, bbo_queue (maxsize=500_000), stop_event
3. Для каждого venue в VENUES: создать asyncio.create_task(ws_venue_task)
4. trades_writer = create_task(writer_task(trades_queue, ..., ticks_path, TICK_SCHEMA))
5. bbo_writer = create_task(writer_task(bbo_queue, ..., bbo_path, BBO_SCHEMA))
6. heartbeat = create_task(heartbeat_task)
7. await asyncio.sleep(COLLECT_SECONDS)
8. stop_event.set()
9. Cancel WS tasks, await writers, cancel heartbeat
10. Итоговая статистика
```

---

## 5. СИГНАЛЫ

### 5.1 Определения

```
Сигнал A (OKX):      OKX Perp VWAP двинулся ≥ threshold за detection_window
Сигнал B (Bybit):    Bybit Perp VWAP двинулся ≥ threshold за detection_window
Сигнал C (confirmed): ОБА двинулись ≥ threshold, второй не позже 500ms после первого
```

### 5.2 Параметры (из first mover анализа)

```
bin_size         = 50ms
detection_window = 500ms (10 бинов)
threshold        = адаптивный, 1.5 × σ_window (σ_window = σ_bin × √10)
cluster_gap      = 3000ms (события ближе 3s = одно событие)
```

### 5.3 Детекция

```python
# Для каждого лидера:
# 1. Вычислить log-returns: r[t] = ln(vwap[t] / vwap[t-1])
# 2. σ_bin = std(r) за всю сессию
# 3. σ_window = σ_bin × sqrt(detection_window / bin_size)
# 4. threshold = 1.5 × σ_window
# 5. Для каждого бина t:
#    Δ = (vwap[t] - vwap[t - window]) / vwap[t - window]
#    Если |Δ| ≥ threshold → event(t, sign(Δ))
# 6. Кластеризация: если events[i].t - events[i-1].t < cluster_gap → объединить
```

---

## 6. МЕТРИКИ FOLLOWER

Для каждого event `e` на лидере и follower `F`:

```
Определения:
  t0        = момент события на лидере (бин)
  direction = sign(Δ лидера) ∈ {+1, -1}
  magnitude = |Δ лидера| за detection_window
  P_F(t)    = VWAP follower в бине t

Метрики:
  1. lag_50_ms = min{τ > 0 : direction × (P_F(t0+τ) - P_F(t0)) ≥ 0.5 × magnitude}
     Время до 50% реакции. None если не достигнуто за 30s.

  2. lag_80_ms = min{τ > 0 : direction × (P_F(t0+τ) - P_F(t0)) ≥ 0.8 × magnitude}

  3. hit = 1 если direction × (P_F(t0+2000ms) - P_F(t0)) > 0, иначе 0

  4. MFE = max over t∈[t0, t0+30s]: direction × (P_F(t) - P_F(t0))
     Maximum Favorable Excursion

  5. MAE = min over t∈[t0, t0+30s]: direction × (P_F(t) - P_F(t0))
     Maximum Adverse Excursion (≤ 0)

  6. pnl(d, h) = direction × (P_F(t0+d+h) - P_F(t0+d)) / P_F(t0+d) × 10000 - 2 × taker_fee_bps
     P&L в bps при входе через d ms, удержании h ms

  7. pnl_realistic(d, h) — для бирж с BBO:
     Вход = ask_price (для buy) или bid_price (для sell) в момент t0+d
     Выход = bid_price (для sell) или ask_price (для buy) в момент t0+d+h
     pnl = direction × (exit_price - entry_price) / entry_price × 10000 - 2 × taker_fee_bps

  8. optimal_delay = argmax over d ∈ {0, 50, 100, 150, ..., 5000} ms:
     mean(pnl(d, optimal_hold))

  9. optimal_hold = argmax over h ∈ {100, 200, 500, 1000, 2000, 5000, 10000, 30000} ms:
     mean(pnl(optimal_delay, h))
```

### Grid search

```
Двумерный grid: delay × hold
  delay: [0, 50, 100, 200, 300, 500, 750, 1000, 1500, 2000, 3000, 5000] ms (12 значений)
  hold:  [100, 200, 500, 1000, 2000, 5000, 10000, 30000] ms (8 значений)
  = 96 комбинаций на follower × signal
  
Для каждой комбинации: mean(net_pnl), median, std, hit_rate, count
Optimal = комбинация с max mean(net_pnl) при count ≥ 20
```

---

## 7. COMPOSITE FOLLOWER SCORE

```
Score = 35% × norm(mean_net_pnl)          — сколько зарабатываем (bps)
      + 25% × hit_rate                     — как часто правы
      + 20% × (1 - cv_lag)                — стабильность лага (cv = std_lag / mean_lag, capped [0,1])
      + 20% × norm(opportunity_window)     — длительность окна входа в ms

norm() = (x - min) / (max - min) по всем followers
opportunity_window = optimal_delay + optimal_hold
```

---

## 8. PIPELINE — ШАГ ЗА ШАГОМ

### Notebook 1: Collector

| Ячейка | Содержание | Выход |
|--------|-----------|-------|
| 1 | Импорты (websockets, pyarrow, asyncio, tqdm, nest_asyncio), VenueConfig dataclass, register_venue(), SESSION_ID, PARQUET paths, schemas | — |
| 2 | Все парсеры §3.1 + §3.2 + фабрики §3.3 | — |
| 3 | Полный WS_CONFIGS dict — register_venue() для каждой из 12 бирж | Печать: "12 бирж зарегистрировано: 2 leaders, 10 followers" |
| 4 | Async engine: ws_venue_task, writer_task, heartbeat_task, run_collector | — |
| 5 | Тест 60s — ТОЛЬКО Lighter + edgeX + Aster (3 новых DEX) + диагностика | Таблица: venue / ticks / rate / lag / issues |
| 6 | Тест 60s — ВСЕ 12 бирж + полная диагностика | Таблица всех 12 venue, проверка quality |
| 7 | Боевой сбор. `COLLECT_SECONDS = 14400` (4 часа рекомендовано). Прогресс-бар. | ticks_{id}.parquet, bbo_{id}.parquet |

### Notebook 2: Analysis

| Ячейка | Содержание | Выход |
|--------|-----------|-------|
| 1 | Загрузка parquet, статистика: ticks/venue, rate, timespan, BBO coverage | Таблица venue stats |
| 2 | Бинирование 50ms VWAP, ffill (max 500ms), log-returns | bins DataFrame |
| 3 | Детекция событий на L1+L2, три сигнала A/B/C, кластеризация | events DataFrame, count per signal |
| 4 | Для каждого signal × follower: lag_50, lag_80, hit, MFE, MAE | Матрица лагов |
| 5 | Grid search delay × hold → pnl matrix | Heatmap optimal params |
| 6 | Bootstrap CI (1000 iter, 95%) для net_pnl и hit_rate | CI bounds |
| 7 | Composite score, ранжирование | **Финальная таблица followers** |
| 8 | Текстовая рекомендация: лучший follower + сигнал + параметры | Recommendation string |

### Notebook 3: Визуализация

| Ячейка | Содержание | Выход |
|--------|-----------|-------|
| 1 | Подготовка JSON: top-N events × лидер + followers bins ±10s | events_vis.json |
| 2 | Генерация standalone HTML (Plotly.js + vanilla JS) | explorer.html |

Виджет explorer.html:
```
┌──────────────────────────────────────────────────────────┐
│ [Event #] [◄ Prev] [Next ►] [Signal: A/B/C ▼]          │
│ [Leader: ▼] [Follower: ▼]                               │
├──────────────────────────────────────────────────────────┤
│ Leader VWAP chart (50ms bins)                            │
│   — красная вертикаль: момент события                    │
│   — shaded area: detection window                        │
├──────────────────────────────────────────────────────────┤
│ Follower VWAP chart (50ms bins)                          │
│   — зелёная вертикаль: момент реакции (lag_50)           │
│   — shaded: opportunity window                           │
├──────────────────────────────────────────────────────────┤
│ ┌─── Stats ───┐  ┌─── Distributions ───────────────┐    │
│ │ Lag: 350ms   │  │ [Lag histogram across events]   │    │
│ │ Hit: 78%     │  │ [P&L histogram across events]   │    │
│ │ P&L: +2.3bps │  │                                 │    │
│ │ MFE: +5.1bps │  │                                 │    │
│ │ MAE: -1.2bps │  │                                 │    │
│ └──────────────┘  └─────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
Keyboard: ← → для prev/next, синхронизованный zoom/pan
```

---

## 9. DELIVERABLES

| # | Артефакт | Формат |
|---|---------|--------|
| 1 | Сырые данные | ticks_{id}.parquet, bbo_{id}.parquet |
| 2 | Ранжированная таблица 10 followers | DataFrame + printed table |
| 3 | Для топ-3 followers: оптимальный сигнал, delay, hold, net P&L ± 95% CI, hit rate, Sharpe | Structured report |
| 4 | Конкретная рекомендация | "Торгуйте X по сигналу Y, вход через D ms, выход через H ms, P&L = N bps" |
| 5 | Интерактивный explorer | explorer.html |

---

## 10. РИСКИ И МИТИГАЦИИ

| # | Риск | Митигация |
|---|------|-----------|
| 1 | Lighter/edgeX/Aster не подключаются или 0 тиков | Отдельный тест 60s (ячейка 5). Если 0 — исключить, продолжить с остальными |
| 2 | Lighter market_id для BTC неизвестен | REST /api/v1/markets при инициализации + fallback=1 |
| 3 | edgeX keepalive: app-level ping/pong | Обработка в основном цикле: при type=ping → отвечать pong |
| 4 | Aster aggTrade агрегация 100ms | Учитывать: минимальный наблюдаемый lag ≥ 100ms. Не баг. |
| 5 | MEXC переехал на protobuf | Если JSON не парсится → исключить |
| 6 | Мало событий (< 100) | Увеличить время сбора до 8h или merge сессий |
| 7 | Все followers быстрее 50ms | Значит latency arb невозможен с данного сервера. Попробовать colocation. |
| 8 | Net P&L < 0 после комиссий везде | Честный результат. Стратегия не работает. |
| 9 | Результаты зависят от сетевого расположения | Записать: где запускается, traceroute до каждой биржи |
| 10 | Промо-комиссии могут измениться | Параметризованы в VenueConfig.taker_fee_bps, легко обновить |

---

## 11. БЕСШОВНОЕ ДОБАВЛЕНИЕ НОВОЙ БИРЖИ

Пример — добавить Kraken Futures:

```python
# Шаг 1: Парсер (5-10 строк)
def parse_kraken_trade(msg, ts):
    if msg.get('feed') != 'trade': return []
    return [{'ts_ms': ts, 'ts_exchange_ms': int(d.get('time', 0) * 1000),
             'price': float(d['price']), 'qty': float(d['qty']),
             'side': d['side']} for d in msg.get('trades', [])]

# Шаг 2: Регистрация
register_venue(VenueConfig(
    name='Kraken Perp', role='follower',
    ws_url='wss://futures.kraken.com/ws/v1',
    subscribe_msg={"event":"subscribe","feed":"trade","product_ids":["PI_XBTUSD"]},
    parser=parse_kraken_trade,
    keepalive_type='ws_ping', keepalive_interval=30,
    taker_fee_bps=5.0,
))

# Шаг 3: Перезапустить collector. Всё.
# Engine, writer, analysis — без изменений.
```