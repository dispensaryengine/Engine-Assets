# Jane (iHeartJane) \+ Flowhub Dispensary Scraping Guide

## Developer Reference — Buffalo-Area Dispensaries

---

## Table of Contents

1. [Overview & Architecture](#1.-overview-&-architecture)  
2. [Platform Stack Explained](#2.-platform-stack-explained)  
3. [Target Dispensaries](#3.-target-dispensaries)  
4. [Algolia API Deep Dive](#4.-algolia-api-deep-dive)  
5. [Credential Discovery](#5.-credential-discovery)  
6. [Request Construction](#6.-request-construction)  
7. [Response Structure & All Fields](#7.-response-structure-&-all-fields)  
8. [Flowhub POS Integration](#8.-flowhub-pos-integration)  
9. [Pricing Model](#9.-pricing-model)  
10. [Photos & Media](#10.-photos-&-media)  
11. [Specials & Promotions](#11.-specials-&-promotions)  
12. [Pagination Strategy](#12.-pagination-strategy)  
13. [Full Working Scraper (Python)](#13.-full-working-scraper-\(python\))  
14. [Empty Product Schema (JSON)](#14.-empty-product-schema-\(json\))  
15. [Anti-Bot / TLS Considerations](#15.-anti-bot-/-tls-considerations)  
16. [Field Reference Table](#16.-field-reference-table)  
17. [Common Errors & Troubleshooting](#17.-common-errors-&-troubleshooting)

---

## 1\. Overview & Architecture {#1.-overview-&-architecture}

Jane dispensaries (branded as **iHeartJane**) embed a menu widget on their own websites that fetches product data from **Algolia** — a hosted search/indexing service. The Algolia index name is:

menu-products-production

hosted at:

https://search.iheartjane.com

Every dispensary on the Jane platform is assigned a numeric `store_id`. When a user visits the dispensary's menu page, the site ships Algolia credentials (App ID \+ API Key) directly in its HTML. The embedded widget then queries the Algolia index filtered to that store's products.

**This means scraping is straightforward:**

1. Fetch the dispensary HTML page → extract App ID \+ API Key \+ Store ID  
2. POST to the Algolia query endpoint with `filters: "store_id:<id>"`  
3. Paginate through results (up to 1,000 hits per page)  
4. Parse each "hit" record

No login, no session tokens, no OAuth. The credentials are public-facing by design.

---

## 2\. Platform Stack Explained {#2.-platform-stack-explained}

| Layer | Technology | Purpose |
| :---- | :---- | :---- |
| Menu Platform | iHeartJane (Jane) | Consumer-facing menu & ordering |
| Search/Index | Algolia (`menu-products-production`) | Product search, filtering, pagination |
| POS System | **Flowhub** | Point-of-sale; provides product UUIDs |
| CDN / Anti-bot | Cloudflare | TLS fingerprinting; use `curl_cffi` to bypass |
| Media CDN | `product-assets.iheartjane.com` (Cloudflare Images) | Product photos at multiple resolutions |

### How Flowhub Connects

Each Algolia product record contains a `pos_product_lookup` field — a dictionary mapping each available weight/unit to a **Flowhub UUID**:

"pos\_product\_lookup": {

  "gram": "33c60720-07ac-4d49-bcbc-a0f21508b99b"

}

These UUIDs are Flowhub's internal product IDs. Jane reads live inventory counts and pricing from Flowhub via a background sync; what you see in Algolia is the already-resolved snapshot. If you need to cross-reference products against a Flowhub API directly, these UUIDs are the join key.

---

## 3\. Target Dispensaries {#3.-target-dispensaries}

Both dispensaries below use **Jane (iHeartJane)** as their menu platform and **Flowhub** as their POS.

| Dispensary | Store ID | Address | Phone | Website | Menu URL |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Buffalo Dreams | **5876** | 900 Niagara Falls Blvd, Buffalo, NY 14223 | (716) 466-2351 | [https://shopbuffalodreams.com/](https://shopbuffalodreams.com/) | [https://shopbuffalodreams.com/shop/](https://shopbuffalodreams.com/shop/) |
| Stonedhouse NY | **6928** | 755 Center St Ste. 7, Lewiston, NY 14092 | (716) 306-3500 | [https://stonedhouseny.com/](https://stonedhouseny.com/) | [https://stonedhouseny.com/order-online/menu/](https://stonedhouseny.com/order-online/menu/) |

**Live product counts (verified 2026-06-23):**

- Buffalo Dreams: **872 products** across 291 Algolia pages  
- Stonedhouse NY: **300 products** across 100 Algolia pages

>   
> **Note:** Each Algolia "page" is a result of the default 3 hits/page. When you set `hitsPerPage: 1000`, you typically get all products in 1–2 requests.

---

## 4\. Algolia API Deep Dive {#4.-algolia-api-deep-dive}

### Endpoint

POST https://search.iheartjane.com/1/indexes/menu-products-production/query

**Query string required:**

?x-algolia-agent=Algolia%20for%20JavaScript%20(4.26.0)%3B%20Browser

Full URL:

https://search.iheartjane.com/1/indexes/menu-products-production/query?x-algolia-agent=Algolia%20for%20JavaScript%20(4.26.0)%3B%20Browser

### Required Headers

X-Algolia-Application-Id: VFM4X0N23A

X-Algolia-API-Key: edc5435c65d771cecbd98bbd488aa8d3

Content-Type: application/json

Referer: https://www.iheartjane.com/

Origin: https://www.iheartjane.com

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36

> **Important:** The `Referer` and `Origin` headers pointing to `iheartjane.com` are required. Requests without them may be rejected.

### Credentials

The credentials shown above are **shared across all Jane-platform dispensaries** (they are the Algolia search-only/public API credentials for iHeartJane's platform). They can also be extracted dynamically from any dispensary's menu HTML or from the iHeartJane store page (`https://www.iheartjane.com/stores/<store_id>`).

**Dynamic extraction regex patterns:**

import re

app\_id\_pattern  \= r'algoliaAppId\["\\s:\]+"(\[A-Z0-9\]{8,})"'

api\_key\_pattern \= r'algoliaApiKey\["\\s:\]+"(\[a-f0-9\]{30,})"'

store\_id\_from\_html \= r'store\[\_\\-\]?id\["\\s:\]+\["\\'\\'\]?(\\d+)\["\\'\\'\]?'

---

## 5\. Credential Discovery {#5.-credential-discovery}

There are **three reliable sources** for Algolia credentials:

### Method A — Dispensary Menu Page (preferred for store\_id)

import re

from curl\_cffi import requests

resp \= requests.get("https://shopbuffalodreams.com/shop/", impersonate="chrome")

html \= resp.text

app\_id  \= re.search(r'algoliaAppId\["\\s:\]+"(\[A-Z0-9\]{8,})"', html).group(1)

api\_key \= re.search(r'algoliaApiKey\["\\s:\]+"(\[a-f0-9\]{30,})"', html).group(1)

store\_id \= re.search(r'store\[\_-\]?id\["\\s:\]+\["\\'\\'\]?(\\d+)\["\\'\\'\]?', html, re.I).group(1)

### Method B — iHeartJane Store Page (fallback)

\# Works even when the dispensary site doesn't embed credentials in HTML

resp \= requests.get(f"https://www.iheartjane.com/stores/{store\_id}", impersonate="chrome")

html \= resp.text

\# Same regex patterns apply

### Method C — Hardcoded Fallback (static, may rotate)

CREDENTIALS \= {

    "app\_id":  "VFM4X0N23A",

    "api\_key": "edc5435c65d771cecbd98bbd488aa8d3",

}

> ⚠️ **Note:** Hardcoded credentials may rotate. Always prefer dynamic extraction. If the page doesn't embed them, fall back to Method B before Method C.

---

## 6\. Request Construction {#6.-request-construction}

### Minimal Working Request Body

{

  "query": "",

  "filters": "store\_id:5876",

  "hitsPerPage": 1000,

  "page": 0

}

### Full Request Body with Optional Parameters

{

  "query": "",

  "filters": "store\_id:5876",

  "hitsPerPage": 1000,

  "page": 0,

  "attributesToRetrieve": \["\*"\],

  "attributesToHighlight": \[\],

  "facets": \["kind", "brand", "effects", "flavors", "terpenes"\],

  "numericFilters": \[\],

  "tagFilters": \[\]

}

### Filtering by Category

You can filter to a specific product category using compound filters:

{

  "filters": "store\_id:5876 AND kind:flower"

}

Available `kind` values: `flower`, `extract`, `edible`, `tincture`, `topical`, `gear`, `pre_roll`, `vaporizer`, `capsule`, `clone`

### Search Within a Store

{

  "query": "OG Kush",

  "filters": "store\_id:5876",

  "hitsPerPage": 20,

  "page": 0

}

---

## 7\. Response Structure & All Fields {#7.-response-structure-&-all-fields}

### Top-Level Response

{

  "hits": \[...\],

  "nbHits": 872,

  "page": 0,

  "nbPages": 1,

  "hitsPerPage": 1000,

  "exhaustiveNbHits": true,

  "query": "",

  "params": "...",

  "processingTimeMS": 5

}

| Field | Type | Description |
| :---- | :---- | :---- |
| `hits` | array | Array of product records |
| `nbHits` | integer | Total matching products |
| `page` | integer | Current page (0-indexed) |
| `nbPages` | integer | Total pages at current hitsPerPage |
| `hitsPerPage` | integer | Products per page requested |
| `exhaustiveNbHits` | boolean | Whether count is exact |

### Product Hit — Complete Field Reference

Each element in `hits` is a product record with the following fields:

#### Identifiers

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `objectID` | string | `"prod_2698533_store_5876"` | Algolia object ID |
| `product_id` | integer | `2698533` | Jane platform product ID |
| `url_slug` | string | `"hash-angie"` | URL-safe product name |
| `unique_slug` | string | `"hash-angie-extract-budder"` | Unique slug with category |
| `searchable_slug` | string | `"hash-angie-extract"` | Search-optimized slug |
| `pos_product_lookup` | object | `{"gram": "33c60720-..."}` | **Flowhub UUID(s) per weight** |
| `product_brand_id` | integer | `1234` | Jane brand entity ID |
| `store_specific_product` | boolean | `false` | Store-exclusive product flag |

#### Product Info

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `name` | string | `"Angie"` | Product name |
| `brand` | string | `"#HASH"` | Brand name |
| `kind` | string | `"extract"` | Top-level category |
| `kind_subtype` | string | `"Budder"` | Sub-category |
| `root_subtype` | string | `"Waxes"` | Root classification |
| `custom_product_type` | string | `"extract"` | Custom type override |
| `custom_product_subtype` | string | `"Waxes"` | Custom subtype override |
| `category` | string | `"extract"` | Alias for `kind` |
| `type` | string | `"extract"` | Alias field |
| `root_types` | array | `["extract"]` | Array of root type(s) |
| `description` | string | `"A smooth..."` | Full product description |
| `strain` | string | `"OG Kush"` or `null` | Strain name |
| `store_notes` | string | `"Limonene: 0.42..."` | Dispensary-added notes |
| `brand_subtype` | string | `null` | Brand-level subtype |
| `collections` | array | `[]` | Product collections/tags |
| `recommendation` | object | `null` | Personalized recommendation data |

#### Potency / Lab

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `percent_thc` | float | `72.38` | THC % (may be null) |
| `percent_thca` | float | `73.499` | THCA % (may be null) |
| `percent_cbd` | float | `null` | CBD % (may be null) |
| `percent_cbda` | float | `null` | CBDA % (may be null) |
| `percent_tac` | float | `null` | Total Active Cannabinoids % |
| `has_thc_potency` | boolean | `true` | Has THC data flag |
| `product_percent_thc` | float | `null` | Product-level THC (not batch) |
| `product_percent_cbd` | float | `null` | Product-level CBD (not batch) |
| `inventory_potencies` | array | see below | Per-weight potency breakdown |
| `cannabinoids` | array | `[]` | Additional cannabinoid data |
| `lab_results` | array | `[]` | COA/lab result objects |
| `lab_result_urls` | array | `[]` | Direct URLs to COA PDFs |
| `compound_names` | array | `[]` | Chemical compound names |

`inventory_potencies` element structure:

{

  "price\_id": "gram",

  "thc\_potency": 72.38,

  "thca\_potency": 73.5,

  "cbd\_potency": 0.0,

  "tac\_potency": 0.0

}

#### Characteristics

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `effects` | array | `["Relaxed", "Happy"]` | Effect tags |
| `flavors` | array | `["Citrus", "Pine"]` | Flavor tags |
| `terpenes` | array | `["Limonene"]` | Terpene names |
| `feelings` | array | `["Calm"]` | Feeling tags |
| `activities` | array | `["Sleeping"]` | Activity tags |
| `allergens` | array | `[]` | Allergen warnings |
| `ingredients` | array | `[]` | Ingredient list (edibles) |

#### Pricing

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `price_gram` | float | `30.00` | Regular price per gram |
| `price_half_gram` | float | `null` | Regular price per 0.5g |
| `price_two_gram` | float | `null` | Regular price per 2g |
| `price_eighth_ounce` | float | `45.00` | Regular price per 1/8 oz |
| `price_quarter_ounce` | float | `null` | Regular price per 1/4 oz |
| `price_half_ounce` | float | `null` | Regular price per 1/2 oz |
| `price_ounce` | float | `null` | Regular price per oz |
| `price_each` | float | `null` | Regular price per unit |
| `discounted_price_gram` | float | `null` | Discounted price per gram |
| `discounted_price_eighth_ounce` | float | `null` | Discounted price per 1/8oz |
| *(same pattern for all weights)* |  |  |  |
| `special_price_gram` | float | `null` | Special/sale price per gram |
| `special_price_eighth_ounce` | float | `null` | Special price per 1/8oz |
| *(same pattern for all weights)* |  |  |  |
| `max_cart_quantity_gram` | integer | `13` | Max purchasable per cart |
| *(same pattern for all weights)* |  |  |  |
| `bucket_price` | float | `30.0` | Lowest available price |
| `sort_price` | float | `30.0` | Price used for sorting |
| `available_weights` | array | `["gram"]` | Which weights are available |

**All price weight suffixes:** `gram`, `half_gram`, `two_gram`, `eighth_ounce`, `quarter_ounce`, `half_ounce`, `ounce`, `each`

#### Specials / Promotions

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `special_id` | integer | `null` | Active special ID |
| `special_title` | string | `null` | Special name |
| `special_amount` | float | `null` | Discount amount |
| `special_custom_badge` | string | `null` | Custom badge text |
| `applicable_special_ids` | array | `null` | All applicable special IDs |
| `applicable_special_types` | array | `["bundle"]` | Types: bundle, cart\_total, etc. |
| `applicable_bulk_special_ids` | array | `null` | Bulk deal IDs |
| `applicable_cart_total_special_ids` | array | `null` | Cart total deal IDs |
| `applicable_bundle_special_ids` | array | `[]` | Bundle deal IDs |
| `applicable_brand_special_ids` | array | `null` | Brand-level deal IDs |
| `applicable_bundle_brand_special_ids` | array | `[]` | Bundle brand deal IDs |
| `applicable_brand_specials_included_user_segments` | array | `[]` | User segment inclusions |
| `applicable_brand_specials_excluded_user_segments` | array | `[]` | User segment exclusions |
| `has_brand_discount` | boolean | `false` | Brand discount flag |
| `brand_special_prices` | object | `{}` | Brand-level special price map |

#### Quantity / Packaging

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `dosage` | string | `null` | Dosage description |
| `amount` | string | `null` | Amount description |
| `quantity_value` | float | `null` | Numeric quantity |
| `quantity_units` | string | `null` | Unit of quantity |
| `net_weight_grams` | float | `0.0` | Net weight in grams |
| `max_cart_quantity` | integer | `13` | Global max cart quantity |
| `allow_multiple_flower_count` | boolean | `false` | Multi-flower count flag |

#### Availability

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `available_for_delivery` | boolean | `false` | Delivery availability |
| `available_for_pickup` | boolean | `true` | Pickup availability |
| `store_types` | array | `["recreational"]` | Store type(s) |

#### Media

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `image_urls` | array | `["https://..."]` | Flat image URL list (legacy) |
| `product_photos` | array | see below | Structured photo objects |
| `photos` | array | see below | Alias for product\_photos |
| `has_photos` | boolean | `true` | Has photos flag |
| `photo_weights` | array | `[]` | Per-weight photo mappings |
| `brand_logo_url` | string | `"https://..."` | Brand logo URL |

`product_photos` / `photos` element structure:

{

  "id": "https://product-assets.iheartjane.com/photos/03/a7/03a7...jpeg",

  "position": 0,

  "urls": {

    "small":      "https://product-assets.iheartjane.com/cdn-cgi/image/width=174,...",

    "medium":     "https://product-assets.iheartjane.com/cdn-cgi/image/width=327,...",

    "original":   "https://product-assets.iheartjane.com/photos/03/a7/...",

    "extraLarge": "https://product-assets.iheartjane.com/cdn-cgi/image/width=1000,..."

  }

}

#### Ratings & Reviews

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `aggregate_rating` | float | `4.5` | Average star rating (0.0 if none) |
| `review_count` | integer | `12` | Number of reviews |

#### Rankings

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `best_seller_rank` | integer | `null` | Best-seller rank |
| `operator_store_rank` | integer | `null` | Operator-assigned rank |

#### Metadata

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `indexed_at` | string | `"2026-06-22T14:30:00Z"` | Last Algolia index time |
| `version` | integer | `3` | Record version number |
| `search_text` | string | `"Angie extract..."` | Concatenated search text |
| `_geoloc` | object | `{"lat": 42.97, "lng": -78.82}` | Store geolocation |
| `_highlightResult` | object | — | Algolia highlight metadata |

#### Business / Compliance

| Field | Type | Example | Notes |
| :---- | :---- | :---- | :---- |
| `business_licenses` | array | `[]` | License records |
| `roots_custom_rows` | array | `[]` | Custom compliance rows |

---

## 8\. Flowhub POS Integration {#8.-flowhub-pos-integration}

Flowhub is the point-of-sale system used by both dispensaries. It manages real-time inventory, batch tracking, and compliance. The connection between Jane and Flowhub manifests in one key field:

### `pos_product_lookup`

{

  "pos\_product\_lookup": {

    "gram":        "33c60720-07ac-4d49-bcbc-a0f21508b99b",

    "eighth\_ounce": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"

  }

}

- Keys are the weight/unit names (same as price field suffixes)  
- Values are **Flowhub product UUID v4** identifiers  
- Each weight variant may have a different Flowhub UUID (separate inventory items in Flowhub)  
- If a product only has one weight, only one key appears

### Use Cases for Flowhub UUIDs

1. **Cross-reference** Jane product data against Flowhub API responses  
2. **Inventory sync** — use UUIDs to subscribe to Flowhub webhooks for stock updates  
3. **Compliance lookup** — Flowhub stores METRC batch IDs; UUIDs are the join key  
4. **Cart submission** — if building a custom ordering interface, Flowhub UUIDs identify items

### How Jane ↔ Flowhub Sync Works

1. Flowhub pushes inventory updates → Jane's ingestion pipeline  
2. Jane normalizes the data and writes to Algolia index  
3. `indexed_at` timestamp reflects last successful sync  
4. Price changes in Flowhub propagate to Algolia within minutes

---

## 9\. Pricing Model {#9.-pricing-model}

Jane supports a three-tier pricing model per weight:

Regular Price  →  discounted\_price  →  special\_price

- **Regular price** (`price_<weight>`): standard retail price  
- **Discounted price** (`discounted_price_<weight>`): always-on member/loyalty discount  
- **Special price** (`special_price_<weight>`): time-limited promotion

**Determining the effective price:**

def effective\_price(hit, weight):

    special   \= hit.get(f"special\_price\_{weight}")

    discounted \= hit.get(f"discounted\_price\_{weight}")

    regular   \= hit.get(f"price\_{weight}")

    

    if special is not None:

        return special, "special"

    if discounted is not None:

        return discounted, "discounted"

    if regular is not None:

        return regular, "regular"

    return None, None

**`bucket_price`** is always the lowest available price across all weights/tiers — useful for sorting and display cards.

---

## 10\. Photos & Media {#10.-photos-&-media}

### Photo URL Pattern

All photos are served via Cloudflare Images on `product-assets.iheartjane.com`:

\# Original (full resolution)

https://product-assets.iheartjane.com/photos/{aa}/{bb}/{uuid}.jpeg

\# Sized variants (via Cloudflare Image Resizing)

https://product-assets.iheartjane.com/cdn-cgi/image/width=174,fit=scale-down,format=auto,metadata=none/photos/{aa}/{bb}/{uuid}.jpeg

**Available sizes:** | Size | Width | Field | |------|-------|-------| | small | 174px | `urls.small` | | medium | 327px | `urls.medium` | | extraLarge | 1000px | `urls.extraLarge` | | original | native | `urls.original` |

### Extraction Priority

1. Use `product_photos` if present (structured, includes position)  
2. Fall back to `photos` (same structure, different field name)  
3. Final fallback: `image_urls` (flat string array, no sizing variants)

---

## 11\. Specials & Promotions {#11.-specials-&-promotions}

Jane supports multiple special types that can apply to a product:

| Type | Field | Description |
| :---- | :---- | :---- |
| Direct special | `special_id`, `special_title` | Single active special |
| Bundle | `applicable_bundle_special_ids` | Buy X get Y deals |
| Cart total | `applicable_cart_total_special_ids` | Spend $X save $Y |
| Brand | `applicable_brand_special_ids` | Brand-wide discounts |
| Bulk | `applicable_bulk_special_ids` | Bulk purchase discounts |

To check if a product is on sale:

def is\_on\_sale(hit):

    for weight in \["gram", "half\_gram", "two\_gram", "eighth\_ounce",

                   "quarter\_ounce", "half\_ounce", "ounce", "each"\]:

        if hit.get(f"special\_price\_{weight}") is not None:

            return True

        if hit.get(f"discounted\_price\_{weight}") is not None:

            return True

    return bool(hit.get("special\_id"))

---

## 12\. Pagination Strategy {#12.-pagination-strategy}

Algolia uses **0-indexed pages**. With `hitsPerPage: 1000` (the maximum), most stores fit in 1–2 requests.

def fetch\_all\_products(store\_id, credentials):

    all\_hits \= \[\]

    page \= 0

    

    while True:

        response \= algolia\_query(store\_id, credentials, page=page, hits\_per\_page=1000)

        hits \= response.get("hits", \[\])

        all\_hits.extend(hits)

        

        nb\_pages \= response.get("nbPages", 1\)

        print(f"Page {page+1}/{nb\_pages}: got {len(hits)} (total: {len(all\_hits)})")

        

        if page \>= nb\_pages \- 1:

            break

        page \+= 1

    

    return all\_hits

**Recommended settings:**

- `hitsPerPage: 1000` — maximum allowed, minimizes requests  
- No `query` (empty string) — returns all products  
- Only `filters: "store_id:<id>"` — no other filters unless intentional

---

## 13\. Full Working Scraper (Python) {#13.-full-working-scraper-(python)}

### Dependencies

pip install curl\_cffi

### Complete Script

\#\!/usr/bin/env python3

"""

Jane (iHeartJane) \+ Flowhub Dispensary Scraper

Targets: Buffalo Dreams (store\_id=5876), Stonedhouse NY (store\_id=6928)

"""

import json

import re

from datetime import datetime, timezone

from pathlib import Path

from curl\_cffi import requests as cffi\_requests

\# ── Configuration ──────────────────────────────────────────────────────────────

DISPENSARIES \= {

    "buffalo\_dreams": {

        "name":           "Buffalo Dreams",

        "store\_id":       5876,

        "store\_url":      "https://shopbuffalodreams.com/shop/",

        "jane\_store\_url": "https://www.iheartjane.com/stores/5876/buffalo-dreams/menu",

    },

    "stonedhouse\_ny": {

        "name":           "Stonedhouse NY",

        "store\_id":       6928,

        "store\_url":      "https://stonedhouseny.com/order-online/menu/",

        "jane\_store\_url": "https://www.iheartjane.com/stores/6928",

    },

}

ALGOLIA\_INDEX    \= "menu-products-production"

ALGOLIA\_BASE\_URL \= "https://search.iheartjane.com"

ALGOLIA\_ENDPOINT \= (

    f"{ALGOLIA\_BASE\_URL}/1/indexes/{ALGOLIA\_INDEX}/query"

    "?x-algolia-agent=Algolia%20for%20JavaScript%20(4.26.0)%3B%20Browser"

)

HITS\_PER\_PAGE    \= 1000

FALLBACK\_CREDENTIALS \= {

    "app\_id":  "VFM4X0N23A",

    "api\_key": "edc5435c65d771cecbd98bbd488aa8d3",

}

WEIGHTS \= \["gram", "half\_gram", "two\_gram", "eighth\_ounce",

           "quarter\_ounce", "half\_ounce", "ounce", "each"\]

\# ── Credential Discovery ───────────────────────────────────────────────────────

def get\_credentials(store\_url: str, store\_id: int) \-\> dict:

    for url in \[store\_url, f"https://www.iheartjane.com/stores/{store\_id}"\]:

        try:

            r \= cffi\_requests.get(url, impersonate="chrome", timeout=15)

            app\_id  \= re.search(r'algoliaAppId\["\\s:\]+"(\[A-Z0-9\]{8,})"', r.text)

            api\_key \= re.search(r'algoliaApiKey\["\\s:\]+"(\[a-f0-9\]{30,})"', r.text)

            if app\_id and api\_key:

                return {"app\_id": app\_id.group(1), "api\_key": api\_key.group(1)}

        except Exception as e:

            print(f"  Warning: credential fetch failed for {url}: {e}")

    print("  Using hardcoded fallback credentials.")

    return FALLBACK\_CREDENTIALS

\# ── Algolia Query ──────────────────────────────────────────────────────────────

def query\_algolia(store\_id: int, creds: dict, page: int \= 0\) \-\> dict:

    headers \= {

        "X-Algolia-Application-Id": creds\["app\_id"\],

        "X-Algolia-API-Key":        creds\["api\_key"\],

        "Content-Type":             "application/json",

        "Referer":                  "https://www.iheartjane.com/",

        "Origin":                   "https://www.iheartjane.com",

        "User-Agent":               "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",

    }

    payload \= {

        "query":        "",

        "filters":      f"store\_id:{store\_id}",

        "hitsPerPage":  HITS\_PER\_PAGE,

        "page":         page,

    }

    resp \= cffi\_requests.post(

        ALGOLIA\_ENDPOINT, headers=headers, json=payload,

        impersonate="chrome", timeout=30

    )

    resp.raise\_for\_status()

    return resp.json()

\# ── Fetch All Products ─────────────────────────────────────────────────────────

def fetch\_all\_products(store\_id: int, creds: dict) \-\> list:

    all\_hits, page \= \[\], 0

    while True:

        data       \= query\_algolia(store\_id, creds, page=page)

        hits       \= data.get("hits", \[\])

        nb\_pages   \= data.get("nbPages", 1\)

        all\_hits.extend(hits)

        print(f"  Page {page+1}/{nb\_pages}: {len(hits)} hits (running: {len(all\_hits)})")

        if page \>= nb\_pages \- 1:

            break

        page \+= 1

    return all\_hits

\# ── Product Extraction ─────────────────────────────────────────────────────────

def extract\_prices(hit: dict) \-\> dict:

    prices \= {}

    for w in WEIGHTS:

        regular \= hit.get(f"price\_{w}")

        if regular is not None:

            prices\[w\] \= {

                "regular":           regular,

                "discounted":        hit.get(f"discounted\_price\_{w}"),

                "special":           hit.get(f"special\_price\_{w}"),

                "max\_cart\_quantity": hit.get(f"max\_cart\_quantity\_{w}"),

            }

    return prices

def extract\_photos(hit: dict) \-\> list:

    photos \= \[\]

    for photo in (hit.get("product\_photos") or hit.get("photos") or \[\]):

        urls \= photo.get("urls", {})

        photos.append({

            "position":   photo.get("position", 0),

            "small":      urls.get("small"),

            "medium":     urls.get("medium"),

            "original":   urls.get("original"),

            "extraLarge": urls.get("extraLarge"),

        })

    if not photos:

        photos \= \[{"original": u} for u in (hit.get("image\_urls") or \[\])\]

    return photos

def extract\_product(hit: dict) \-\> dict:

    return {

        \# Identifiers

        "id":                    hit.get("product\_id"),

        "objectID":              hit.get("objectID"),

        "url\_slug":              hit.get("url\_slug"),

        "unique\_slug":           hit.get("unique\_slug"),

        "searchable\_slug":       hit.get("searchable\_slug"),

        "pos\_product\_lookup":    hit.get("pos\_product\_lookup"),    \# Flowhub UUID(s)

        "product\_brand\_id":      hit.get("product\_brand\_id"),

        "store\_specific\_product": hit.get("store\_specific\_product"),

        \# Core info

        "name":                  hit.get("name"),

        "brand":                 hit.get("brand"),

        "category":              hit.get("kind"),

        "subcategory":           hit.get("kind\_subtype") or hit.get("root\_subtype"),

        "custom\_product\_type":   hit.get("custom\_product\_type"),

        "custom\_product\_subtype": hit.get("custom\_product\_subtype"),

        "description":           hit.get("description"),

        "strain":                hit.get("strain"),

        "store\_notes":           hit.get("store\_notes"),

        "collections":           hit.get("collections", \[\]),

        \# Potency

        "percent\_thc":           hit.get("percent\_thc"),

        "percent\_thca":          hit.get("percent\_thca"),

        "percent\_cbd":           hit.get("percent\_cbd"),

        "percent\_cbda":          hit.get("percent\_cbda"),

        "percent\_tac":           hit.get("percent\_tac"),

        "has\_thc\_potency":       hit.get("has\_thc\_potency"),

        "inventory\_potencies":   hit.get("inventory\_potencies", \[\]),

        "cannabinoids":          hit.get("cannabinoids", \[\]),

        "lab\_results":           hit.get("lab\_results", \[\]),

        "lab\_result\_urls":       hit.get("lab\_result\_urls", \[\]),

        \# Characteristics

        "effects":               hit.get("effects", \[\]),

        "flavors":               hit.get("flavors", \[\]),

        "terpenes":              hit.get("terpenes", \[\]),

        "feelings":              hit.get("feelings", \[\]),

        "activities":            hit.get("activities", \[\]),

        "allergens":             hit.get("allergens", \[\]),

        "ingredients":           hit.get("ingredients", \[\]),

        \# Pricing

        "prices":                extract\_prices(hit),

        "available\_weights":     hit.get("available\_weights", \[\]),

        "bucket\_price":          hit.get("bucket\_price"),

        "sort\_price":            hit.get("sort\_price"),

        \# Specials

        "special\_id":            hit.get("special\_id"),

        "special\_title":         hit.get("special\_title"),

        "special\_amount":        hit.get("special\_amount"),

        "has\_brand\_discount":    hit.get("has\_brand\_discount"),

        \# Quantity

        "dosage":                hit.get("dosage"),

        "amount":                hit.get("amount"),

        "quantity\_value":        hit.get("quantity\_value"),

        "quantity\_units":        hit.get("quantity\_units"),

        "net\_weight\_grams":      hit.get("net\_weight\_grams"),

        "max\_cart\_quantity":     hit.get("max\_cart\_quantity"),

        \# Availability

        "available\_for\_delivery": hit.get("available\_for\_delivery"),

        "available\_for\_pickup":  hit.get("available\_for\_pickup"),

        "store\_types":           hit.get("store\_types", \[\]),

        \# Media

        "photos":                extract\_photos(hit),

        "brand\_logo\_url":        hit.get("brand\_logo\_url"),

        \# Ratings

        "aggregate\_rating":      hit.get("aggregate\_rating"),

        "review\_count":          hit.get("review\_count"),

        \# Meta

        "indexed\_at":            hit.get("indexed\_at"),

        "version":               hit.get("version"),

    }

\# ── Main ───────────────────────────────────────────────────────────────────────

def scrape(key: str, config: dict, output\_dir: Path):

    print(f"\\nScraping: {config\['name'\]} (store\_id={config\['store\_id'\]})")

    creds    \= get\_credentials(config\["store\_url"\], config\["store\_id"\])

    raw\_hits \= fetch\_all\_products(config\["store\_id"\], creds)

    

    by\_category \= {}

    for hit in raw\_hits:

        product  \= extract\_product(hit)

        category \= product\["category"\] or "unknown"

        by\_category.setdefault(category, \[\]).append(product)

    

    output \= {

        "dispensary":     config\["name"\],

        "store\_id":       config\["store\_id"\],

        "store\_url":      config\["store\_url"\],

        "jane\_store\_url": config\["jane\_store\_url"\],

        "total\_products": len(raw\_hits),

        "categories":     {cat: len(items) for cat, items in by\_category.items()},

        "scraped\_at":     datetime.now(timezone.utc).isoformat(),

        "inventory":      by\_category,

    }

    

    path \= output\_dir / f"{key}\_inventory.json"

    path.write\_text(json.dumps(output, indent=2, ensure\_ascii=False))

    print(f"  Saved {len(raw\_hits)} products → {path}")

    return output

if \_\_name\_\_ \== "\_\_main\_\_":

    out \= Path("output")

    out.mkdir(exist\_ok=True)

    for key, config in DISPENSARIES.items():

        scrape(key, config, out)

---

## 14\. Empty Product Schema (JSON) {#14.-empty-product-schema-(json)}

See `jane_flowhub_product_schema.json` (delivered alongside this document).

---

## 15\. Anti-Bot / TLS Considerations {#15.-anti-bot-/-tls-considerations}

### Why `curl_cffi`?

The Algolia endpoint and the dispensary HTML pages are served behind Cloudflare. Standard Python `requests` uses a Python TLS fingerprint that Cloudflare detects and blocks. `curl_cffi` uses compiled libcurl with Chrome's exact TLS fingerprint:

pip install curl\_cffi

from curl\_cffi import requests

resp \= requests.get(url, impersonate="chrome", timeout=15)

resp \= requests.post(url, headers=..., json=..., impersonate="chrome", timeout=30)

### Fallback: Standard `requests`

If `curl_cffi` is unavailable, standard `requests` may still work for the Algolia endpoint (since it's a POST with the proper Algolia headers). Use it as a fallback:

try:

    from curl\_cffi import requests as http

    IMPERSONATE \= {"impersonate": "chrome"}

except ImportError:

    import requests as http

    IMPERSONATE \= {}

resp \= http.post(url, headers=headers, json=payload, timeout=30, \*\*IMPERSONATE)

### Rate Limiting

Algolia search endpoints are generally permissive. However:

- Add a 0.5–1s delay between pages if scraping many stores in sequence  
- Do not fire more than \~10 concurrent requests  
- The API key is read-only / search-only — it cannot modify data

### Credential Rotation

If requests start returning 403:

1. Re-fetch credentials from the dispensary HTML page  
2. Fall back to `https://www.iheartjane.com/stores/<store_id>` for the same credentials  
3. The credentials are public-facing and rotate infrequently (weeks to months)

---

## 16\. Field Reference Table {#16.-field-reference-table}

| Field | Source | Flowhub | Always Present | Notes |
| :---- | :---- | :---- | :---- | :---- |
| `product_id` | Jane | No | Yes | Jane's internal product ID |
| `objectID` | Algolia | No | Yes | Algolia record ID |
| `name` | Jane/Flowhub | Yes | Yes | Product display name |
| `brand` | Jane/Flowhub | Yes | Yes | Brand name |
| `kind` | Jane | No | Yes | Top-level category |
| `kind_subtype` | Jane | No | No | Sub-category |
| `pos_product_lookup` | Flowhub | **Yes** | Yes | Flowhub UUID per weight |
| `percent_thc` | Flowhub/Lab | Yes | No | THC potency % |
| `percent_thca` | Flowhub/Lab | Yes | No | THCA potency % |
| `price_*` | Flowhub | Yes | Partial | Per-weight pricing |
| `special_price_*` | Jane | No | No | Time-limited discounts |
| `available_for_pickup` | Flowhub | Yes | Yes | Live stock flag |
| `available_for_delivery` | Flowhub | Yes | Yes | Live delivery flag |
| `product_photos` | Jane | No | No | CDN-hosted images |
| `indexed_at` | Algolia | No | Yes | Last sync timestamp |
| `lab_results` | Flowhub/COA | Yes | No | Certificate of Analysis |
| `store_notes` | Flowhub | Yes | No | Dispensary-added notes |

---

## 17\. Common Errors & Troubleshooting {#17.-common-errors-&-troubleshooting}

| Error | Cause | Fix |
| :---- | :---- | :---- |
| `403 Forbidden` on Algolia | Bad/expired API key | Re-fetch credentials dynamically |
| `403 Forbidden` on dispensary site | Cloudflare blocking | Use `curl_cffi` with `impersonate="chrome"` |
| Empty `hits` array | Wrong `store_id` | Verify store\_id from page HTML |
| `nbHits: 0` | Store temporarily offline | Retry after 1–2 min; check `indexed_at` |
| Missing price fields | Product out of stock | Check `available_for_pickup` flag |
| `pos_product_lookup` is `null` | Non-Flowhub product | Rare; some demo/test products |
| SSL/TLS error | Cloudflare TLS | Upgrade `curl_cffi`; use `impersonate="chrome126"` |
| Credentials not in HTML | SPA-rendered site | Fetch from `iheartjane.com/stores/<id>` directly |
| Pagination loop | `nbPages` \= 0 | Treat as 1 page; `while page < max(nbPages, 1)` |

---

## Appendix A: Store ID Discovery

To find the `store_id` for any Jane dispensary:

1. **From dispensary HTML:** search for `store_id` or `storeId` in the page source  
2. **From iHeartJane URL:** `https://www.iheartjane.com/stores/NNNN/...` — the number is the store\_id  
3. **From Algolia objectID:** each product's `objectID` follows the pattern `prod_<product_id>_store_<store_id>`  
4. **Google search:** `site:iheartjane.com "dispensary name"` to find the iHeartJane store page

## Appendix B: Category Values

Known `kind` values in the Jane system:

flower        — Flower/bud

extract       — Concentrates, wax, shatter, etc.

edible        — Edibles

pre\_roll      — Pre-rolls

tincture      — Tinctures

topical       — Topicals, lotions, patches

vaporizer     — Vapes/cartridges

gear          — Accessories

capsule       — Capsules

clone         — Clone plants

## Appendix C: Algolia Filter Syntax

\# Single store

"filters": "store\_id:5876"

\# Store \+ category

"filters": "store\_id:5876 AND kind:flower"

\# Store \+ multiple categories

"filters": "store\_id:5876 AND (kind:flower OR kind:pre\_roll)"

\# Store \+ price range

"filters": "store\_id:5876 AND bucket\_price:1 TO 50"

\# Store \+ in-stock only

"filters": "store\_id:5876 AND available\_for\_pickup:true"

\# Store \+ on-sale

"filters": "store\_id:5876 AND special\_id \> 0"

---

*Report generated: 2026-06-23 | Platform: iHeartJane (Jane) | POS: Flowhub | Scraping method: Algolia REST API*  
