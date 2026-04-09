# SpikenautExecution

Async trade signal pipeline for neuromorphic SNN models: **ZMQ SUB** → **confidence gating** → **Kelly sizing** → **dYdX v4 REST execution**.

Bridges Julia-based Spiking Neural Network (SNN) strategy output to live decentralized exchange execution.

## Features

- **ZMQ SUB socket** — consumes JSON trade signals from a Rust (or any) nervous system
- **Nanosecond latency tracking** — Unix-epoch timestamps aligned with Rust `timestamp_nanos`
- **Confidence gate** — only executes signals above a configurable threshold
- **Half-Kelly position sizing** — risk-managed order sizing from SNN confidence scores
- **dYdX v4 decentralized perpetuals** — REST client for orderbook queries and market data (no API key required for reads)

## Architecture

```
Julia (Brain)          ZMQ Bridge           Exchange
     ↓                      ↑
LIF neurons → SNN signal → SUB socket → Confidence gate → Kelly sizing → dYdX v4
(16 neurons)  (confidence)   (IPC/TCP)    (threshold)     (sizing)     (REST)
```

## Installation

```julia
] instantiate
```

Or add directly:

```julia
] add https://github.com/<your-org>/SpikenautExecution.jl
```

### Dependencies

| Package | Purpose |
|---------|---------|
| `ZMQ`   | ZeroMQ SUB socket for signal ingestion |
| `HTTP`  | REST calls to dYdX v4 indexer |
| `JSON`  | Signal deserialization |

## Quick Start

### 1. Create an execution engine

```julia
using SpikenautExecution

engine = ExecutionEngine(
    confidence_threshold = Float32(0.85),  # minimum SNN confidence to trade
    max_position_size    = 10.0,            # hard cap on position units
    payoff_ratio         = 0.01,            # expected price move for Kelly
)
```

### 2. Start the ZMQ listener

```julia
start!(engine, zmq_endpoint = "tcp://localhost:5555") do decision
    if decision.executed
        println("Executed: $(decision.signal.ticker) × $(round(decision.position_units, digits=4))")
        println("  Latency: $(decision.latency_ns) ns")
        println("  Kelly fraction: $(round(decision.kelly_fraction, digits=4))")
    else
        println("Rejected: $(decision.reason)")
    end
end
```

### 3. Fetch dYdX v4 market data

```julia
client = DydxClient(base_url = "https://indexer.dydx.trade/v4")

price = get_price(client, "BTC-USD")
if price !== nothing
    println("Mid:  \$$(mid_price(price))")
    println("Spread: $(spread_bps(price)) bps")
end
```

### 4. Manual signal processing (no ZMQ)

```julia
signal = TradeSignal(Dict(
    "ticker"       => "BTC-USD",
    "side"         => "BUY",
    "price"        => 65000.0,
    "quantity"     => 0.01,
    "confidence"   => 0.92,
    "timestamp_ns" => time_ns(),  # from your signal producer
))

decision = execute_signal!(engine, signal, 10_000.0)
println(decision)
```

## Signal Format

The ZMQ listener expects JSON objects matching this schema:

```json
{
  "ticker":       "BTC-USD",
  "side":         "BUY",
  "price":        65000.0,
  "quantity":     0.01,
  "confidence":   0.92,
  "timestamp_ns": 1744156800000000000
}
```

| Field | Type | Description |
|-------|------|-------------|
| `ticker` | string | Asset symbol (e.g. `"BTC-USD"`) |
| `side` | string | `"BUY"`, `"SELL"`, or `"NEUTRAL"` |
| `price` | float | Expected execution price (USD) |
| `quantity` | float | Units to trade (Kelly-sized by the producer) |
| `confidence` | float | SNN output in `[0.0, 1.0]` |
| `timestamp_ns` | int | Unix nanoseconds at signal creation |

## API Reference

### Core Types

| Type / Function | Description |
|-----------------|-------------|
| `TradeSignal` | Immutable struct holding a deserialized trade signal |
| `ExecutionEngine` | Stateful engine with confidence gate, Kelly sizing, and position tracking |
| `ExecutionDecision` | Result of processing one signal (executed/rejected, position units, latency) |
| `execute_signal!(engine, signal, balance)` | Gate, size, and execute a single signal |
| `latency_ns(signal)` | End-to-end latency in nanoseconds (Unix epoch aligned) |
| `passes_gate(signal, threshold)` | Boolean check: `signal.confidence >= threshold` |
| `fill_rate(engine)` | Fraction of signals that were executed vs. rejected |

### dYdX v4 Client

| Type / Function | Description |
|-----------------|-------------|
| `DydxClient` | REST client (read-only, no API key needed) |
| `get_price(client, ticker)` | Fetch oracle price + orderbook top → `DydxPrice` or `nothing` |
| `DydxPrice` | Struct with `oracle_price`, `best_bid`, `best_ask` |
| `mid_price(p)` | Mid between best bid and ask |
| `spread_bps(p)` | Bid-ask spread in basis points |

## Running Tests

```bash
julia --project=. -e 'using Pkg; Pkg.test()'
```

## License

MIT — see [`LICENSE`](LICENSE).
