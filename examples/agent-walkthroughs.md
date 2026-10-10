# Agent walkthroughs

Questions an AI agent gets, and the emem calls that answer them. Every call
targets `https://emem.dev`; swap in your own node's URL when self-hosting.
Each command below is run against production by CI, so if one stops working
the build fails rather than the reader.

Cells are written out literally so each block runs on its own. In your own
code, take the `cell64` from the `locate` call before it.

## 1. "What's at 35.36°N, 138.73°E?"

```bash
# lat/lng to the one cell64 every agent resolves identically
curl -s -X POST https://emem.dev/v1/locate \
  -H 'content-type: application/json' \
  -d '{"lat":35.3606,"lng":138.7274}' | jq -r '.cell64'
# defi.zb592.nemu.zEvE

# the signed facts there, fetched and signed on a miss
curl -s -X POST https://emem.dev/v1/recall \
  -H 'content-type: application/json' \
  -d '{"cell":"defi.zb592.nemu.zEvE","bands":["copdem30m.elevation_mean"]}' | jq '.facts[0] | {value, unit, memory_token}'
```

Cite the `memory_token`, not the number alone: anyone can resolve it to the
same signed bytes and check the receipt offline.

## 2. "How high is Mount Everest?"

```bash
curl -s -X POST https://emem.dev/v1/recall \
  -H 'content-type: application/json' \
  -d '{"place":"Mount Everest","bands":["copdem30m.elevation_mean"]}' | jq '.facts[0] | {cell, value, unit}'
```

`recall` takes a `place` directly. A place name resolves to one 10 m cell, and
a name it cannot resolve confidently is refused with the reason rather
than guessed; for an area, use section 7.

## 3. "Is Everest base camp above 5,000 m?"

```bash
curl -s -X POST https://emem.dev/v1/verify \
  -H 'content-type: application/json' \
  -d '{"cell":"defi.zb53e.wUdA.rIhe","claim":{"band":"copdem30m.elevation_mean","op":"gt","value":5000.0,"tslot":0}}' | jq '{verdict, evidence}'
```

`verdict` is `true`, `false` or `unknown`, with the signed evidence it rested
on. `unknown` means no fact was found, which is not the same as `false`.

## 4. "How does Madrid differ from Lisbon?"

```bash
curl -s -X POST https://emem.dev/v1/compare \
  -H 'content-type: application/json' \
  -d '{"a":"defi.zb5cb.zda5f.nEqI","b":"defi.zb5b8.qIgO.pIho"}' | jq '{shared_bands, per_band}'
```

`compare` works over the bands both cells already hold, and says which bands
only one side has (`only_a`, `only_b`). Recall the bands you care about at both
cells first if `shared_bands` comes back short.

## 5. "Find places like my farm"

```bash
curl -s -X POST https://emem.dev/v1/find_similar \
  -H 'content-type: application/json' \
  -d '{"key":"defi.zb5d8.hAgU.pIxO","k":5}' | jq '.neighbors[] | {cell, band_used}'
```

Each neighbour names the band the similarity was computed on. Read
`interpretation` before you say two places are alike.

## 6. "How has this plot changed over the last two seasons?"

```bash
curl -s -X POST https://emem.dev/v1/field_series \
  -H 'content-type: application/json' \
  -d '{"geometry":{"type":"Polygon","coordinates":[[[-5.3012,6.8501],[-5.2988,6.8501],[-5.2988,6.8524],[-5.3012,6.8524],[-5.3012,6.8501]]]},"index":"ndvi","start_date":"2025-01-01","end_date":"2026-09-30"}' | jq '{rows: (.rows | length), anomaly, cautions}'
```

One signed series over the area: per-scene distributions, an anomaly against
the area's own history, and a change map. `emem_intent` with
`type: "area_over_time"` routes here from a plain question.

## 7. "What's the average across these places?"

```bash
curl -s -X POST https://emem.dev/v1/query_region \
  -H 'content-type: application/json' \
  -d '{"geometry":"cells:defi.zb5cb.zda5f.nEqI,defi.zb5b8.qIgO.pIho","bands":["copdem30m.elevation_mean"],"agg":"mean"}' | jq '.aggregates'
```

For a named area rather than a list of cells, `POST /v1/recall_polygon` and
`POST /v1/grid` sample the area and aggregate in one call.

## 8. "I don't know what to call"

```bash
curl -s -X POST https://emem.dev/v1/ask \
  -H 'content-type: application/json' \
  -d '{"q":"how is the vegetation doing around Manaus?"}' | jq '{answer, receipt: .receipt.fact_cids}'
```

`ask` routes a plain-language question and returns a signed answer with the
facts it read. Over MCP, `emem_tools` searches the full catalogue when the
tool you need is not in your list.

## When to call what

| The question | First call | Then |
|---|---|---|
| a lat/lng or a place name | `/v1/locate` | `/v1/recall` |
| "is X true here?" | `/v1/verify` | |
| "how does X differ from Y?" | `/v1/recall` at both | `/v1/compare` |
| "find places like X" | `/v1/find_similar` | `/v1/recall` at a neighbour |
| "how has this area changed?" | `/v1/field_series` | |
| "what changed between two dates at one cell?" | `/v1/intent` with `did_change` | `/v1/diff` |
| "the average over these places" | `/v1/query_region` | |
| "I don't know" | `/v1/ask` | |
| "check what someone handed me" | `/v1/memory_token/resolve` | `/v1/verify_receipt` |

## In your reply

1. State the fact the user asked about.
2. Cite its `emem:fact:` token, or one `emem:bundle:` line for several, so the
   reader can check it without trusting you.
3. Say what the check does not prove: a signature says who signed, not that
   the value is right.
