---
name: resolving-etoro-instruments
description: Developer playbook for building a robust instrument-resolver — symbol→ID search, batched metadata lookups with adaptive 413/414 sizing, image-variant selection, and background straggler retries. Use when writing the resolution / enrichment layer of a server-side app, populating a watchlist or portfolio UI, or bulk-enriching trade-history rows.
---

# Resolving eToro Instruments

Use this skill when displaying instrument data (logos, names, exchange) in a UI, when populating a watchlist or portfolio, or when enriching trade-history rows with human-readable instrument info.

## When to use

- Building a portfolio dashboard that shows instrument logos and display names.
- Resolving a symbol the user typed (e.g. `AAPL`) to an `instrumentId` for trading.
- Bulk-enriching many positions at once (typical: portfolio refresh).

## Step 1 — Look up `instrumentId` for unknown symbols

> The persistence-first principle (hardcode known symbols, live-resolve unknowns) is covered in the `etoro-id-resolution` rule. This step is the live-resolve path only.

```typescript
async function searchInstrument(symbol: string, ctx: EtoroRequestContext) {
  const items = await etoroFetch<{
    items: { instrumentId: number; symbolFull: string }[];
  }>(
    `/market-data/search?internalSymbolFull=${encodeURIComponent(symbol)}`,
    ctx,
  );
  // Confirm exact match — search may return MSFT, MSFT.RTH, MSFT.EUR
  const exact = items.items.find((it) => it.symbolFull === symbol);
  return exact?.instrumentId ?? items.items[0]?.instrumentId ?? null;
}
```

Note for the code: `/market-data/search` returns `instrumentId` (lowercase `d`); the metadata endpoint in Step 2 returns `instrumentID` (capital `D`). Full casing table is in the `etoro-api-conventions` rule.

## Step 2 — Batch metadata lookups (25–50 IDs per request)

`/market-data/instruments` rejects large batches with HTTP 413 / 414, so cap each call at **25 to 50** IDs. The full implementation handles the adaptive ladder when 413/414 fires:

```typescript
const BATCH_SIZE_LADDER = [50, 25] as const;

async function fetchInstrumentsResilient(
  ids: number[],
  ctx: EtoroRequestContext,
): Promise<Map<number, InstrumentMeta>> {
  const out = new Map<number, InstrumentMeta>();

  let batchSize: number = BATCH_SIZE_LADDER[0];
  let i = 0;

  while (i < ids.length) {
    const chunk = ids.slice(i, i + batchSize);
    try {
      const data = await etoroFetch<{
        instrumentDisplayDatas: InstrumentMeta[];
      }>(`/market-data/instruments?instrumentIds=${chunk.join(',')}`, ctx);
      for (const item of data.instrumentDisplayDatas) {
        out.set(item.instrumentID, item);
      }
      i += batchSize;
    } catch (err: any) {
      if (err.statusCode === 413 || err.statusCode === 414) {
        // Adaptive batching: shrink and retry the SAME chunk
        const idx = BATCH_SIZE_LADDER.indexOf(batchSize as 50 | 25);
        const next = BATCH_SIZE_LADDER[idx + 1];
        if (next) {
          batchSize = next;
          continue;
        }
        // Already at minimum — skip this batch and log
        console.warn('Instrument batch failed at minimum size:', chunk);
        i += batchSize;
      } else {
        // 429, 5xx, etc. — let outer retry/backoff layer handle it
        throw err;
      }
    }
  }

  return out;
}
```

**Critical rule:** shrink the batch **only on 413/414** (payload-size errors). For 429 (rate limit) and 5xx (transient errors), back off and retry at the same size — shrinking the batch hides the real problem behind sizes that succeed by accident.

## Step 3 — Pick the right image variant

Each instrument's `images[]` array is **unsorted** in the response — don't blindly use `images[0]`. The variants typically include:

- A vector "card" SVG that carries `backgroundColor` + `textColor` metadata (best for tile-style UI).
- Several PNG rasters at different widths (50px, 90px, 150px, …).
- Sometimes legacy formats.

Selection logic:

```typescript
function selectInstrumentImageUrl(images: EtoroInstrumentImage[]): string | null {
  if (!images?.length) return null;
  const card = images.find((i) => i.format === 'svg' && i.backgroundColor);
  if (card) return card.uri;
  const pngs = images.filter(
    (i) => i.format === 'png' && typeof i.width === 'number',
  );
  if (pngs.length) {
    return pngs.sort((a, b) => (b.width ?? 0) - (a.width ?? 0))[0].uri;
  }
  return images[0].uri;
}
```

## Step 4 — Background straggler retry for non-blocking enrichment

When a portfolio has many positions and a few metadata lookups fail (transient 5xx, partial timeout), don't make the user wait. Resolve what you can synchronously, and schedule a delayed retry for the rest:

```typescript
const STRAGGLER_DELAY_MS = 15_000;
const STRAGGLER_CACHE_TTL_MS = 60 * 60 * 1000;

void (async () => {
  await sleep(STRAGGLER_DELAY_MS);
  try {
    const stragglers = await fetchInstrumentsResilient(missingIds, ctx);
    for (const meta of stragglers.values()) {
      cache.set(meta.instrumentID, meta, STRAGGLER_CACHE_TTL_MS);
    }
  } catch (err) {
    console.warn('Straggler retry failed:', err);
  }
})();
```

The next polling cycle picks up the cached results without the user noticing the gap.

## Step 5 — Cache aggressively

Suggested TTLs to layer on top of the persistence pattern from `etoro-id-resolution`:

- **Display metadata** (`instrumentDisplayName`, `symbolFull`, `images[]`): TTL of 24 h is generous; eToro rarely changes these.
- **Live rates** (`/market-data/instruments/rates`): never cache beyond the polling interval — they're, by definition, live.

## Sanity checks

- [ ] Metadata batches are 25–50 IDs.
- [ ] 413/414 triggers shrink-and-retry; 429/5xx triggers back-off-and-retry at the same size.
- [ ] `images[0]` is never used unconditionally — variant selection runs first.
- [ ] Stragglers are retried in the background, never synchronously holding a user request.
