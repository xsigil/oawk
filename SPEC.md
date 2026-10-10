# oawk Specification (Draft v0.1.0)

**oawk** is a modern, statically sound, column-and-window-oriented text stream processing engine implemented in OCaml. It retains the ubiquitous one-liner usability of POSIX AWK while introducing **native row-window addressing (`$k[offset]`)** and pure, deterministic stream semantics.

---

## 1. Core Philosophy & Motivation

1. **Keep the One-Liner Soul**:
   No heavy setups, no Python/Pandas dependencies, and no bloated SQL engines. If you can pipe it, you can process it.
2. **Break AWK's One-Line-at-a-Time Trap**:
   In traditional AWK, calculating diffs, moving averages, or state transitions requires tedious ring buffer state management. `oawk` introduces first-class **row-window addressing** directly into field syntax.
3. **Soundness & Zero-Cost Streaming**:
   Implemented in OCaml with an internal bounded ring-buffer. Memory consumption remains $O(W)$ (where $W$ is the maximum window span), achieving predictable performance and infinite-stream suitability.

---

## 2. Syntax & Field/Window Model

### 2.1 Standard Field Syntax (AWK-Compatible)
- `$0`: Current entire line/record.
- `$1, $2, ..., $NF`: 1-based positional fields.
- `FS`: Field separator (default: whitespace, customizable via `-F`).
- `NR`: Total records processed so far.
- `NF`: Number of fields in the current record.

### 2.2 Row-Window Addressing (The Game Changer)
Fields support an optional row-offset subscript: `$k[offset]` or relative range slice `$k[start..end]`.

| Syntax | Description | SQL / Analytics Equivalent |
| :--- | :--- | :--- |
| `$k` | Field $k$ of the current row (implicit `$k[0]`). | Current value |
| `$k[-1]` | Field $k$ of the previous row. | `LAG(col, 1)` |
| `$k[+1]` | Field $k$ of the next row (auto lookahead). | `LEAD(col, 1)` |
| `$k[-m..0]` | Array/slice of field $k$ spanning from $-m$ rows ago to current row. | `ROWS BETWEEN m PRECEDING AND CURRENT ROW` |
| `$k[-m..+n]` | Window slice spanning $m$ rows past to $n$ rows ahead. | Centered rolling window |

### 2.3 Boundary Semantics (Out-of-Bounds Behavior)
When accessing rows outside the stream boundary (e.g., evaluating `$2[-1]` on `NR == 1`, or accessing lookahead past EOF):
- **Default**: Evaluates to an empty value (coerces to `0` in numeric context, `""` in string context, or `false` in conditionals).
- **Safe Option Check**: `is_null($k[offset])` or `default($k[-1], 0)` can be used to handle edge cases explicitly.

---

## 3. Window Aggregation Primitives

When a window slice is addressed (e.g., `$2[-2..0]`), built-in aggregate functions operate directly over the vector:

- `sum(slice)`: Sum of values in window.
- `avg(slice)`: Arithmetic mean.
- `min(slice)` / `max(slice)`: Min/Max values.
- `diff($k)`: Shorthand for `$k - $k[-1]`.
- `ratio($k)`: Shorthand for `$k / $k[-1]`.

---

## 4. Use Case Examples

### 4.1 Spike Detection (Diff against previous record)
Detect sudden CPU or latency spikes where current line exceeds previous line by more than 20%:
```awk
$2 - $2[-1] > 20 {
    print $1, "SPIKE:", $2[-1], "->", $2
}

```

### 4.2 3-Row Rolling Average

```awk
NR >= 3 {
    print $1, avg($2[-2..0])
}

```

### 4.3 Lookahead State Transition (Future Lookahead)

Identify when a status code shifts from `200` to `500` in the subsequent event:

```awk
$2 == 200 && $2[+1] == 500 {
    print "Warning: Transition to 500 imminent after event at", $1
}

```

---

## 5. Architectural Pipeline & Execution Model

```
                    +-----------------------------+
                    |       oawk Lexer/Parser     |
                    | (ocamllex & menhir / pure)  |
                    +--------------+--------------+
                                   |
                       Static Window Analysis
                     (Computes Min L and Max R)
                                   |
                                   v
[Input Stream] ---> [ Sliding Window Buffer (Size: L + R + 1) ] ---> [ Eval Loop ] ---> [ Stdout ]
                    (Ring-buffer / Deque in OCaml)

```

1. **Static Window Analysis Phase**:
* The parser inspects all field accesses across rules (`BEGIN`, main patterns, `END`).
* Determines:
* `L = max(0, -min_offset)` (past depth required)
* `R = max(0, max_offset)` (future lookahead required)




2. **Streaming Engine**:
* Buffer capacity is fixed at `L + R + 1`.
* The stream pre-fetches `R` lines before evaluating rule for the base line.
* For every new line pushed, the oldest line beyond `L` is evicted.
* Memory complexity is strictly **$O(L + R)$**, independent of total input size.



---

## 6. CLI Interface

```bash
oawk [OPTIONS] 'program' [file ...]
oawk [OPTIONS] -f progfile [file ...]

Options:
  -F <fs>         Set field separator (regex or string)
  -v var=val      Assign global variable
  --check         Parse and print window bounds (L, R) without running
  --version       Display version

```
