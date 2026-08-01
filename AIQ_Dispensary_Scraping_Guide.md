# AIQ Dispensaries — Developer Scraping Guide

**Platform:** AIQ Ecommerce (formerly Dispense)  
**API Base URL:** `https://api.dispenseapp.com`  
**API Version:** `2023-03`  
**Last Validated:** June 2026

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)  
2. [Authentication & API Key](#2-authentication--api-key)  
3. [Required Request Headers](#3-required-request-headers)  
4. [Venue Discovery & Validation](#4-venue-discovery--validation)  
5. [Core API Endpoints](#5-core-api-endpoints)  
6. [Pagination Pattern](#6-pagination-pattern)  
7. [Product Schema](#7-product-schema)  
8. [Category Schema](#8-category-schema)  
9. [Venue Schema](#9-venue-schema)  
10. [Offer/Deal Schema](#10-offerdeal-schema)  
11. [Enum Reference Values](#11-enum-reference-values)  
12. [De-duplication Strategy](#12-de-duplication-strategy)  
13. [Complete Scraper — Reference Implementation (Python)](#13-complete-scraper--reference-implementation-python)  
14. [Gotchas & Edge Cases](#14-gotchas--edge-cases)  
15. [Empty JSON Schemas](#15-empty-json-schemas)

---

## 1\. Architecture Overview

AIQ dispensary menus are served via a **Next.js SPA** at:

https://menus.dispenseapp.com/{venue\_slug}/menu

> ⚠️ The menu page is **client-side rendered** — no `__NEXT_DATA__` or SSR product data is embedded in the HTML. All product, category, and venue data is fetched at runtime from `api.dispenseapp.com`.

**Key architectural facts:**

- All product data lives entirely in the REST API — no DOM scraping required  
- No CDN bypass or CAPTCHA handling needed — standard HTTPS requests return `200`  
- The same API powers all AIQ-hosted dispensary menus regardless of their custom domain  
- Venues are identified by a **16-character hex ID** (e.g. `578f14c8e658dfdc`) embedded in menu URLs and API paths  
- The API version path segment is `2023-03` (not `/v1/` — earlier docs mentioned `/v1/` but the live, correct version is `2023-03`)

---

## 2\. Authentication & API Key

### The Key

Every request requires a single static public API key sent as the `x-dispense-api-key` header.

**Current working key (as of June 2026):**

a4b23334-1c11-4e6c-bc18-29f3287c05e6

This key is the same across all AIQ/Dispense-hosted menus — it is a **platform-level public key**, not per-dispensary.

### How to Re-Derive the Key (if it rotates)

The key is embedded in the webpack JS bundle loaded by any Dispense menu page. Steps to extract:

1. Navigate any live Dispense menu (e.g. `https://www.highscore-cannabis.com/shop`)  
2. Open DevTools → **Network tab** → filter by `api.dispenseapp.com`  
3. Inspect the `x-dispense-api-key` request header on any outbound API call

**Or via JavaScript (in browser console on any Dispense menu page):**

// Intercept outbound fetch calls to capture the key

const \_fetch \= window.fetch;

window.fetch \= function(url, opts) {

  if (url.includes('api.dispenseapp.com')) {

    console.log('API Key:', opts?.headers?.\['x-dispense-api-key'\]);

  }

  return \_fetch.apply(this, arguments);

};

**Or via webpack module bundle (headless):**

import re, httpx

\# 1\. Fetch the menu HTML to get current chunk URLs

html \= httpx.get("https://menus.dispenseapp.com/highscore-cannabis/menu").text

chunk\_urls \= re.findall(r'src="(https://menus-\[^"\]+/\_next/static/chunks/\[^"\]+\\.js)"', html)

\# 2\. Search chunks for the UUID-shaped API key

uuid\_pattern \= re.compile(r"x-dispense-api-key\["'\]?:\\s\*\["'\](\[0-9a-f-\]{36})\["'\]")

for url in chunk\_urls:

    text \= httpx.get(url).text

    match \= uuid\_pattern.search(text)

    if match:

        print("Found key:", match.group(1))

        break

---

## 3\. Required Request Headers

| Header | Value | Notes |
| :---- | :---- | :---- |
| `x-dispense-api-key` | `a4b23334-1c11-4e6c-bc18-29f3287c05e6` | Required on every request |
| `content-type` | `application/json` | Required |
| `Referer` | `https://{dispensary_domain}/` | Recommended — set to the dispensary's own domain |

**Missing `x-dispense-api-key` returns:**

{"name":"UnauthenticatedError","message":"Unauthorized","errors":\[{"type":"UnauthenticatedError","message":"invalid venue id"}\]}

**Optional headers (only needed for logged-in user sessions):**

x-access-token: \<jwt\>          \# authenticated user

x-prospect-token: \<token\>      \# guest/prospect session

---

## 4\. Venue Discovery & Validation

### Finding the Venue ID

The venue ID is the 16-char hex slug in the API URL, visible in network requests when a menu page loads:

https://api.dispenseapp.com/2023-03/venues/578f14c8e658dfdc

It also appears in `productUrl` fields on any product:

https://menus.dispenseapp.com/578f14c8e658dfdc/menu/none/{product-slug}

### Verifying a Venue is Active

**Always call this first before scraping a venue's menu:**

GET https://api.dispenseapp.com/2023-03/venues/{venue\_id}

| Response | Meaning |
| :---- | :---- |
| `200` with venue object | Venue is active — proceed with scraping |
| `404` `{"message":"venue not found"}` | Venue is deleted or never existed |

> ⚠️ **Critical gotcha:** A deleted venue's `/products` endpoint returns `{"data":[],"count":0}` with HTTP `200`. An empty product list does NOT mean the venue is down — always verify with the `/venues/{id}` endpoint first.

### Listing All Venues (Organization Level)

If you have an `organizationId`, you can list all venues in the org:

GET https://api.dispenseapp.com/2023-03/venues?organizationId={org\_id}

The `organizationId` is a 16-char hex string found in product objects as the `organization` field (e.g. `0a4b555c8fbcc621`).

---

## 5\. Core API Endpoints

All endpoints base: `https://api.dispenseapp.com/2023-03`

### 5.1 Venue Info

GET /venues/{venue\_id}

Returns venue name, address, hours, logo, SEO config. Use to validate venue existence.

### 5.2 Product Categories

GET /categories?venueId={venue\_id}

Returns all category definitions for the venue (Flower, Edibles, Offers, etc.)  
**Key response fields:** `id`, `name`, `type` (`SYSTEM` | `OFFERS` | `CUSTOM`), `slug`, `enable`, `filterTypes`, `iconImage`

### 5.3 List Products (Primary endpoint)

GET /products?venueId={venue\_id}\&active=true\&enable=true\&quantityMin=1\&group=true\&limit=250\&skip=0

**Key query parameters:**

| Param | Type | Description |
| :---- | :---- | :---- |
| `venueId` | string | **Required** — 16-char hex venue ID |
| `active` | boolean | Filter active products (`true` recommended) |
| `enable` | boolean | Filter enabled products (`true` recommended) |
| `quantityMin` | number | Minimum stock (use `1` to exclude out-of-stock) |
| `group` | boolean | `true` \= return one row per product group (multiple weights merged); `false` \= one row per SKU/variant |
| `limit` | number | Page size — max `250`, default `100` |
| `skip` | number | Offset for pagination |
| `cannabisComplianceType` | string/array | Filter by product type (see enum list §11) |
| `cannabisType` | string/array | Filter by SATIVA/INDICA/HYBRID/etc. |
| `categoryId` | string/array | Filter by category ID |
| `brand` | string/array | Filter by brand name |
| `search` | string | Full-text product search |
| `featured` | boolean | Only featured products |
| `new` | boolean | Only products flagged as new |
| `discounted` | boolean | Only discounted products |
| `weightFormatted` | string/array | Filter by weight (e.g. `3.5g`, `100mg`) |
| `sort` | enum | `name`, `-name`, `brand.name`, `-brand.name` |
| `createdStart` / `createdEnd` | ISO datetime string | Date range filter on created |
| `modifiedStart` / `modifiedEnd` | ISO datetime string | Date range filter on modified |

### 5.4 Get Single Product

GET /products/{product\_id}?venueId={venue\_id}

Returns full product detail. The `venueId` query param is required even for single-item lookup.

### 5.5 Offers / Deals

GET /offers?venueId={venue\_id}\&active=true

Returns deal structures (BOGO, percent-off, flat-off, etc.) with their rule sets.

### 5.6 Product Tags

GET /product-tags?venueId={venue\_id}

Returns custom tag definitions assigned to products.

### 5.7 Product Reviews

GET /reviews?venueId={venue\_id}\&productId={product\_id}\&limit=100\&skip=0

### 5.8 Banners

GET /banners?venueId={venue\_id}

Returns promotional banners configured for the menu.

### 5.9 Delivery Zones

GET /delivery-zones?venueId={venue\_id}

---

## 6\. Pagination Pattern

The `/products` (and most list) endpoints use `skip` \+ `limit` pagination.

**Response envelope:**

{

  "count": 19,

  "pageCount": 19,

  "skip": 0,

  "limit": 250,

  "data": \[...\]

}

> ⚠️ `pageCount` is the count of items in the **current page**, NOT the total page count. Use `count` for the total record count.

**Paginate until exhausted:**

all\_products \= \[\]

skip \= 0

limit \= 250

while True:

    resp \= requests.get(

        f"{BASE\_URL}/products",

        params={"venueId": venue\_id, "active": "true", "enable": "true",

                "group": "false", "limit": limit, "skip": skip},

        headers=HEADERS

    )

    batch \= resp.json()

    all\_products.extend(batch\["data"\])

    if len(batch\["data"\]) \< limit:

        break

    skip \+= limit

**Tip:** Use `group=false` during full pagination to get every individual SKU/variant. Use `group=true` for a grouped/de-duped view suitable for display.

---

## 7\. Product Schema

Full annotated schema of a product object as returned by the API:

{

  "id":                     "16-char hex string — primary identifier",

  "\_id":                    "same as id — legacy alias",

  "created":                "ISO 8601 datetime — when product was created",

  "modified":               "ISO 8601 datetime — last update timestamp",

  "name":                   "string — product display name",

  "slug":                   "string — URL-safe identifier",

  "description":            "string (may contain HTML) | null",

  "sku":                    "string | null — POS SKU",

  "image":                  "string — primary image URL (imgix CDN)",

  "images": \[

    {

      "fileUrl":            "string — image URL",

      "order":              "number — display sort order"

    }

  \],

  "productUrl":             "string — canonical menu URL for this product",

  "venue":                  "string — venue hex ID this product belongs to",

  "organization":           "string — organization hex ID",

  "cannabisComplianceType": "enum string — see §11 for all values",

  "productCategoryName":    "string — human-readable category label",

  "cannabisType":           "enum string — SATIVA | INDICA | HYBRID | HYBRID\_INDICA | CBD | NA",

  "cannabisStrain":         "string | null — strain name",

  "subType":                "string | null — sub-category (e.g. Sauce, Pre Rolls \- Single, Tier 2 Flower)",

  "brand": {

    "name":                 "string | null"

  },

  "weight":                 "number | null — weight value",

  "weightUnit":             "string | null — GRAMS | MILLIGRAMS | OUNCES | POUNDS | KILOGRAMS",

  "weightFormatted":        "string — human readable (e.g. 3.5g, 100mg)",

  "flowerEquivalentInGrams":"number | null — for compliance equivalency calculations",

  "price":                  "number — retail price in USD",

  "priceType":              "string — REGULAR | MEMBER",

  "priceGross":             "number — gross price before discounts (on variant objects)",

  "priceNet":               "number — net price after discounts (on variant objects)",

  "priceWithDiscounts":     "number — final price after all discounts applied",

  "discountValue":          "number — discount amount (0.1 \= 10% or $10 depending on type)",

  "discountType":           "string — PERCENT | FLAT",

  "discountAmountFinal":    "number — computed discount dollar amount (on variants)",

  "discounts": \[

    {

      "value":              "number",

      "type":               "string — PERCENT | FLAT"

    }

  \],

  "quantity":               "number — current in-stock quantity",

  "quantityTotal":          "number | null",

  "quantitySold":           "number | null",

  "quantityThreshold":      "number | null — low-stock threshold",

  "groupId":                "string | null — UUID linking product variants into a group",

  "labs": {

    "thc":                  "number | null — THC % or mg value",

    "thcMax":               "number | null — max THC (often the thcA value)",

    "thcA":                 "number | null — THCa percentage",

    "thcContentUnit":       "string | null — % or mg",

    "thcAContentUnit":      "string | null — %",

    "cbd":                  "number | null — CBD value",

    "cbdMax":               "number | null",

    "cbdContentUnit":       "string | null — % or mg",

    "cbg":                  "number | null",

    "cbn":                  "number | null",

    "terpenes": \[

      "string — terpene name"

    \]

  },

  "posLastSyncData":        "string | null — raw JSON string with POS/lab fallback data",

  "effects":                \["string"\] ,

  "terpenes":               \["string"\] ,

  "featured":               "boolean",

  "new":                    "boolean",

  "enable":                 "boolean — product is enabled/visible",

  "variants": \[

    {

      "id":                 "string",

      "\_id":                "string",

      "groupId":            "string — UUID linking this variant to its product group",

      "groupOrder":         "number — sort order within group",

      "name":               "string",

      "enable":             "boolean",

      "deleted":            "boolean",

      "price":              "number",

      "priceGross":         "number",

      "priceNet":           "number",

      "priceWithDiscounts": "number",

      "discountAmountFinal":"number",

      "discountValueFinal": "number",

      "discountTypeFinal":  "string",

      "discountValue":      "number",

      "discountType":       "string",

      "discounts":          \[\],

      "tiers":              \[\],

      "modifierGroups":     \[\],

      "cannabisComplianceType": "string",

      "quantity":           "number",

      "quantityThreshold":  "number | null",

      "weight":             "number | null",

      "weightInGrams":      "number | null",

      "weightUnit":         "string | null",

      "weightFormatted":    "string",

      "images":             \[\],

      "productCategory":    "string — category hex ID",

      "productCategoryName":"string",

      "productCategoryIconImage": "string — URL",

      "slug":               "string"

    }

  \],

  "reviewStats": {

    "total":                "number — total review count",

    "averageRating":        "number — 0-5 star average"

  }

}

### Lab Values Fallback Strategy

The `labs` object is often partially populated or null on older products. Fall back to `posLastSyncData`:

import json as \_json

def get\_lab\_value(product, field):

    \# Try labs object first

    labs \= product.get("labs") or {}

    if labs.get(field) is not None:

        return labs\[field\]

    \# Fall back to posLastSyncData JSON string

    raw \= product.get("posLastSyncData")

    if raw:

        try:

            pos \= \_json.loads(raw)

            return pos.get(f"labs.{field}") or pos.get(field)

        except Exception:

            pass

    return None

---

## 8\. Category Schema

{

  "id":                     "string — 16-char hex",

  "name":                   "string — display name (Flower, Edibles, etc.)",

  "slug":                   "string — URL slug",

  "type":                   "string — SYSTEM | OFFERS | CUSTOM",

  "order":                  "number — display sort order",

  "enable":                 "boolean",

  "iconImage":              "string — URL to category icon",

  "defaultProductImage":    "string | null — fallback image URL",

  "showHomeIcon":           "boolean",

  "showHomeList":           "boolean",

  "sort":                   "string | null — default sort (ALPHABETICAL, etc.)",

  "venue":                  "string — venue hex ID",

  "organization":           "string — org hex ID",

  "filterTypes":            \["string — cannabisComplianceType values"\],

  "filterBrands":           \["string"\],

  "filterCannabisTypes":    \["string"\],

  "filterSubTypes":         \["string"\] ,

  "filterDiscounted":       "boolean",

  "filterNew":              "boolean",

  "filterFeatured":         "boolean",

  "filterCategories":       \["string"\],

  "filterTerpeneTypes":     \["string"\],

  "filterProducts":         \["string — product IDs"\],

  "pinnedProducts":         \["string — product IDs pinned to top"\],

  "pinnedFilters":          \["string"\],

  "posMappings":            \["object — POS system mappings"\]

}

---

## 9\. Venue Schema

{

  "id":                     "string — 16-char hex venue identifier",

  "name":                   "string — dispensary display name",

  "medicalDispensary":      "boolean",

  "organization":           "string — org hex ID",

  "address": {

    "number":               "string — street number",

    "street":               "string",

    "city":                 "string",

    "administrativeArea":   "string — state code (e.g. MA)",

    "administrativeAreaDisplay": "string — full state name",

    "country":              "string — ISO country code",

    "postalCode":           "string",

    "latitude":             "number",

    "longitude":            "number"

  },

  "addressFormatted":       "string — single-line address",

  "logo":                   "string — URL",

  "logoSquare":             "string — URL",

  "favicon":                "string — URL",

  "schedule": \[

    {

      "day":                "number — 0=Sunday through 6=Saturday",

      "closed":             "boolean",

      "hours": \[

        {

          "start":          "ISO 8601 datetime (date portion is epoch placeholder)",

          "end":            "ISO 8601 datetime"

        }

      \]

    }

  \],

  "seoMenuMetaData":        "object — SEO title/description templates"

}

---

## 10\. Offer/Deal Schema

{

  "id":                     "string — 16-char hex",

  "name":                   "string — deal display name",

  "internalName":           "string — internal label",

  "description":            "string",

  "enable":                 "boolean",

  "deleted":                "boolean",

  "order":                  "number",

  "type":                   "string — DEAL | LOYALTY | etc.",

  "dealType":               "string — BOGO | PERCENT | FLAT | BUNDLE | etc.",

  "rewardType":             "string — PERCENT | FLAT",

  "rewardValue":            "number",

  "minimumSpend":           "number",

  "promoCodeRequired":      "boolean",

  "scheduleEnabled":        "boolean",

  "schedule": {

    "days":                 \["number"\],

    "startDate":            "ISO 8601 | null",

    "endDate":              "ISO 8601 | null",

    "startTimeHour":        "number | null",

    "startTimeMinutes":     "number | null",

    "endTimeHour":          "number | null",

    "endTimeMinutes":       "number | null"

  },

  "rules": \[

    {

      "id":                 "string",

      "quantity":           "number — items required to trigger rule",

      "products":           \["string — product IDs"\],

      "productsExclude":    "boolean",

      "productFilters": {

        "categories":       \["string"\],

        "categoriesExclude":"boolean",

        "brands":           \["string"\],

        "brandsExclude":    "boolean",

        "subTypes":         \["string"\],

        "subTypesExclude":  "boolean",

        "weights":          \["string"\],

        "weightsExclude":   "boolean",

        "cannabisTypes":    \["string"\],

        "cannabisTypesExclude": "boolean",

        "tags":             \["string"\],

        "tagsExclude":      "boolean"

      },

      "rewardType":         "string — PERCENT | FLAT",

      "rewardValue":        "number | null"

    }

  \],

  "segmentation": {

    "venues":               \["string — venue IDs"\],

    "orderPickUpTypes":     \["string"\]

  },

  "disableStacking":        "boolean",

  "stackingLimit":          "number | null",

  "customerLimit":          "number | null",

  "redemptions":            \[\]

}

---

## 11\. Enum Reference Values

### `cannabisComplianceType`

| Value | Display Name |
| :---- | :---- |
| `FLOWER` | Flower |
| `PRE_ROLLS` | Pre Rolls |
| `VAPORIZERS` | Vaporizers |
| `CONCENTRATES` | Concentrates |
| `EDIBLES` | Edibles |
| `TINCTURES` | Tinctures |
| `TOPICALS` | Topicals |
| `BEVERAGES` | Beverages |
| `ACCESSORIES` | Accessories |
| `MERCHANDISE` | Merchandise |
| `CAPSULES` | Capsules |
| `SEEDS` | Seeds |

### `cannabisType`

`SATIVA` | `INDICA` | `HYBRID` | `HYBRID_INDICA` | `HYBRID_SATIVA` | `CBD` | `NA`

### `weightUnit`

`GRAMS` | `MILLIGRAMS` | `OUNCES` | `POUNDS` | `KILOGRAMS`

### `discountType`

`PERCENT` | `FLAT`

### `priceType`

`REGULAR` | `MEMBER`

### `category.type`

`SYSTEM` | `OFFERS` | `CUSTOM`

### `schedule.day` (day of week)

`0` \= Sunday, `1` \= Monday, `2` \= Tuesday, `3` \= Wednesday, `4` \= Thursday, `5` \= Friday, `6` \= Saturday

---

## 12\. De-duplication Strategy

Products appear in **multiple categories**. A product in the "Offers" (`OFFERS` type) category is also in its native `SYSTEM` category (Flower, Edibles, etc.) — but with `discountValue`/`discountType` populated when on sale.

**Rule:**

- If iterating over all categories and their products, **always de-duplicate by `product.id`** before counting or storing unique products  
- The `OFFERS` / `CUSTOM` category copies are useful for capturing active discount metadata — scrape them, but de-dupe by ID before your final product list

seen\_ids \= set()

unique\_products \= \[\]

for product in all\_category\_products:

    if product\["id"\] not in seen\_ids:

        seen\_ids.add(product\["id"\])

        unique\_products.append(product)

---

## 13\. Complete Scraper — Reference Implementation (Python)

import httpx

import time

import json

from typing import Generator

BASE\_URL \= "https://api.dispenseapp.com/2023-03"

API\_KEY  \= "a4b23334-1c11-4e6c-bc18-29f3287c05e6"  \# public platform key

def get\_headers(dispensary\_domain: str) \-\> dict:

    return {

        "x-dispense-api-key": API\_KEY,

        "content-type": "application/json",

        "Referer": f"https://{dispensary\_domain}/",

    }

def validate\_venue(venue\_id: str, domain: str) \-\> dict | None:

    """Returns venue dict if active, None if deleted/not found."""

    resp \= httpx.get(

        f"{BASE\_URL}/venues/{venue\_id}",

        headers=get\_headers(domain),

        timeout=15

    )

    if resp.status\_code \== 404:

        print(f"\[WARN\] Venue {venue\_id} not found or deleted")

        return None

    resp.raise\_for\_status()

    return resp.json()

def get\_categories(venue\_id: str, domain: str) \-\> list:

    resp \= httpx.get(

        f"{BASE\_URL}/categories",

        params={"venueId": venue\_id},

        headers=get\_headers(domain),

        timeout=15

    )

    resp.raise\_for\_status()

    return resp.json().get("data", \[\])

def paginate\_products(venue\_id: str, domain: str, \*\*filters) \-\> Generator\[dict, None, None\]:

    """Yield all products for a venue, handling pagination automatically."""

    limit \= 250

    skip  \= 0

    params \= {

        "venueId": venue\_id,

        "active": "true",

        "enable": "true",

        "group": "false",   \# get individual SKUs

        "limit": limit,

        "skip": skip,

        \*\*filters,

    }

    while True:

        params\["skip"\] \= skip

        resp \= httpx.get(

            f"{BASE\_URL}/products",

            params=params,

            headers=get\_headers(domain),

            timeout=30

        )

        resp.raise\_for\_status()

        body \= resp.json()

        batch \= body.get("data", \[\])

        for product in batch:

            yield product

        if len(batch) \< limit:

            break

        skip \+= limit

        time.sleep(0.25)  \# polite delay — no hard rate limit observed

def get\_offers(venue\_id: str, domain: str) \-\> list:

    resp \= httpx.get(

        f"{BASE\_URL}/offers",

        params={"venueId": venue\_id, "active": "true"},

        headers=get\_headers(domain),

        timeout=15

    )

    resp.raise\_for\_status()

    return resp.json().get("data", \[\])

def scrape\_venue(venue\_id: str, dispensary\_domain: str) \-\> dict:

    """

    Full scrape of a single AIQ dispensary venue.

    Returns structured dict with venue info, categories, products, and offers.

    """

    \# Step 1: Validate venue exists

    venue \= validate\_venue(venue\_id, dispensary\_domain)

    if not venue:

        return {"error": "venue\_not\_found", "venue\_id": venue\_id}

    print(f"Scraping: {venue\['name'\]} ({venue\['addressFormatted'\]})")

    \# Step 2: Get categories

    categories \= get\_categories(venue\_id, dispensary\_domain)

    print(f"  Categories: {\[c\['name'\] for c in categories\]}")

    \# Step 3: Paginate all products (de-duplicated)

    all\_products \= {}

    for product in paginate\_products(venue\_id, dispensary\_domain):

        all\_products\[product\["id"\]\] \= product

    print(f"  Products: {len(all\_products)} unique SKUs")

    \# Step 4: Get offers/deals

    offers \= get\_offers(venue\_id, dispensary\_domain)

    print(f"  Offers: {len(offers)}")

    return {

        "venue": venue,

        "categories": categories,

        "products": list(all\_products.values()),

        "offers": offers,

    }

\# Example usage:

if \_\_name\_\_ \== "\_\_main\_\_":

    result \= scrape\_venue(

        venue\_id="578f14c8e658dfdc",

        dispensary\_domain="www.highscore-cannabis.com"

    )

    with open("highscore\_menu.json", "w") as f:

        json.dump(result, f, indent=2)

    print(f"Saved {len(result\['products'\])} products")

---

## 14\. Gotchas & Edge Cases

| Scenario | Behavior | How to Handle |
| :---- | :---- | :---- |
| Deleted venue `/products` call | Returns `{"data":[],"count":0}` with `200 OK` | Always validate with `/venues/{id}` first |
| Missing `venueId` on `/products/{id}` | Returns `401 UnauthenticatedError` | Always include `?venueId=` even on single-product fetches |
| `labs` is null or partially empty | `labs.thc`, `labs.cbd` may be missing | Fall back to `posLastSyncData` JSON string for lab values |
| Products in multiple categories | Same `_id` appears in OFFERS and SYSTEM categories | De-duplicate by `product.id` |
| `group=true` vs `group=false` | `true` merges weight variants into one row; `false` returns each weight as its own row | Use `false` for full inventory, `true` for display |
| `pageCount` field | This is the count of items on the **current page**, NOT the number of pages | Use `count` for total records; paginate while batch size \== limit |
| API key rotation | The key is public but may change on platform deploys | Re-extract from browser network tab headers if getting `401` |
| `price` vs `priceWithDiscounts` | `price` \= list price, `priceWithDiscounts` \= price after active discounts | Always present both for accurate display |
| `weightFormatted` can be empty string | Even when `weight` has a value, `weightFormatted` may be `""` | Format manually: `f"{weight}{unit_abbrev[weightUnit]}"` |
| `effects` can be `null` or `[]` | Not all products have effect tags | Treat both as "no effects" |
| `brand.name` can be `null` | Brand object exists but name is null | Always check `brand?.name` not just `brand` |

---

## 15\. Empty JSON Schemas

### 15.1 Empty Product Schema

{

  "id": null,

  "\_id": null,

  "created": null,

  "modified": null,

  "name": null,

  "slug": null,

  "description": null,

  "sku": null,

  "image": null,

  "images": \[\],

  "productUrl": null,

  "venue": null,

  "organization": null,

  "cannabisComplianceType": null,

  "productCategoryName": null,

  "cannabisType": null,

  "cannabisStrain": null,

  "subType": null,

  "brand": {

    "name": null

  },

  "weight": null,

  "weightUnit": null,

  "weightFormatted": null,

  "flowerEquivalentInGrams": null,

  "price": null,

  "priceType": null,

  "priceGross": null,

  "priceNet": null,

  "priceWithDiscounts": null,

  "discountValue": null,

  "discountType": null,

  "discountAmountFinal": null,

  "discounts": \[\],

  "quantity": null,

  "quantityTotal": null,

  "quantitySold": null,

  "quantityThreshold": null,

  "groupId": null,

  "labs": {

    "thc": null,

    "thcMax": null,

    "thcA": null,

    "thcContentUnit": null,

    "thcAContentUnit": null,

    "cbd": null,

    "cbdMax": null,

    "cbdContentUnit": null,

    "cbg": null,

    "cbn": null,

    "terpenes": \[\]

  },

  "posLastSyncData": null,

  "effects": \[\],

  "terpenes": \[\],

  "featured": null,

  "new": null,

  "enable": null,

  "variants": \[\],

  "reviewStats": {

    "total": null,

    "averageRating": null

  }

}

### 15.2 Empty Venue Schema

{

  "id": null,

  "name": null,

  "medicalDispensary": null,

  "organization": null,

  "address": {

    "number": null,

    "street": null,

    "city": null,

    "administrativeArea": null,

    "administrativeAreaDisplay": null,

    "country": null,

    "postalCode": null,

    "latitude": null,

    "longitude": null

  },

  "addressFormatted": null,

  "logo": null,

  "logoSquare": null,

  "favicon": null,

  "schedule": \[\]

}

### 15.3 Empty Category Schema

{

  "id": null,

  "name": null,

  "slug": null,

  "type": null,

  "order": null,

  "enable": null,

  "iconImage": null,

  "defaultProductImage": null,

  "showHomeIcon": null,

  "showHomeList": null,

  "sort": null,

  "venue": null,

  "organization": null,

  "filterTypes": \[\],

  "filterBrands": \[\],

  "filterCannabisTypes": \[\],

  "filterSubTypes": \[\],

  "filterDiscounted": null,

  "filterNew": null,

  "filterFeatured": null,

  "filterCategories": \[\],

  "filterTerpeneTypes": \[\],

  "filterProducts": \[\],

  "pinnedProducts": \[\],

  "pinnedFilters": \[\],

  "posMappings": \[\]

}

### 15.4 Empty Offer Schema

{

  "id": null,

  "name": null,

  "internalName": null,

  "description": null,

  "enable": null,

  "deleted": null,

  "order": null,

  "type": null,

  "dealType": null,

  "rewardType": null,

  "rewardValue": null,

  "minimumSpend": null,

  "promoCodeRequired": null,

  "scheduleEnabled": null,

  "schedule": {

    "days": \[\],

    "startDate": null,

    "endDate": null,

    "startTimeHour": null,

    "startTimeMinutes": null,

    "endTimeHour": null,

    "endTimeMinutes": null

  },

  "rules": \[\],

  "segmentation": {

    "venues": \[\],

    "orderPickUpTypes": \[\]

  },

  "disableStacking": null,

  "stackingLimit": null,

  "customerLimit": null,

  "redemptions": \[\]

}

---

*Report generated June 2026\. API validated against Highscore Cannabis (venue `578f14c8e658dfdc`). All schemas are derived from live API responses.*  
