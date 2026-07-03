# Tornatech Document API — Contract (Frontend Integration)

**Status:**  Live on staging (**v1.2** — adds `tornatech_doc_table()` for multi-row config tables; fail-closed resolution). Indexed **8,199** real files (series 96% / doc-type 98% / usable 96%).
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

## Identify the product  (how to scope a request)
Pass **one** of these to scope a request to a product line:
- **`series`** = the series code directly (e.g. `GPA`) — **preferred**, always reliable.
- `product` = the product post **slug**; resolved to its `product_series` term. Only reliable if the
  slug actually maps to a series on the site (see below).
- `model` = a `product_model` term (id or slug); resolved to a series via the product that carries it
  (for the model-only archive route).

**Fail-closed guarantee (v1.1):** if `product`/`model` is supplied but cannot be resolved to a
series, every endpoint returns **empty / `url:null`** (never the whole catalog, never a wrong doc)
and includes `"unresolved": true`. Passing **no** scope at all is treated as an unscoped admin/debug
query and returns everything — so always pass a scope from the frontend.

> **Recommendation for the theme:** derive the series in-template with
> `get_the_terms($product_id, 'product_series')` (or `tornatech_series_for_model($model_id)` on the
> model archive) and pass **`series=`**. This avoids depending on product-slug↔series mapping.

Shared params: `series | product | model`, `market`, `lang`, `doc_type`, `voltage`, `hp`.
- `lang`: `en fr es de it nl pt th tr ar zh he in`. Missing-language requests fall back to EN /
  untagged (covers EN-only submittals).
- `market`: `ca` (Canada/all) · `be` (Belgium/Europe) · `ae` (Dubai/ME+Asia). Untagged files are
  treated as **universal** and returned for every market.

---

## Recommended: call the PHP helpers directly (server-side, no HTTP)
The plugin exposes these globals — use them in templates / AJAX handlers:

```php
// derive the series once, in-template (reliable, no slug dependency)
$terms  = get_the_terms($product_id, 'product_series');
$series = ($terms && !is_wp_error($terms)) ? $terms[0]->slug : '';   // e.g. "GPA"

// 1) list of docs (general + config-specific)
$docs = tornatech_documents([
    'series' => $series,            // preferred; 'product'=>$slug and 'model'=>$id also accepted
    'market' => $market,            // 'ca' | 'be' | 'ae'
    'lang'   => apply_filters('wpml_current_language', null),
]);
// => ['count'=>N, 'general'=>[...], 'filtered'=>[...]]   (empty + 'unresolved'=>true if scope can't resolve)

// 2) single best-match URL (replace ONE hard-coded ACF link) — array style takes series/model/etc.
$url = tornatech_get_doc_url(['series'=>$series, 'doc_type'=>'manual']);   // auto WPML lang; '' if none
// (string style still works for a known product slug: tornatech_get_doc_url('gpd','manual','fr'))

// 3) model-only archive: get the series from a product_model term
$series = tornatech_series_for_model($model_term_id);   // '' if it can't map

// 4) doc_type -> display label
tornatech_doc_label('brochure');   // "Brochure"

// 5) v1.2 — MULTI-ROW tables (spec / submittal / dimensional): grouped by voltage x hp
$t = tornatech_doc_table(['series'=>$series, 'doc_type'=>'submittal', 'lang'=>$lang]);
// => ['count'=>N, 'groups'=>[ ['voltage'=>'440V-480V','hp'=>'100HP-125HP','docs'=>[ ['url','lang','rev','filename'], ...]], ... ]]
foreach ($t['groups'] as $g) { /* one table row per (voltage,hp); $g['docs'] = its PDFs (lang/rev variants) */ }
// exact single cell: tornatech_get_doc_url(['series'=>$series,'doc_type'=>'submittal','voltage'=>$v,'hp'=>$hp])
```
> **For the spec/submittal/dimensional tables use `tornatech_doc_table()` (a LIST), not
> `tornatech_get_doc_url()` (a single URL).** The list/table helpers accept `voltage` + `hp` to
> filter to a specific config; the single-URL helper is only for one-of-a-kind links (manual, brochure).

### Example — replacing a Downloadables repeater
```php
// BEFORE: while ( have_rows('downloadables', ...) ) { ... get_sub_field('downloadable_link') ... }
$terms  = get_the_terms(get_the_ID(), 'product_series');
$series = ($terms && !is_wp_error($terms)) ? $terms[0]->slug : '';
$docs   = tornatech_documents(['series'=>$series, 'market'=>$market, 'lang'=>$lang]);
if ($docs['count'] === 0) { /* fall back to existing ACF rows here */ }
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


## Scope for this pass
- **In scope:** product pages + `product_model` archive (`single-product.php`, `product-detail-1.php`,
  `single_product_model.php`, `taxonomy-product_model.php`) and the two AJAX handlers. Scope each by
  deriving `series` (or `tornatech_series_for_model()` on the archive).
- **Out of scope (v1):** the **manuals** templates. The `manual` CPT is decoupled from product/series
  and has no reliable identifier to map yet — **leave those on ACF** for this pass. We'll design a
  manual→series link in the naming-convention phase.

## Rollout safety
Use **API-first with ACF fallback** during integration: if the API returns `count===0` / `url===''`
(incl. the fail-closed `unresolved` case), fall back to the existing ACF rows so no download section
ever renders empty. Retire the ACF fields once coverage is confirmed. Live site untouched until
staging sign-off.
