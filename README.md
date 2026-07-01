# Tornatech Document API — Contract (Frontend Integration)

**Status:**  Live on staging. Indexed **8,199** real files (series 96% / doc-type 98% / usable 96%).
**Plugin:** `tornatech-doc-indexer` (custom, additive, read-only).

## Base URL
- Staging: `https://tornatechstage.wpengine.com/wp-json/tornatech/v1`
- Prod (later): `https://www.tornatech.com/wp-json/tornatech/v1`

Namespace `tornatech/v1` is shared with the theme's existing `link-translation` route — our
document routes are added alongside it.

## Auth
- All endpoints are **GET, public (read-only)** — no WP auth needed to read.
- Staging sits behind WP Engine HTTP Basic Auth (`tornatechstage / designshopp`). This is
  **transparent for anything running on staging** (server-side PHP helpers, same-origin JS in
  the browser). Only *external* tools (curl/Postman) need the basic-auth header.
- (Only `POST /reindex` requires a logged-in admin; you won't call that from the frontend.)

## Identify the product
Every endpoint accepts **either**:
- `product` = the product post **slug** (e.g. `gpa`) — resolved to its `product_series` term automatically, **or**
- `series` = the series code directly (e.g. `GPA`).

Shared params: `product | series`, `market`, `lang`, `doc_type`, `voltage`, `hp`.
- `lang`: `en fr es de it nl pt th tr ar zh he in`. Missing-language requests fall back to EN /
  untagged (covers EN-only submittals).
- `market`: `ca` (Canada/all) · `be` (Belgium/Europe) · `ae` (Dubai/ME+Asia). Untagged files are
  treated as **universal** and returned for every market.

---

## Recommended: call the PHP helpers directly (server-side, no HTTP)
The plugin exposes these globals — use them in templates / AJAX handlers:

```php
// 1) list of docs (general + config-specific) for a product
$docs = tornatech_documents([
    'product' => $product_slug,     // or 'series' => 'GPA'
    'market'  => $market,           // 'ca' | 'be' | 'ae'
    'lang'    => apply_filters('wpml_current_language', null),
]);
// => ['count'=>N, 'general'=>[...], 'filtered'=>[...]]

// 2) single best-match URL (replace ONE hard-coded ACF link)
$url = tornatech_get_doc_url($product_slug, 'manual');   // auto WPML lang; '' if none

// 3) doc_type -> display label
tornatech_doc_label('brochure');   // "Brochure"
```

### Example — replacing a Downloadables repeater
```php
// BEFORE: while ( have_rows('downloadables', ...) ) { ... get_sub_field('downloadable_link') ... }
$docs = tornatech_documents(['product'=>$product_slug, 'market'=>$market, 'lang'=>$lang]);
foreach ($docs['general'] as $doc) : ?>
    <a href="<?php echo esc_url($doc['url']); ?>" class="dwnload">
        <?php echo esc_html(tornatech_doc_label($doc['doc_type'])); ?>
    </a>
<?php endforeach;
```

---

## REST endpoints (for JS / dropdown UI)

### 1. `GET /filters` — dropdown options
`/filters?product=gpa`
```json
{
  "series":   [{"value":"GPA","label":"GPA"}],
  "markets":  [],
  "voltages": [{"value":"220V-240V","label":"220V-240V"}, ...],
  "hp":       [{"value":"5HP-30HP","label":"5HP-30HP"}, ...],
  "doc_types":[{"value":"submittal","label":"submittal"}, ...]
}
```
Empty array = don't render that dropdown (e.g. GPD diesel has no voltages).
> Note: voltage/HP values are raw source granularity right now (e.g. `200V` and `200V-208V`
> may both appear). Render as-is; the naming convention will normalize these later.

### 2. `GET /documents` — list matching documents
`/documents?product=gpd&doc_type=manual&lang=en`
```json
{
  "query":   {"series":"GPD","lang":"en","doc_type":"manual"},
  "count":   4,
  "general": [ {"filename":"...","url":"...","series":"GPD","doc_type":"manual","voltage":"","hp":"","market":"","lang":"en","rev":""}, ... ],
  "filtered":[ ... ]
}
```
- `general[]` = always-show (manual, brochure, motor-connections, weight-dimensions).
- `filtered[]` = config-specific (submittal, specification, dimensional, datasheet …).

### 3. `GET /url` — single best-match URL
`/url?product=gpd&doc_type=brochure&lang=fr`
```json
{ "url":"https://.../pdf/GPDV2-3WAY-Manual-FR.pdf", "filename":"...", "lang":"fr", "rev":"", "matched":8 }
```
No match → `{ "url": null, "matched": 0 }`. Ranking: exact language first, then highest revision.

---

## Doc-type vocabulary
`submittal`, `specification`, `brochure`, `manual`, `dimensional` (3D drawing), `datasheet`,
`motor-connections`, `weight-dimensions`, `flowmeter`, `foam`, `certificate`, `step-file`, `firmware`.


## Rollout safety
Use **API-first with ACF fallback** during integration (if the API returns nothing for a product,
fall back to the existing ACF rows) so no download section ever renders empty. Retire the ACF
fields once coverage is confirmed. Live site untouched until staging sign-off.
