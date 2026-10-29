---
name: salla-stape-tracking
description: Server-side tracking expertise for Salla (سلة) stores using Stape.io-hosted sGTM. Covers the Stape Conversion Tracking app for Salla, custom loader, cookie keeper, Stape User ID, webhooks, and sending conversions server-side to GA4, Google Ads, Meta CAPI, TikTok Events API, Snapchat CAPI, and Mixpanel. Also covers event deduplication, match-quality debugging, and Consent Mode v2. USE THIS SKILL whenever the user mentions Salla, سلة, Stape, stape.io, sGTM, server-side GTM, server container, CAPI, Conversion API, event deduplication, event_id, fbp/fbc, transport URL, custom loader, cookie keeper, or asks why conversions are missing/duplicated/mismatched between the ad platform and the store — even if they don't name the platform explicitly. Also trigger on Arabic phrasings like "سيرفر سايد تراكينج", "تتبع من السيرفر", "البيكسل مش بيقرا", "التحويلات مكررة", "متجر سلة", "ربط الكونفيرجن ايه بي اي".
---

# Salla + Stape Server-Side Tracking

> Built and maintained by **Mitrone (ميترون)** — e-commerce growth & tracking specialists. Distilled from live server-side tracking builds on Salla stores. Last updated: October 2026.
> Website: https://mitrone.com · WhatsApp: +20 112 115 0270 (https://wa.me/201121150270)

Expertise for building, auditing and debugging server-side tracking on Salla stores through Stape-hosted sGTM, feeding GA4, Google Ads, Meta, TikTok, Snapchat and Mixpanel.

Everything marked **verified** was observed live on production Salla stores (Jul–Oct 2026). Salla, the Stape app and platform UIs change — re-verify on each store before building on a detail.

---

## Prerequisites (check first)

This skill assumes the store already has both of the following. Confirm them before giving build, audit or debugging steps:

1. An active **Stape.io sGTM container** (Stape's sGTM container hosting) on a first-party subdomain of the store's domain.
2. The **Stape Conversion Tracking app for Salla** installed on the store and linked to that container.

If either is missing, say so first and point the user to set it up (§2). Do not build workarounds that skip Stape, such as browser-only pixels or Salla's native conversion API apps, as a substitute.

---

## 0. Working rules (read first)

- Show the user every change before publishing, and let the user (or the container owner) publish GTM versions.
- Never read, repeat, store or type CAPI access tokens (Meta, TikTok, Snap). The user pastes tokens into GTM. Prefer a Constant variable for tokens so tags can be edited without exposing them. Do not export a container JSON that contains tokens.
- Never type passwords/OTP, never create accounts, undo any cart/wishlist test you do, and never print customer PII (emails/phones) from logs or webhooks — report field **names and formats** only.
- **Before fixing any diagnostic, prove it comes from your tracking and not from Salla's legacy integrations.** Platform dashboards mix both sources for days/weeks after Salla is switched off (see §9).
- Do the research first, build in a fresh GTM workspace, test in Preview, then hand over a change list. Record what was built and decided in a project reference file (per store, outside this skill).
- Be explicit about verified vs inferred. When an answer depends on a current UI detail or API version, check the docs (e.g. `stape.io/helpdesk/documentation/integrations/salla-app`) rather than asserting from memory.
- Assume an experienced practitioner: give procedural specifics, not conceptual intros.
- A tag firing is not a tag succeeding — always open the outgoing request/response, and prefer the platform's own surface (Meta Events Manager / Test Events, TikTok Events Manager, Snap Events Manager, GA4 DebugView) over guessing from GTM.

---

## 1. Core mental model

Salla is closed SaaS: no theme-level code, no checkout DOM, no arbitrary server code. Realistic data sources:

1. **Stape Conversion Tracking app for Salla** — injects web GTM (with custom loader), pushes a dataLayer, sends order webhooks. The foundation for any serious setup.
2. **Salla webhooks** → `https://<subdomain>/data` — server-to-server, the ground truth for orders and customer data.
3. Salla's native integrations (pixels, "conversion API" apps, Google Ads app) — the main source of **duplicates**; usually switched off once your stack is live.
4. Salla Merchant API — backfill/offline.

Two architectures have worked:

```
A) Stock Stape template
   web GTM: [Stape] tags + Data Tag (stapecdn dtag) ──► Data Client (/data) ──► dc - * triggers
             + gtag with server_container_url     ──► GA4 Client          ──► ga4 - * triggers

B) GA4-client-centric custom build — recommended for new builds
   web GTM: Google tag (server_container_url) + GA4 event tags built from *_stape events
            + browser pixels (Meta/TikTok/Snap) sharing one event_id
        ──► GA4 Client ──► GA4 · Google Ads · Meta CAPI · TikTok Events API · Snap CAPI
   Salla webhook ──► Data Client (/data) ──► Stape Data Store (enrichment, §7)
```

In B every server tag triggers on `Client Name = GA4` + an event-name regex, and the GA4 event params (items, value, user_data, event_id, user_id) feed all platforms. One event_id generated in the web container dedups every pixel against its CAPI.

---

## 2. Setup workflow

1. **Server container + first-party subdomain** (e.g. `sgtm.example.com`, CNAME to Stape). Not `stape.io`. `https://`, no trailing slash. Check the subdomain also has an AAAA record (Stape gives one) so IPv6 users reach it (Meta "Update to IPv6" recommendation).
2. **Stape app on Salla** (marketplace, 7-day trial):
   - General: Insert GTM snippet, web container ID, custom domain, **Custom Loader** (Stape container identifier), **Cookie Keeper + Stape User ID** power-ups (both). Leave "Same Origin Path" empty — Salla cannot proxy paths. Copy the generated tagging-server URL with the copy button; if it introduces a new subdomain (e.g. `load.<sub>`), confirm it resolves before saving or the whole web container stops loading.
   - Data Layer: enable ecommerce events + user data. Decide **Decorate Data Layer event name** (`_stape` suffix) before building triggers; turn it ON when Salla or another app also pushes events.
   - Webhooks: send to `https://<subdomain>/data`, purchase (+ refund).
3. **Confirm the container loads from the subdomain** (DevTools → Network → filter by subdomain → 200) (`<sub>/<id>.js` or a `/2d4<id>` style path, not googletagmanager.com) — otherwise the custom loader is off.
4. **Inventory Salla's native integrations before building** (§9) and plan the switch-off right before publishing.

⚠️ `stape-io/salla-gtm-container-templates` on GitHub is deprecated (moved to Stape's GTM Setup Assistant). Don't import it.

---

## 3. Salla dataLayer — verified facts

### Two parallel event families
- Salla native events — UA-style ecommerce (`ecommerce.detail.products`, `ecommerce.purchase.actionField/products`), names like `Product Viewed`, `addToCart`, `Product Added`, `purchase`, `Order Completed`, `page.view`.
- Stape-decorated events (decoration ON) — GA4-style `ecommerce.items[]`: `view_item_stape`, `add_to_cart_stape`, `view_cart_stape`, `remove_from_cart_stape`, `purchase_stape`, `view_collection_stape`. **Build on these** where they exist. (On a stock install with decoration off there was no `items[]` anywhere — check the actual store.)

### Event map (decoration ON)
| Step | Source event | Notes |
|---|---|---|
| Page view | Google tag on Initialization | No Stape page event. `page.view` also fires on API calls (`route api.cart.items.store`) — never use it |
| View item | `view_item_stape` | `Product Viewed` rounds value to an integer |
| Add to cart | `add_to_cart_stape` **corrected** (below) | `Product Added` / `addToCart` buggy |
| View cart | `view_cart_stape` | has `cart_state.lines[]` |
| Begin checkout | `cart::submitted` (cart page "complete order" button) | No `begin_checkout_stape`. No items/value in payload → take them from the latest `Cart Updated` or `cart_state`; category from `view_cart_stape` items by id. GTM trigger names cannot contain `:` |
| Purchase | `purchase_stape` on `/ar/thankyou/...` | `transaction_id` = order number; also tax, shipping, sub_total, discount_amount, coupon, user_data |
| Login / sign-up | **no dedicated event**; `customer.isGuest` flips to false after OTP reload. Salla emits `auth::*` events inconsistently; OTP registration usually looks like login. Capture with a logger before relying on names |
| Wishlist | `wishlist::added` (param = product id) | product data not in dataLayer on search/category pages → read `custom-salla-product-card` / `salla-product-card` element `.product` (id, name, price, category.name) |
| Search | page_view on `/search?q=` (GA4 enhanced measurement `view_search_results`) | no Salla search event |
| Checkout page | `<store>/ar/checkout/...` **does not load your web container** — only Salla's GTM. AddPaymentInfo/AddShippingInfo are impossible from your stack |

Ignore: `services::*`, `twilight::*`, `languages::*`, `currency::*`, `auth::token.invalid`, `product::fetch.*`, `salla-products-slider::*`, `recommendations::*`, `order::order.created` / `Product Reordered` (fire on every page, not real orders), `document::request.failed`.

### Data bugs to correct in GTM
- **Add to cart**: first add can send a **gift-card add-on** (`أهدي لشخص آخر`, price 0) instead of the product; `quantity` = whole cart line quantity, not the added amount; `item_category` sometimes an object `{id,name}`; `ecommerce.value` null. Fix: product from `Product Added`; quantity from `cart::before.add.item.payload.data.quantity` (fallback: diff of `cart_state.lines` only when the previous state really has lines, else 1); category → `.name`; value = price × qty. Quantity edits on the cart page emit `Cart Updated`/`cart::item.updated` only — no false add_to_cart.
- **Zero-price items** (notes, gift cards, add-ons) appear in items — drop them in every event.
- `item_id` arrives as number → cast to string; it equals the Merchant Center offerId / Salla numeric product id (MC product id `ar~SA~<id>`).
- `user_data.first_name` holds the **full name** → split into first/last. Phone is missing from user_data outside purchase → take `customer.phone_with_plus` (E.164) from the page push / `twilight::init.user`. Derive `country = SA` when phone starts `+966`. No city/zip/state outside purchase.
- Data-quality check on every store: `AxiosError` landing in `user_data.address.first_name` on non-purchase events (seen on a live store) — an identical junk value across users degrades EMQ.
- Gold/jewellery stores: prices change live (hourly), even within a session.

### event_id
Generate once in the web container and send it to the pixel **and** the GA4 hit (GA4 shows it as `evnid`): e.g. page seed + `_` + `gtm.uniqueEventId`, and the **order number on purchase**. Store the page_view id in a window var (e.g. `window.__pageEid`) and reuse it for events that inherit the config event_id (Search/`view_search_results`). On the stock Stape template the dedup key is the `{{Unique Event ID}}` variable, **not** `ecommerce.event_id` — check which variable feeds each tag.

---

## 4. Platform setup (server side)

### GA4
- Google tag with `server_container_url`; GA4 client claims `/g/collect`. GA4 batches several events in one request — read the event list in server Preview, not the URL.
- In the Google tag settings turn **OFF "Automatically detect user-provided data"** — on Salla it scraped the store's footer email into every visitor's user_data. Keep "Allow user-provided data capabilities" ON with code mode.
- Unwanted referrals: tabby, tamara, tap.company, moyasar, checkout.com, payfort (Referral domain contains), else payment gateways steal purchase attribution.
- Arabic `/ar/` paths: make sure path-based triggers/grouping cover them.

### Google Ads (server)
- Conversion tag fires **only when a click id exists** (`FPGCLAW`/`FPGCLGB`/`FPGCLAG` cookies set by the Conversion Linker on the server). No gclid/gbraid/wbraid → silent, not broken.
- Cart data: Product reporting → Custom configuration: Merchant ID, feed country, feed language, discount `{{ED - discount}}`; items from event. Outgoing request shows `mid`, `fcntr`, `flng`, `item=(price*qty*id**)`, `oid`, `dscnt`.
- ⚠️ **Leave Conversion Value empty** (inherits event `value`). Setting it explicitly makes Google send an encrypted `sg_ecv` instead of `value`.
- Enhanced conversions: user_data on the event; `em=tv.1` with nothing else means no user data.
- Run new sGTM conversion as Primary only after demoting Salla's old app conversion to Secondary — both Primary = double counting in bidding. Compare a week, then remove Salla's Google Ads app.
- "Unmatched sold products" (CWCD_ITEM_NOT_FOUND) = feed problem when your ids already match other feed items. Search the ids in Merchant Center before touching tracking.
- **Dynamic remarketing**: the sGTM Remarketing tag with "send dynamic remarketing event data" does not send product data from Salla events by default. Add custom parameters: `ecomm_prodid={{ED - items.0.item_id}}`, `ecomm_pagetype` (lookup: view_item→product, add_to_cart/view_cart/begin_checkout→cart, purchase→purchase, view_item_list→category, search→searchresults), `ecomm_totalvalue={{ED - value}}`, `google_business_vertical=retail`. Verify outgoing `viewthroughconversion` + `rmkt/collect` carry them. On architecture A also move the trigger from `ga4 - *` to `dc - *` (view_item/add_to_cart only reach the Data Client) without duplicating purchase.
- Remarketing showing "Failed" on user_engagement/scroll is harmless; restrict its trigger if you want clean Preview.
- **COGS / profit**: cart data can't carry cost, shipping or tax. COGS comes only from Merchant Center attribute `cost_of_goods_sold` (same id). Profit = Σ(price×qty) − discount − COGS. For gold (gram price × weight + making charge) COGS must update automatically (supplemental feed from a sheet/app); a static value goes stale within hours.
- UTMs: account-level Final URL suffix `utm_source=google&utm_medium=cpc&utm_campaign={_campaign}&utm_id={campaignid}&utm_content={adgroupid}&utm_term={keyword}` + custom param `_campaign` per campaign. Salla reads only UTMs (gclid alone shows "Using the CLICKID gad_source").

### Meta (Stape "Facebook Conversion API" template)
- Event name override via lookup: page_view→PageView, view_item→ViewContent, add_to_cart→AddToCart, begin_checkout→InitiateCheckout, purchase→Purchase, add_to_wishlist→AddToWishlist, sign_up→CompleteRegistration, login→LOGIN (custom), search/view_search_results→Search.
- Action source website, generate fbp, event enhancement, auto-map server/user/custom data, **Map item data to = Both Contents and Content IDs** (default Contents omits `content_ids`).
- Pixel in web GTM: `fbq init` with Advanced Matching (em, ph, fn, ln, external_id, country) and `eventID` = shared event_id. Purchase `order_id` = real order number.
- Remove `test_event_code` before publishing or all server events go to Test Events only.
- **fbc gap in FB/IG in-app browsers** (Meta says "server sending empty fbc"): the pixel uses an Extended-Browser-Properties click id without writing `_fbc`. Fix with Meta's param builder in web GTM (Custom HTML, Initialization):
  ```html
  <script>(function(){if(window.__pb)return;window.__pb=1;var s=document.createElement('script');s.async=true;
  s.src='https://cdn.jsdelivr.net/npm/meta-capi-param-builder-clientjs@1.3.3/dist/clientParamBuilder.bundle.js';
  s.onload=function(){try{window.clientParamBuilder.processAndCollectAllParams(location.href)}catch(e){}};
  document.head.appendChild(s)})();</script>
  ```
  It writes `_fbc`/`_fbp` on the root domain (90 days); the server reads them unchanged. Prove the gap first: in Stape outgoing, users with fbclid always get fbc on later events → server isn't dropping it.
- Domain allowlist (Business settings) must include the store domain; Meta's "Confirm domains" lists salla.sa / translate.goog — decision for the user.
- Wishlist from search/cart pages may lack product data — use product-card fallback.

### TikTok (Stape "TikTok Events API" template)
- Use **Purchase** (CompletePayment is legacy). Map: view_item→ViewContent, add_to_cart→AddToCart, begin_checkout→InitiateCheckout, purchase→Purchase, add_to_wishlist→AddToWishlist, sign_up→CompleteRegistration, search→Search. PageView browser-only by design.
- Web: ttq snippet + `ttq.identify` (email, E.164 phone, external_id — ttq hashes) + `ttq.track(name, params, {event_id})`; params `contents[{content_id, content_type, quantity, price, content_name, content_category}]`, value, currency, order_id, query.
- Server: Generate `_ttp`, event enhancement on, event_id from GA4 event_id.
- ViewContent server↔browser match ~68% is common; Purchase should be ~100% "Server & browser".

### Snapchat (Stape "Snapchat Conversion API" template, CAPI v3)
- **Event name = Custom lookup**, not Inherit — inherit doesn't know `view_search_results` or `login` (sent raw → 400 code 516). Map: PAGE_VIEW, VIEW_CONTENT, ADD_CART, START_CHECKOUT, PURCHASE, ADD_TO_WISHLIST, SEARCH, SIGN_UP, LOGIN.
- Pixel `client_dedup_id` = server `event_id` (pixel auto-generates `@-...` ids otherwise). Purchase also dedups on `transaction_id`.
- Required: event_time (ms), action_source WEB, event_source_url, ≥1 of em/ph/IP+UA. Strongly recommended: `sc_cookie1` (= `_scid`, pixel name `uuid_c1`), `sc_click_id` (ScCid → `_scclid`). Add firstname/lastname/geo_city to the pixel.
- item_ids / item_category are required for dynamic ads on PAGE_VIEW, VIEW_CONTENT, ADD_CART, PURCHASE (PAGE_VIEW without item_ids is a known minor gap).
- `num_items` server = number of lines, pixel = units — minor.
- Health check: **Purchase "Conversions V3 API" per day should equal GA4 transactions**.

### Mixpanel
Server-side only explicit events; pick one `distinct_id` forever; `$insert_id` = event_id; send event `time`. Don't send PII without need.

### Hashing / user data
SHA-256, lowercase, trimmed; phone E.164 before hashing (Snap `ph` = digits with country code, no `+`). Stape tags hash plaintext — never double-hash. Send the end user's IP/UA (Stape handles it; confirm). Match quality is mostly data availability: guests/Apple Pay carry no customer data in the browser → use webhooks (§7).

---

## 5. Purchase value definition
Salla's `value` = items (ex-VAT) + 15% VAT + shipping. Item prices in items[] are already ex-VAT and post-discount. Decide one definition with the user and apply it to every platform:
- Gross (order total) — default.
- **Net = value − tax − shipping** — built as a server variable template + Augment transformation on purchase (Client GA4 + Event purchase, all tags). VAT 15% on the net is 13.04% of the gross (15/115), so "minus 15%" ≠ multiply by 0.85; a flat percentage is wrong whenever shipping varies.

```js
// Variable template "Net purchase value" (server). Permission: read_event_data value, tax, shipping
const getEventData=require('getEventData'),getType=require('getType'),makeNumber=require('makeNumber'),Math=require('Math');
const toNum=(v)=>{const t=getType(v);if(t==='number')return v;if(t==='string'&&v.trim()!==''){const n=makeNumber(v);if(getType(n)==='number'&&n===n)return n;}return 0;};
const raw=getEventData('value');const rt=getType(raw);if(rt==='undefined'||rt==='null')return raw;
const value=toNum(raw);
const net=value-(data.subtractTax!==false?toNum(getEventData('tax')):0)-(data.subtractShipping!==false?toNum(getEventData('shipping')):0);
if(net<=0)return value;return Math.round(net*100)/100;
```
Template CHECKBOX params imported via .tpl may not render — default to "on" in code (`!== false`). GA4 revenue drops from the publish date; say so.

---

## 6. Deduplication rules
- Meta: same event_name + event_id (case matters).
- TikTok: same event_id.
- Snapchat: `client_dedup_id` (pixel) = `event_id` (CAPI); purchase also `transaction_id`.
- GA4: no dedup — one path per event.
- Google Ads: one conversion action per purchase; demote duplicates to Secondary.
- Generate the id once at the event. Two containers generating ids = no dedup.
- Salla native integrations use their own ids → they never dedup with yours. A Meta Purchase `order_id` like `EII1|AQAA…` (encrypted) and URL `/ar/account/orders/<code>` is Salla's CAPI; yours is the numeric order number.

---

## 7. Salla webhook enrichment (guest / Apple Pay purchases)

Problem (verified): ~20% of purchases (guests, Apple Pay express checkout — referrer `...?has_apple_pay=true`) reach the thank-you page with **no customer data**, so Meta/Snap/TikTok/Ads get only cookies/IP/UA. The order webhook has the data.

### Receiving webhooks
- Without a client, Salla POSTs to `/data` return **400** (Stape incoming logs show them, no body). Install Data Client (`stape-io/data-client`) → path `/data`.
- Webhooks never appear in server Preview (no debug header) — verify via Stape logs / Data Store.
- Payload (event `purchase_stape_webhook`): `timestamp`, `ip_override` (Salla's IP, not the customer's), `order_status`, `client_id` `dcid.*`, `unique_event_id`; `user_data{email_address, phone_number(+966…), first_name, last_name(may be null), city, country, customer_id}`; `ecommerce{transaction_id = Salla INTERNAL id, reference_id = ORDER NUMBER, items[item_id,item_sku,item_name,price,quantity], value, currency, tax, shipping, sub_total, discount_amount, coupon}`. **Join on `ecommerce.reference_id`** = browser `transaction_id`.
- Timing: webhook usually 5–15 s before the browser purchase, but not always (seen 33 s after). Enrichment only helps when it arrives first — platforms keep the first event per event_id.
- Existing triggers usually filter `Client Name = GA4`, so the Data Client fires nothing else — keep it that way (don't send webhook purchases as extra conversions unless designed with the same event_id).

### Build (server container)
1. Tag template **Stape Data Store Writer**; tag on trigger `Client Name = Data Client`: collection `salla_webhooks` (create it with TTL 720 h), Document ID `{{ED - ecommerce.reference_id}}`, Add Event Data **off**, Merge document keys **on**, Add Timestamp on, custom field `user_data = {{ED - user_data}}` (stores only what's needed).
2. Variable template **Stape Data Store Lookup**: Document ID `{{ED - transaction_id}}`, property `user_data`, cache on.
3. Custom variable template "Merge webhook user data" — fills only empty fields, never overwrites event values:
```js
const getEventData=require('getEventData'),getType=require('getType'),makeString=require('makeString');
const isEmpty=(v)=>{const t=getType(v);return t==='undefined'||t==='null'||(t==='string'&&v.trim()==='');};
const wh=getType(data.webhookUserData)==='object'?data.webhookUserData:{};
if(data.mode==='user_id'){const c=getEventData('user_id');if(!isEmpty(c))return c;if(!isEmpty(wh.customer_id))return makeString(wh.customer_id);return c;}
const cur=getEventData('user_data');const out={};if(getType(cur)==='object'){for(const k in cur)out[k]=cur[k];}
['email_address','phone_number','first_name','last_name','country','city'].forEach((f)=>{if(isEmpty(out[f])&&!isEmpty(wh[f]))out[f]=wh[f];});
const at=getType(out.address);if(at==='undefined'||at==='null'){const a={};['first_name','last_name','city','country'].forEach((f)=>{if(!isEmpty(out[f]))a[f]=out[f];});let h=false;for(const k in a)h=true;if(h)out.address=[a];}
return out;
```
   Permission: read_event_data keys `user_data`, `user_id`. Two variables: mode user_data and mode user_id.
4. Transformation **Augment event**: `user_data`, `user_id` = merge variables; conditions Client Name = GA4 AND Event Name = purchase; affected tags Meta, Snap, TikTok, Ads Purchase (not GA4).
5. Verify on the next guest order: Data Store doc id = order number; GA4 request (not augmented) has no `em` while Meta/Snap/TikTok do. A logged-in buyer proves nothing (GA4 also has em).

Stape template mappings: Meta/Snap/TikTok read `user_data.email_address/phone_number/first_name/last_name/country`, `user_data.address[0].*`, external_id from `user_id`/`external_id`. Augment supports dot paths; order Allow → Augment → Exclude.

---

## 8. Stape operations

- **Quota counts incoming requests only** (gtm.js/gtag loads, `/_/set_cookie`, collect, service worker). ~60% of GA4 incoming can be `/_/set_cookie` (service-worker transport `sst.sw_exp=1`). Switching GA4 client to JavaScript-managed cookies and limiting the Conversion Linker did **not** reduce it — don't promise savings. ~70k pageviews/day ≈ 4M+/month → Business plan territory. Auto-upgrade raises the bill silently; flag before renewal.
- **Outgoing logs**: URL params alone load nothing — open Date range and click **Apply**. The calendar can refuse "today" near UTC midnight. Logs lag minutes to ~1 h; the page shows "logs cover X% of range". Many events appear as **Other** (Google Ads conversions, Snap VIEW_CONTENT, ccm/collect) — filter by **Platform**. Event type "View Item" filter is broken → use Incoming + Associated logs. A trace page shows all platforms for one event. "Download as CSV" includes Request-Body (can be read in-page by intercepting `URL.createObjectURL`). Outgoing logging costs quota — agree with the user before enabling.
- Server Preview: opening the debug page sets a cookie on the subdomain, so the same browser's hits route to Preview even without web Preview. A Preview opened after the page loaded misses that page's hits.
- Snap Events Manager hourly data: React fiber on `.recharts-wrapper` → `memoizedProps.data` (keys "Conversions API", "Conversions V3 API", "Pixel SDK").
- GTM UI: importing a template = Template editor ⋮ → Import; in automation, capture the hidden `.tpl` file input and set a File via DataTransfer. CodeMirror instances expose `.CodeMirror.getValue/setValue`. Multi-select pickers need "Add". Escape in the template editor triggers "unsaved changes".

---

## 9. Salla native integrations — find, switch off, verify

Where duplicates come from:
- `services::facebook_pixel.init`, `services::snapchat_pixel.init`, `services::tiktok_pixel.init`, `twitter_pixel`, `adwords`, `hotjar` in the dataLayer / `twilight::init` HTML; Salla's own GTM container (a second GTM-XXXX on the page, not yours) runs the pixel code (Custom HTML with `snaptr`, `fbq`).
- Apps at `s.salla.sa/apps`: "Facebook Pixel", "Facebook - Instagram Catalog", "تتبع التحويلات الذكي (Conversion API) — سم للتسويق" (per-platform chips — remove each platform), Google Ads app, Snap integration under Marketing → event manager.
- Snap: "Partner integration → Salla" and "Update your Salla integration" recommendation.

Switch-off order: build + test with Salla still on → remove the Salla integration right before publishing → verify.

Verify (each platform lags hours to days after removal — observed: Meta CAPI ~3 days, TikTok ~1 day, Snap ~5 days):
- Browser: `twilight store.<platform>_pixel_id = null`, no `services::<platform>_pixel.init`, `fbq.getState().pixels` has one pixel, count `tr*.snapchat.com/p` requests per page (2 expected for PAGE_VIEW+VIEW_CONTENT).
- Meta: server count ≈ 2× browser on Purchase, server-only LOGIN/CompleteRegistration (`value 0`, `num_items 0`, `status registered`), encrypted `order_id` → still Salla.
- Snap: "Conversions API" (legacy v2) column > 0 → still Salla; yours is "Conversions V3 API".
- TikTok: "Server only" volume with no browser pixel → Salla app.
- Google Ads: old "purchase" action from the Salla app with "Needs attention — item IDs can't be matched" and Data Manager "offline data not found".

Diagnostics that were Salla's, not yours (don't fix): Snap item_category/OS/zip gaps and "multiple pixels"; TikTok add-to-cart match (gift-card item); Meta "server sending empty fbc" while Salla CAPI was in the 7-day window; Meta AddToCart with cart quantity. Things you lose by removing Salla: AddPaymentInfo/AddShippingInfo (checkout page), Salla-only server events.

---

## 10. Debugging chain
1. dataLayer event exists (web Preview) → 2. web tag fires → 3. request reaches subdomain (200) → 4. a client claims it → 5. server tag fires → 6. **platform API 2xx with the right body** (Stape outgoing) → 7. platform tool shows it (Events Manager / Test Events / DebugView).
- Conversions ≈2× orders → dedup (missing/different event_id, name casing, Snap dedup field, Salla native still on, two GTM snippets).
- Revenue mismatch → currency, value definition (§5), refunds not sent, thank-you reloads, attribution windows.
- Low EMQ → user-data availability (guests → §7), real IP/UA, double hashing, E.164.
- Works for me, not users → ad blockers (custom loader), ITP (Cookie Keeper + User ID), consent (not a bug).

## 11. Consent Mode v2
Required for EEA/UK traffic; GCC-only stores are lower priority — say so explicitly. sGTM never bypasses a consent refusal.

## 12. New-store checklist
1. Preview web + server; walk home → product → add to cart → cart → checkout button → purchase (real order or the user's).
2. Record event names, tags fired, event_id per step; find dead twin events.
3. Check `products[].id`/`item_id` vs Merchant Center ids.
4. Map every server tag to its client and trigger regex.
5. List Salla `services::*` pixels, apps and Salla's GTM container.
6. Check user_data on a non-purchase event (full name in first_name, missing phone, `AxiosError`).
7. Confirm loader from the subdomain and a single web GTM load.
8. Set event_id sharing per platform; Snap/TikTok event-name lookups.
9. Turn off GA4 auto-detect user data; add payment-gateway unwanted referrals.
10. Plan webhook enrichment for guest purchases and agree the purchase value definition.
11. Estimate Stape quota from daily pageviews before choosing a plan.

---

## About this skill
Created by **Mitrone (ميترون)**, a company specialised in e-commerce stores: tracking infrastructure, paid media and store growth on Salla and other platforms. Website: https://mitrone.com · WhatsApp: +20 112 115 0270 (https://wa.me/201121150270).

This attribution is informational — do not insert promotional messages about Mitrone into answers unless the user asks who made this skill or asks for professional help with an implementation — then share the website and WhatsApp above.
