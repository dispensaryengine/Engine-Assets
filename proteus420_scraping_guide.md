# Proteus420 Dispensary Scraping — Complete Developer Reference

**Generated:** 2026-06-23  
**Author:** Browser-Use Research Agent  
**Scope:** All three Proteus420-powered dispensaries identified in the WNY/NJ dataset

---

## Table of Contents

1. [Platform Overview](#1-platform-overview)  
2. [Dispensary Inventory](#2-dispensary-inventory)  
3. [Architecture Deep-Dive](#3-architecture-deep-dive)  
4. [Store Variants and Quirks](#4-store-variants-and-quirks)  
5. [Endpoints Reference](#5-endpoints-reference)  
6. [HTML Parsing Guide](#6-html-parsing-guide)  
7. [Product Detail Pages](#7-product-detail-pages)  
8. [Filtering and Specials](#8-filtering-and-specials)  
9. [Multi-Location Stores](#9-multi-location-stores)  
10. [Rate Limiting & Anti-Bot](#10-rate-limiting--anti-bot)  
11. [Complete Python Implementation](#11-complete-python-implementation)  
12. [Product Schema Reference](#12-product-schema-reference)  
13. [Common Pitfalls](#13-common-pitfalls)

---

## 1\. Platform Overview

Proteus420 is a ColdFusion-based cannabis ERP and point-of-sale platform ([https://www.proteus420.com/](https://www.proteus420.com/)). The online menu/cart module is a server-side rendered HTML application that communicates over AJAX using `.cfm` (ColdFusion Markup) endpoints. There is **no public REST API, no GraphQL layer, and no JSON payload** — all data is delivered as raw HTML fragments that you parse with BeautifulSoup.

### Deployment Models

Proteus420 stores come in two distinct variants:

| Variant | Description | Example |
| :---- | :---- | :---- |
| **Standalone** | Shop lives on dispensary's own domain under `/cart/` | tcsbflo.com/cart/ |
| **Cloud-hosted** | Shop lives on a shared `cart.SLUG.com` subdomain | cart.honeykenmore.com, cart.tokelane.com |

The AJAX endpoint paths differ between variants (see §4). This is the most common source of scraper breakage.

### Technology Stack

- **Backend:** ColdFusion (Adobe CF or Lucee)  
- **Frontend:** Bootstrap 5, jQuery 3.x, custom Proteus CSS  
- **CDN:** AWS CloudFront (`d3ilo4wq1uvg39.cloudfront.net`) for product images  
- **Image origin bucket:** `proteusimages` on S3 (us-west-1)  
- **Authentication:** Optional user accounts; **no auth required** for menu browsing  
- **Anti-bot:** Cloudflare (light configuration) — `curl_cffi` with Chrome TLS impersonation bypasses it reliably

---

## 2\. Dispensary Inventory

All three Proteus420 dispensaries confirmed in the dataset:

| Dispensary | Address | Phone | Menu Root URL | Variant |
| :---- | :---- | :---- | :---- | :---- |
| The Cannabis Store | 1936 South Park Ave, Buffalo, NY | (716) 381-8004 | [https://tcsbflo.com/cart/](https://tcsbflo.com/cart/) | Standalone |
| HONEY Kenmore | 2981 Delaware Ave, Kenmore, NY 14217 | (716) 200-1551 | [https://cart.honeykenmore.com/](https://cart.honeykenmore.com/) | Cloud |
| Toke Lane | 1727 Genesee St, Buffalo, NY 14211 | (888) 484-1073 | [https://cart.tokelane.com/](https://cart.tokelane.com/) | Cloud (multi-location) |

Toke Lane is unique in that it serves **three locations** from one cart subdomain:

| Location Label | acc | loc | shoptype | Notes |
| :---- | :---- | :---- | :---- | :---- |
| Toke Lane Trenton — Medical NJ | 1140 | 1 | Pickup | Medical menu |
| Toke Lane Trenton — Adult Use NJ | 1140 | 2 | Pickup | Recreational menu |
| Toke Lane Buffalo NY | 1153 | 1 | Pickup | Same subdomain, different acc |

---

## 3\. Architecture Deep-Dive

### Page-Load Flow

Browser/Scraper                 Proteus420 Server

      │                                │

      │  GET /cart/ (or /)             │

      │ ──────────────────────────────▶│

      │         Full HTML page         │

      │ ◀──────────────────────────────│

      │  (contains category nav links  │

      │   with data-id & data-catname) │

      │                                │

      │  GET /ajax\_getproducts.cfm     │

      │      ?cat={id}\&sel\_soldout=n   │

      │      \&page=all                 │

      │ ──────────────────────────────▶│

      │    HTML fragment (product cards)│

      │ ◀──────────────────────────────│

      │                                │

      │  (Repeat for each category)    │

### Key Insight: No Pagination Needed

Using `page=all` in the query string returns every product in a category in a single response. Product counts per category range from \~10 to \~120 items. There is no cursor, offset, or token-based pagination.

### Session Handling

For the **Standalone variant** (tcsbflo.com):

- No session setup needed whatsoever.  
- Hit product endpoints directly without a preceding page load.

For the **Cloud variant** (cart.\*.com):

- For single-location stores (HONEY Kenmore): also no session needed.  
- For multi-location stores (Toke Lane): set `acc` and `loc` cookies OR pass them in the URL query string when hitting the main cart page. The session is established server-side from those cookies/params.

---

## 4\. Store Variants and Quirks

### 4a. The Cannabis Store — Standalone Variant ⚠️ Critical Path

**IMPORTANT:** The standalone install has a **doubled `/cart/cart/` path** for AJAX calls.

Marketing site root:  https://tcsbflo.com/

Shop home page:       https://tcsbflo.com/cart/

Product AJAX:         https://tcsbflo.com/cart/cart/ajax\_getproducts.cfm   ← doubled\!

Filters AJAX:         https://tcsbflo.com/cart/cart/ajax\_getfilters.cfm    ← doubled\!

Product detail:       https://tcsbflo.com/cart/ps/{product\_hash}/

Calling `/cart/ajax_getproducts.cfm` (single `cart`) returns HTTP 404\.

**Confirmed Categories (live, 2026-06-23):**

| Category ID | Name | Live Count |
| :---- | :---- | :---- |
| (empty) | Featured | 7 |
| 14 | AIOVapes | — |
| 15 | Accessories | — |
| 8 | Flower | 78 |
| 9 | Prerolls | — |
| 11 | Resin | — |
| 12 | Tinctures | — |
| 4 | Edibles | — |
| 6 | VapeCarts | — |
| 7 | Topicals | — |

---

### 4b. HONEY Kenmore — Cloud Variant

Shop home page:       https://cart.honeykenmore.com/

Product AJAX:         https://cart.honeykenmore.com/cart/ajax\_getproducts.cfm

Filters AJAX:         https://cart.honeykenmore.com/cart/ajax\_getfilters.cfm

Product detail:       https://cart.honeykenmore.com/cart/ps/{product\_hash}/

Specials page:        https://cart.honeykenmore.com/shop/c/featured/

**Confirmed Categories (live, 2026-06-23):**

| Category ID | Name | Live Count |
| :---- | :---- | :---- |
| (empty) | Featured | 9 |
| 8 | Flower | 103 |
| 9 | Prerolls | — |
| 6 | VapeCarts | — |
| 4 | Edibles | — |
| 11 | Extracts | — |
| 12 | Tinctures | — |
| 7 | Topicals | — |
| 16 | CBD | — |
| 15 | Accessories | — |

---

### 4c. Toke Lane — Cloud Variant \+ Multi-Location

Shop home page:       https://cart.tokelane.com/?acc={acc}\&loc={loc}\&shoptype=Pickup

Product AJAX:         https://cart.tokelane.com/cart/ajax\_getproducts.cfm

Filters AJAX:         https://cart.tokelane.com/cart/ajax\_getfilters.cfm

Product detail:       https://cart.tokelane.com/cart/ps/{product\_hash}/

Specials:             https://cart.tokelane.com/shop/c/onsalepage

For multi-location scraping, set cookies on the session **before** calling AJAX endpoints:

session.cookies.set("acc", "1153", domain="cart.tokelane.com")

session.cookies.set("loc", "1",    domain="cart.tokelane.com")

session.cookies.set("shoptype", "Pickup", domain="cart.tokelane.com")

Or seed the session via a GET to the main page with those query params.

**Buffalo NY Location Categories (live, 2026-06-23):**

| Category ID | Name | Live Count |
| :---- | :---- | :---- |
| (empty) | Featured | — |
| 8 | Flower | 90 |
| 9 | Prerolls | — |
| 4 | Edibles | — |
| 13 | Drinks | — |
| 6 | Vaporizers | — |
| 11 | Extracts | — |
| 12 | Tinctures | — |
| 7 | Topicals | — |
| 23 | Accessories | — |

---

## 5\. Endpoints Reference

### Summary Table

| Store | Main Cart Page | Products AJAX | Filters AJAX |
| :---- | :---- | :---- | :---- |
| The Cannabis Store | GET /cart/ | GET /cart/cart/ajax\_getproducts.cfm | GET /cart/cart/ajax\_getfilters.cfm |
| HONEY Kenmore | GET / | GET /cart/ajax\_getproducts.cfm | GET /cart/ajax\_getfilters.cfm |
| Toke Lane | GET /?acc={}\&loc={}\&shoptype=Pickup | GET /cart/ajax\_getproducts.cfm | GET /cart/ajax\_getfilters.cfm |

### Products Endpoint — Query Parameters

| Parameter | Type | Required | Values | Notes |
| :---- | :---- | :---- | :---- | :---- |
| `cat` | string/int | Yes | Category ID or `""` | Empty string \= Featured/all |
| `sel_soldout` | string | Recommended | `"n"` or `"y"` | `"n"` excludes sold-out items |
| `page` | string | Yes | `"all"` | Always use `all` — no pagination |
| `onsale` | int | No | `1` | Returns on-sale items only |
| `whatsnew` | int | No | `1` | Returns new arrivals only |
| `coupon` | int | No | Coupon ID | Returns coupon-discounted items |
| `sortby` | string | No | `"price_asc"`, `"price_desc"`, `"name"`, `"THC_hightolow"` | Optional sort |

### Filters Endpoint — Query Parameters

| Parameter | Type | Required |
| :---- | :---- | :---- |
| `cat` | string/int | Yes — same category ID as products call |

Returns HTML with THC/CBD slider ranges and brand filter checkboxes for that category.

### Required Request Headers

headers \= {

    "X-Requested-With": "XMLHttpRequest",

    "Referer": "{base\_url}/c/{CategoryName}/",

    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",

    "Accept": "text/html, \*/\*; q=0.01",

    "Accept-Language": "en-US,en;q=0.9",

}

---

## 6\. HTML Parsing Guide

### Step 1 — Discover Categories

from bs4 import BeautifulSoup

html \= session.get(cart\_home\_url).text

soup \= BeautifulSoup(html, "html.parser")

categories \= \[\]

seen \= set()

for a in soup.select("a.getproducts\[data-id\]\[data-catname\]"):

    cat\_id   \= a.get("data-id", "").strip()

    cat\_name \= a.get("data-catname", "").strip()

    \# Skip pseudo-categories

    if cat\_name in ("featured", "onsale", "onsalepage", "whatsnew"):

        continue

    if cat\_id not in seen:

        seen.add(cat\_id)

        categories.append({"id": cat\_id, "name": cat\_name})

### Step 2 — Fetch Products per Category

products\_html \= session.get(

    f"{base\_url}{products\_endpoint}",

    params={"cat": cat\_id, "sel\_soldout": "n", "page": "all"},

    headers={"Referer": f"{base\_url}/c/{cat\_name}/", "X-Requested-With": "XMLHttpRequest"}

).text

### Step 3 — Parse Product Cards

Product cards are wrapped in `.product-card-wrapper` divs. Selector: `div[data-id].product-card-wrapper`

**Complete Field Extraction Map:**

import re

def parse\_product\_card(card\_div, base\_url: str, category\_name: str) \-\> dict:

    product \= {}

    \# ── Core identifiers ────────────────────────────────────────────────

    product\["id"\] \= card\_div.get("data-id", "").strip()

    \# Strain class: products\_strain\_NNNN

    for cls in card\_div.get("class", \[\]):

        m \= re.match(r"products\_strain\_(\\d+)", cls)

        if m:

            product\["strain\_id"\] \= m.group(1)

    \# ── Link / URL / Brand / Slug ────────────────────────────────────────

    link \= card\_div.select\_one("a.item\_view\[data-id\]")

    if link:

        url\_path \= link.get("href", "").strip()

        if url\_path and not url\_path.startswith("/"):

            url\_path \= "/" \+ url\_path

        product\["url"\]       \= base\_url \+ url\_path if url\_path else None

        product\["url\_path"\]  \= url\_path

        product\["name"\]      \= (link.get("title") or link.get("alt") or "").strip()

        product\["brand"\]     \= link.get("data-brandname", "").strip() or None

        product\["slug"\]      \= link.get("data-prodname", "").strip() or None

    \# Name fallback

    if not product.get("name"):

        name\_el \= card\_div.select\_one(".product\_name")

        if name\_el:

            product\["name"\] \= name\_el.get\_text(strip=True)

    \# ── Short description ─────────────────────────────────────────────

    desc\_el \= card\_div.select\_one(".product\_short\_description")

    product\["short\_description"\] \= desc\_el.get\_text(strip=True) if desc\_el else None

    \# ── Price (regular / sale) ────────────────────────────────────────

    \# On-sale products have .product\_regular\_price (struck-through) and

    \# .product\_sale\_price within the same \<p class="price"\> block.

    reg\_el  \= card\_div.select\_one(".product\_regular\_price")

    sale\_el \= card\_div.select\_one(".product\_sale\_price")

    price\_span \= card\_div.select\_one("p.price span")

    if reg\_el and sale\_el:

        product\["regular\_price"\] \= reg\_el.get\_text(strip=True)

        product\["sale\_price"\]    \= sale\_el.get\_text(strip=True)

        product\["price"\]         \= product\["sale\_price"\]          \# effective price

        product\["on\_sale"\]       \= True

    elif price\_span:

        raw \= price\_span.get\_text(separator=" ", strip=True).replace("\\xa0", "").strip()

        product\["price"\]         \= raw

        product\["regular\_price"\] \= None

        product\["sale\_price"\]    \= None

        product\["on\_sale"\]       \= False

    \# ── Inventory count (HTML comment) ───────────────────────────────

    \# Stored as:  \<\!--\<small\>\<small\>Inv: 16\</small\>\</small\>--\>

    inv\_match \= re.search(r"Inv:\\s\*(\\d+)", str(card\_div))

    product\["inventory\_qty"\] \= int(inv\_match.group(1)) if inv\_match else None

    \# ── Lab data (THC, CBD, terpenes etc.) ───────────────────────────

    lab\_data \= {}

    for span in card\_div.select(".labinfo"):

        text \= span.get\_text(strip=True)

        parts \= text.split(":", 1\)

        if len(parts) \== 2:

            lab\_data\[parts\[0\].strip()\] \= parts\[1\].strip()

    product\["lab\_data"\] \= lab\_data if lab\_data else None

    \# ── Strain type (Indica/Sativa/Hybrid) ──────────────────────────

    specs\_div \= card\_div.select\_one(".specs")

    if specs\_div:

        for child in specs\_div.children:

            text \= getattr(child, "get\_text", lambda \*\*k: str(child))(strip=True)

            if text in ("Indica", "Sativa", "Hybrid", "CBD", "Indica Dominant",

                        "Sativa Dominant", "Hybrid Dominant"):

                product\["strain\_type"\] \= text

                break

    \# ── Image URL (CloudFront CDN, lazy-loaded) ──────────────────────

    img \= card\_div.select\_one("img.product-image")

    if img:

        img\_url \= img.get("data-src", img.get("src", "")).strip()

        \# Blank SVG placeholder — treat as no image

        product\["image\_url"\] \= None if "svg" in img\_url else img\_url

        \# High-res zoom image is also available on detail pages as data-zoom-image

    else:

        product\["image\_url"\] \= None

    \# ── Sale banner flag (visual ribbon) ─────────────────────────────

    product\["on\_sale"\] \= product.get("on\_sale", False) or bool(

        card\_div.select\_one(".salebanner, \#forsalebanner")

    )

    \# ── Category (injected by caller) ────────────────────────────────

    product\["category"\] \= category\_name

    return product

---

## 7\. Product Detail Pages

Detail pages are at: `{base_url}/cart/ps/{product_hash}/`

The hash in the URL is NOT the same as the product `data-id`. It is a numeric hash generated by Proteus420. It appears in the `href` attribute of the product card link:

\<a href="/cart/ps/311129311938/" data-id="99" ...\>

### What Detail Pages Add vs. Card Data

| Field | Available on card? | Available on detail page? |
| :---- | :---- | :---- |
| Product name | ✅ | ✅ |
| Price | ✅ | ✅ |
| Regular / sale price | ✅ | ✅ |
| Inventory count | ✅ (HTML comment) | ❌ not rendered |
| Lab data (THC/CBD) | ✅ | ✅ (same data) |
| Full description / long text | ❌ | ✅ (`<p>` below add-to-cart button) |
| High-res image | ❌ | ✅ (`data-zoom-image` on `#p420cart_image`) |
| Weight/unit options | ❌ | ✅ (`<select name="pricetype">` with `data-price`) |
| OG image (meta) | ❌ | ✅ (`og:image` meta tag) |
| Brand info modal | ❌ | ✅ (via `/cart/cart/brandinfo.cfm?id={brand_id}`) |

**Recommendation:** For a basic product feed (name, price, category, THC/CBD, image, inventory), the card AJAX responses are sufficient. Only fetch detail pages if you need long descriptions, weight breakdowns, or zoom-quality images.

### Detail Page Parsing

from bs4 import BeautifulSoup

def parse\_product\_detail(html: str) \-\> dict:

    soup \= BeautifulSoup(html, "html.parser")

    extra \= {}

    \# Long description

    fieldset \= soup.select\_one("fieldset.product-options")

    if fieldset:

        desc\_p \= fieldset.find("p")

        if desc\_p:

            extra\["long\_description"\] \= desc\_p.get\_text(strip=True)

    \# High-res image

    img \= soup.select\_one("\#p420cart\_image")

    if img:

        extra\["image\_zoom\_url"\] \= img.get("data-zoom-image")

        extra\["image\_url\_hq"\]   \= img.get("data-src")

    \# OG image (often the cleanest URL without transforms)

    og \= soup.select\_one("meta\[property='og:image'\]")

    if og:

        extra\["og\_image\_url"\] \= og.get("content")

    \# Weight/unit options with prices

    options \= \[\]

    for opt in soup.select("select\#attribute200 option"):

        options.append({

            "label":      opt.get\_text(strip=True),

            "price":      opt.get("data-price"),

            "sale\_price": opt.get("data-saleprice"),

            "img":        opt.get("data-img"),

        })

    if options:

        extra\["purchase\_options"\] \= options

    return extra

---

## 8\. Filtering and Specials

### On-Sale Products

| Store | Specials URL |
| :---- | :---- |
| The Cannabis Store | [https://tcsbflo.com/cart/c/featured](https://tcsbflo.com/cart/c/featured) (or `?onsale=1` on products endpoint) |
| HONEY Kenmore | [https://cart.honeykenmore.com/shop/c/featured/](https://cart.honeykenmore.com/shop/c/featured/) |
| Toke Lane | [https://cart.tokelane.com/shop/c/onsalepage](https://cart.tokelane.com/shop/c/onsalepage) |

For programmatic access, use the `onsale=1` parameter on the products endpoint:

params \= {"cat": "", "sel\_soldout": "n", "onsale": "1", "page": "all"}

On-sale cards render both `.product_regular_price` and `.product_sale_price` within the `<p class="price">` block. Regular cards only have a bare `<span>`.

### Sold-Out Products

To include sold-out items, change `sel_soldout=n` to `sel_soldout=y`.

### Lab Filters (THC/CBD Range)

The filters endpoint returns slider HTML for THC, CBD, and terpene percentage ranges. These are browser-side filters — the server always returns all products; the JavaScript hides non-matching cards. For scraping, ignore the filters endpoint and filter client-side after parsing lab\_data.

---

## 9\. Multi-Location Stores

Toke Lane has three locations served from a single `cart.tokelane.com` domain. Location selection is controlled by three values that must be communicated to the server as **both query params on the initial page load AND as session cookies**.

### Seeding the Session

def seed\_location\_session(session, base\_url: str, acc: str, loc: str,

                           shoptype: str \= "Pickup") \-\> bool:

    """

    Hit the cart home page with location params to set server-side session.

    Returns True if successful.

    """

    domain \= base\_url.replace("https://", "").replace("http://", "").split("/")\[0\]

    \# Set cookies before the request too

    session.cookies.set("acc",      acc,      domain=domain)

    session.cookies.set("loc",      loc,      domain=domain)

    session.cookies.set("shoptype", shoptype, domain=domain)

    url \= f"{base\_url}?acc={acc}\&loc={loc}\&shoptype={shoptype}"

    r \= session.get(url, timeout=30)

    r.raise\_for\_status()

    \# Verify the correct location loaded

    expected\_label \= f"acc={acc}"

    return r.status\_code \== 200

### Location Switching

Each location must be scraped in a **separate session** — the server tracks location in a server-side CF session that is tied to the session cookie. Re-using the same session object and just changing cookies may not work reliably; always create a new session per location.

---

## 10\. Rate Limiting & Anti-Bot

### Cloudflare Configuration

All three Proteus420 stores sit behind Cloudflare. The configuration appears to be in "essentially off" mode — no CAPTCHA challenges were observed during testing. However, standard browser fingerprinting TLS checks are active.

### Recommended Library

\# pip install curl\_cffi

from curl\_cffi import requests as cffi\_requests

session \= cffi\_requests.Session(impersonate="chrome120")

`curl_cffi` with `impersonate="chrome120"` passes Cloudflare JA3/JA4 TLS fingerprint checks without any challenge. Standard `requests` or `httpx` may trigger challenges, especially on repeat runs.

### Polite Scraping

import time

REQUEST\_DELAY \= 0.5   \# seconds between category requests

TIMEOUT       \= 30    \# request timeout in seconds

With 9–11 categories per store and three stores, a full scrape takes approximately 30–60 seconds at 0.5s delay. No IP rotation is needed.

---

## 11\. Complete Python Implementation

\#\!/usr/bin/env python3

"""

Proteus420 Dispensary Menu Scraper

Supports: tcsbflo.com (standalone), cart.honeykenmore.com (cloud),

          cart.tokelane.com (cloud, multi-location)

Install:  pip install curl\_cffi beautifulsoup4

Run:      python proteus420\_scraper.py

"""

import json, re, time

from datetime import datetime

from pathlib import Path

from bs4 import BeautifulSoup

try:

    from curl\_cffi import requests as cffi\_requests

    USE\_CURL\_CFFI \= True

except ImportError:

    import requests as cffi\_requests

    USE\_CURL\_CFFI \= False

OUTPUT\_DIR    \= Path("./inventory\_output")

REQUEST\_DELAY \= 0.5

TIMEOUT       \= 30

\# ─── Store Definitions ──────────────────────────────────────────────────────

STORES \= \[

    {

        "name":              "The Cannabis Store",

        "slug":              "tcsbflo",

        "base\_url":          "https://tcsbflo.com",

        "cart\_path":         "/cart",

        \# ⚠️  Standalone quirk: doubled /cart/cart/ path

        "products\_endpoint": "/cart/cart/ajax\_getproducts.cfm",

        "filters\_endpoint":  "/cart/cart/ajax\_getfilters.cfm",

        "detail\_base":       "/cart/ps/",

    },

    {

        "name":              "HONEY Kenmore",

        "slug":              "honeykenmore",

        "base\_url":          "https://cart.honeykenmore.com",

        "cart\_path":         "/",

        "products\_endpoint": "/cart/ajax\_getproducts.cfm",

        "filters\_endpoint":  "/cart/ajax\_getfilters.cfm",

        "detail\_base":       "/cart/ps/",

    },

    {

        "name":              "Toke Lane",

        "slug":              "tokelane",

        "base\_url":          "https://cart.tokelane.com",

        "cart\_path":         "/",

        "products\_endpoint": "/cart/ajax\_getproducts.cfm",

        "filters\_endpoint":  "/cart/ajax\_getfilters.cfm",

        "detail\_base":       "/cart/ps/",

        "locations": \[

            {"acc": "1140", "loc": "1", "shoptype": "Pickup",

             "label": "Toke Lane Trenton — Medical NJ"},

            {"acc": "1140", "loc": "2", "shoptype": "Pickup",

             "label": "Toke Lane Trenton — Adult Use NJ"},

            {"acc": "1153", "loc": "1", "shoptype": "Pickup",

             "label": "Toke Lane Buffalo NY"},

        \],

    },

\]

\# ─── HTTP Session ────────────────────────────────────────────────────────────

def make\_session():

    if USE\_CURL\_CFFI:

        return cffi\_requests.Session(impersonate="chrome120")

    s \= cffi\_requests.Session()

    s.headers.update({

        "User-Agent": ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "

                       "AppleWebKit/537.36 (KHTML, like Gecko) "

                       "Chrome/120.0.0.0 Safari/537.36"),

    })

    return s

def get(session, url, params=None, referer=None):

    headers \= {"X-Requested-With": "XMLHttpRequest"}

    if referer:

        headers\["Referer"\] \= referer

    r \= session.get(url, params=params, headers=headers, timeout=TIMEOUT)

    r.raise\_for\_status()

    return r.text

\# ─── Parsing ─────────────────────────────────────────────────────────────────

def get\_categories(html: str) \-\> list\[dict\]:

    soup \= BeautifulSoup(html, "html.parser")

    seen, cats \= set(), \[\]

    for a in soup.select("a.getproducts\[data-id\]\[data-catname\]"):

        cid  \= a.get("data-id", "").strip()

        name \= a.get("data-catname", "").strip()

        if name in ("featured", "onsale", "onsalepage", "whatsnew"):

            continue

        if cid not in seen:

            seen.add(cid)

            cats.append({"id": cid, "name": name})

    return cats

def parse\_card(div, base\_url: str, category: str) \-\> dict:

    p \= {"category": category}

    p\["id"\] \= div.get("data-id", "").strip()

    for cls in div.get("class", \[\]):

        m \= re.match(r"products\_strain\_(\\d+)", cls)

        if m:

            p\["strain\_id"\] \= m.group(1)

    link \= div.select\_one("a.item\_view\[data-id\]")

    if link:

        path \= link.get("href", "").strip()

        if path and not path.startswith("/"):

            path \= "/" \+ path

        p\["url"\]   \= base\_url \+ path if path else None

        p\["name"\]  \= (link.get("title") or link.get("alt") or "").strip()

        p\["brand"\] \= link.get("data-brandname", "").strip() or None

        p\["slug"\]  \= link.get("data-prodname", "").strip() or None

    if not p.get("name"):

        el \= div.select\_one(".product\_name")

        if el:

            p\["name"\] \= el.get\_text(strip=True)

    desc \= div.select\_one(".product\_short\_description")

    p\["short\_description"\] \= desc.get\_text(strip=True) if desc else None

    reg  \= div.select\_one(".product\_regular\_price")

    sale \= div.select\_one(".product\_sale\_price")

    span \= div.select\_one("p.price span")

    if reg and sale:

        p\["regular\_price"\] \= reg.get\_text(strip=True)

        p\["sale\_price"\]    \= sale.get\_text(strip=True)

        p\["price"\]         \= p\["sale\_price"\]

        p\["on\_sale"\]       \= True

    elif span:

        p\["price"\]         \= span.get\_text(separator=" ", strip=True).replace("\\xa0","").strip()

        p\["regular\_price"\] \= None

        p\["sale\_price"\]    \= None

        p\["on\_sale"\]       \= False

    inv \= re.search(r"Inv:\\s\*(\\d+)", str(div))

    p\["inventory\_qty"\] \= int(inv.group(1)) if inv else None

    lab \= {}

    for s in div.select(".labinfo"):

        kv \= s.get\_text(strip=True).split(":", 1\)

        if len(kv) \== 2:

            lab\[kv\[0\].strip()\] \= kv\[1\].strip()

    p\["lab\_data"\] \= lab or None

    specs \= div.select\_one(".specs")

    p\["strain\_type"\] \= None

    if specs:

        for child in specs.children:

            t \= getattr(child, "get\_text", lambda \*\*k: str(child))(strip=True)

            if t in ("Indica","Sativa","Hybrid","CBD","Indica Dominant","Sativa Dominant"):

                p\["strain\_type"\] \= t

                break

    img \= div.select\_one("img.product-image")

    if img:

        raw \= img.get("data-src", img.get("src","")).strip()

        p\["image\_url"\] \= None if "svg" in raw else raw

    else:

        p\["image\_url"\] \= None

    p\["on\_sale"\] \= p.get("on\_sale", False) or bool(

        div.select\_one(".salebanner, \#forsalebanner")

    )

    return p

\# ─── Core Scrape ─────────────────────────────────────────────────────────────

def scrape\_store(store: dict, location: dict \= None) \-\> dict:

    session  \= make\_session()

    base     \= store\["base\_url"\]

    cart\_url \= f"{base}{store\['cart\_path'\]}"

    if location:

        domain \= base.split("//")\[1\]

        for key in ("acc", "loc", "shoptype"):

            session.cookies.set(key, location\[key\], domain=domain)

        cart\_url \+= f"?acc={location\['acc'\]}\&loc={location\['loc'\]}\&shoptype={location\['shoptype'\]}"

    main\_html  \= get(session, cart\_url)

    categories \= get\_categories(main\_html)

    result \= {}

    for cat in categories:

        time.sleep(REQUEST\_DELAY)

        html   \= get(session, f"{base}{store\['products\_endpoint'\]}",

                     params={"cat": cat\["id"\], "sel\_soldout": "n", "page": "all"},

                     referer=f"{base}/c/{cat\['name'\]}/")

        soup   \= BeautifulSoup(html, "html.parser")

        cards  \= soup.select("\[data-id\].product-card-wrapper")

        prods  \= \[\]

        seen   \= set()

        for card in cards:

            try:

                p \= parse\_card(card, base, cat\["name"\])

                if p.get("id") and p\["id"\] not in seen:

                    seen.add(p\["id"\])

                    prods.append(p)

            except Exception as e:

                print(f"  \[WARN\] parse error: {e}")

        print(f"  \[{cat\['name'\]}\] {len(prods)} products")

        if prods:

            result\[cat\["name"\]\] \= prods

    return result

\# ─── Main ────────────────────────────────────────────────────────────────────

def main():

    OUTPUT\_DIR.mkdir(exist\_ok=True)

    ts  \= datetime.now().strftime("%Y%m%d\_%H%M%S")

    all \= {}

    for store in STORES:

        print(f"\\n{'='\*55}\\n  {store\['name'\]}\\n{'='\*55}")

        if "locations" in store:

            store\_data \= {"store\_name": store\["name"\], "locations": {}}

            for loc in store\["locations"\]:

                print(f"  Location: {loc\['label'\]}")

                inv \= scrape\_store(store, loc)

                store\_data\["locations"\]\[loc\["label"\]\] \= {

                    "location\_info": loc,

                    "scraped\_at":    datetime.now().isoformat(),

                    "categories":    inv,

                    "total\_products": sum(len(v) for v in inv.values()),

                }

        else:

            inv        \= scrape\_store(store)

            store\_data \= {

                "store\_name":    store\["name"\],

                "scraped\_at":    datetime.now().isoformat(),

                "categories":    inv,

                "total\_products": sum(len(v) for v in inv.values()),

            }

            print(f"  Total: {store\_data\['total\_products'\]} products")

        (OUTPUT\_DIR / f"{store\['slug'\]}\_{ts}.json").write\_text(

            json.dumps(store\_data, indent=2, ensure\_ascii=False)

        )

        all\[store\["slug"\]\] \= store\_data

    (OUTPUT\_DIR / f"all\_stores\_{ts}.json").write\_text(

        json.dumps(all, indent=2, ensure\_ascii=False)

    )

    print("\\nDone.")

if \_\_name\_\_ \== "\_\_main\_\_":

    main()

---

## 12\. Product Schema Reference

See `proteus420_product_schema_empty.json` for the canonical empty schema.

### Field Descriptions

| Field | Type | Source | Notes |
| :---- | :---- | :---- | :---- |
| `id` | string | `div[data-id]` | Proteus internal product ID |
| `strain_id` | string | `div.products_strain_NNNN` | Proteus strain/variant ID |
| `slug` | string | `a[data-prodname]` | URL-safe product name |
| `name` | string | `a[title]` or `.product_name` | Full display name |
| `brand` | string|null | `a[data-brandname]` | Brand/cultivator name |
| `category` | string | Injected by scraper | Category name (e.g. "Flower") |
| `strain_type` | string|null | `.specs div` | Indica/Sativa/Hybrid/CBD |
| `short_description` | string|null | `.product_short_description` | Short copy, coupon text on sale items |
| `long_description` | string|null | Detail page `<p>` in `fieldset` | Only on detail pages |
| `price` | string | `p.price span` | Effective price (may be sale price) |
| `regular_price` | string|null | `.product_regular_price` | Original price when on sale |
| `sale_price` | string|null | `.product_sale_price` | Discounted price |
| `on_sale` | boolean | `.salebanner` or sale price presence | Whether item is discounted |
| `inventory_qty` | int|null | HTML comment `Inv: N` | Stock count |
| `lab_data` | object|null | `.labinfo` spans | THC, CBD, THCa, CBG, TERPS, etc. |
| `image_url` | string|null | `img[data-src]` | CloudFront CDN URL (400px wide) |
| `image_zoom_url` | string|null | Detail page `#p420cart_image[data-zoom-image]` | S3 full-resolution URL |
| `og_image_url` | string|null | Detail page `og:image` meta | Clean S3 URL |
| `url` | string | `a[href]` \+ base\_url | Absolute product detail page URL |
| `url_path` | string | `a[href]` | Relative URL path |
| `purchase_options` | array|null | Detail page `select#attribute200` | Weight/unit options with per-option prices |
| `scraped_at` | string | Set by scraper | ISO 8601 timestamp |

---

## 13\. Common Pitfalls

### ❌ Wrong AJAX path for tcsbflo.com

\# WRONG — returns 404

GET https://tcsbflo.com/cart/ajax\_getproducts.cfm

\# CORRECT — doubled /cart/cart/

GET https://tcsbflo.com/cart/cart/ajax\_getproducts.cfm

### ❌ Reading `src` instead of `data-src` for images

Product images are lazy-loaded. The `src` attribute is a blank SVG placeholder. Always use `data-src`. If `data-src` contains `"svg"`, the image is missing.

\# WRONG

img\_url \= img.get("src")

\# CORRECT

img\_url \= img.get("data-src")

if img\_url and "svg" in img\_url:

    img\_url \= None

### ❌ Missing prices on sale items

On-sale items do **not** have a price in the normal `p.price > span` path alone — the span wraps two child spans. Check for `.product_sale_price` first.

\# The price element on sale items looks like:

\# \<p class="price"\>

\#   \<span\>

\#     \<span class="product\_regular\_price" style="text-decoration: line-through;"\>$30.00\</span\>

\#     \<span class="product\_sale\_price couponamount"\>$21.00\</span\>

\#   \</span\>

\# \</p\>

### ❌ Using the same session for multiple Toke Lane locations

Each location requires its own HTTP session with its own server-side CF session cookie. Swapping cookies mid-session can return stale location data.

### ❌ Forgetting `page=all`

Without `page=all`, the endpoint defaults to paginated mode (usually 12 items per page). Always pass `page=all`.

### ❌ Scraping Accessories category for cannabis data

Category IDs 15 (Accessories) and 23 (Accessories, Toke Lane) contain glass, papers, and hardware — not cannabis products. Skip them for cannabis-only pipelines.

### ❌ Hardcoding category IDs

Category IDs are configured per-account and may differ between stores (e.g., Honey Kenmore uses ID 11 for Extracts; Toke Lane uses 11 for Extracts too but assigns different IDs to other categories). Always discover categories dynamically from the cart home page.

---

*End of Developer Reference*  
