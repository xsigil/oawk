# oawk

A modern, window-aware text stream processing engine implemented in OCaml.

`oawk` retains the one-liner simplicity and pipeline integration of POSIX AWK while introducing **native row-window addressing (`$k[offset]`)** for stateful metrics and streaming analytics.

---

## Key Features

- **Row-Window Addressing (`$k[offset]`)**: Access prior (`$2[-1]`) or subsequent (`$2[+1]`) rows natively without manual state or buffer management.
- **Window Slices & Aggregates**: Compute rolling metrics directly with slice syntax (`avg($2[-2..0])`).
- **Bounded Ring-Buffer Streaming**: Bounded memory footprint ($O(W)$ where $W$ is the window width) powered by OCaml, suitable for infinite streams.
- **Zero-Dependency Native Binary**: Fast startup and predictable execution.

---

## Quick Example

### Spike Detection
Detect a 20+ jump in column 2 compared to the immediately preceding row:

```bash
cat metrics.log | oawk '$2 - $2[-1] > 20 { print $1, "SPIKE:", $2[-1], "->", $2 }'

```

### Rolling Average

Calculate a 3-row moving average:

```bash
cat data.tsv | oawk 'NR >= 3 { print $1, avg($2[-2..0]) }'

```

---

## Specification

See [SPEC.md](SPEC.md) for the complete design, syntax, and execution model.

## License

MIT
