# Google Shopping API: Is There an Official One, What Fields Do You Actually Get, and How Do the Per-Request Costs Stack Up?

Google's own Shopping APIs manage the product feed you upload to your Merchant Center account. They do not take a keyword and return other merchants' offers. If you want a search term in and a page of products with prices, sellers, ratings and delivery terms out, that data has to come from scraping Google Shopping result pages, which is what a SERP-based API does. ScraperAPI runs this as a structured data endpoint and bills it at **25 credits per Google/Bing request**, subdomains included.

That distinction decides your whole build. Everything below covers what the official Google APIs actually do, which fields a scraping endpoint returns, how filtering and pagination behave, and where the credit math turns expensive.

> **Most readers:** start on the free 1,000 credits (about 40 Google Shopping requests) to confirm the JSON shape fits your parser, then move to Hobby at 100,000 credits, roughly 4,000 Google Shopping requests a month. Go to Startup or above when you're tracking more than a few hundred queries a day, because at that volume concurrent threads, not credit balance, becomes what limits you.
> 👉 [See the current ScraperAPI plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## All ScraperAPI plans, and what each one buys in Google Shopping requests

Every tier draws from the same credit pool, and a Google Shopping request costs 25 credits whether you're on the free plan or Enterprise. That makes the credit allowance the number to compare, not the sticker price.

| Plan | Monthly API credits | Google Shopping requests/mo* | Concurrent threads | Overage handling | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | ~40 | 5 | None | [Start on the free credits](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | ~4,000 | 20 | Auto-upgrade (opt-in) | [See Hobby pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | ~40,000 | 50 | Auto-upgrade (opt-in) | [See Startup pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | ~120,000 | 100 | Auto-upgrade (opt-in) | [See Business pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | ~200,000 | 200 | Auto-upgrade + pay-as-you-go | [See Scaling pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | ~420,000 | 300 | Auto-upgrade + pay-as-you-go | [See Professional pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | ~860,000 | 500 | Pay-as-you-go only | [See Advanced pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | ~880,000+ | 500+ | Pay-as-you-go only | [See Enterprise pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |

\*At 25 credits per request, with no add-ons and no other endpoints sharing the pool.

A few billing rules change how you should size this. Credits reset when the subscription renews and **do not roll over**, so unused headroom at the end of the month is gone. Annual billing takes **10% off** on every self-serve plan, with Enterprise priced individually. Pay-as-you-go, which lets you keep scraping past your limit at a fixed per-credit rate up to a cap you set, only appears from Scaling upward. Auto-upgrade, which bumps you to the next tier when credits run out, is opt-in and available from Hobby through Professional.

Concurrency is enforced separately from credits. Exceed your threads and you get a **429** telling you to back off; the request is rejected rather than queued, and you aren't charged for it.

For exact monthly prices per tier, check the pricing page directly. The figure that matters more in practice is cost per usable Google Shopping request, and that number depends on the add-ons below.

## What Google's official Shopping APIs actually do

There are two names you'll run into, and both serve merchants rather than data consumers. The Content API for Shopping has been succeeded by the Merchant API, which is built around a `products` resource covering **more than 60 product attributes**. Some are mandatory for approval, others become mandatory depending on location, product type, variants or bundles.

The resource supports `products.insert`, `get`, `update`, `delete`, `list` and `custombatch` for bulk operations. There's also a `productstatuses` resource that returns whether a given product is approved or rejected for a specific destination, which is how you trace a data quality problem back to a missing attribute.

One behavior catches people building sync jobs. If a product already exists with the same `channel`, `contentLanguage`, `offerId` and `feedLabel`, `insert` updates that entry and **replaces all data previously written by API calls**. Partial updates belong in `update` with an update mask, not in `insert`.

The reporting side is older. `reports.search` is marked deprecated, and its metrics (clicks, impressions, CTR, orders, shipped orders, days to ship, average order value in micros) describe your own Merchant Center account. Conversions, conversion value and conversion rate are only available for the `FREE_PRODUCT_LISTING` program. Monetary values come back in micros, where 1 USD equals 1,000,000 micros.

None of this exposes a keyword search across Google Shopping. Your inputs are your merchant ID and your feed; the outputs are your catalog status and your own traffic reports.

## What a SERP-based Google Shopping API returns

The response is a `shopping_results` array. SerpApi documents the fullest field list, and ScraperAPI's endpoint returns the same core shape:

- `position`, the rank on the page
- `title`
- `link`, the retailer's product page
- `product_link`, the Google Shopping listing
- `product_id`, Google's catalog ID
- `source`, the seller or retailer name
- `price`, formatted text such as a currency string
- `extracted_price`, the same value as a float
- `rating`, out of 5
- `reviews`, an integer count
- `thumbnail`
- `delivery` (ScraperAPI labels it `delivery_options`)
- `extensions`, an array holding discount and shipping notes

Two of those decide whether your product works. Sort on `extracted_price`, never on `price`, because a formatted string sorts alphabetically and puts a cheap item above an expensive one the moment currency symbols or thousands separators differ. ScraperAPI's own tutorial makes the same point: use `extracted_price` for numeric comparisons.

The second is discount data. Apify's Google Shopping actor returns `old_price` and a `tag` field carrying the percentage off, which means a deal-finder can filter on a real field instead of regexing a discount out of a title. If discount discovery is your product, check that your provider returns both; if you're only tracking list prices, `extracted_price` plus `delivery` covers it.

ScraperAPI returns `output_format` as `json` (default) or `csv`. Both are free add-ons, so exporting straight into a spreadsheet costs the same as JSON.

## Filters, localization, and the parameters that matter

ScraperAPI's Google Shopping endpoint takes `country_code` for the market, `domain` for the Google TLD, `uule` to encode a specific city for local results, and `num` for results per page. When you need a Google domain scraped from another country, both the TLD and the country code have to be set, otherwise the localization silently doesn't apply.

Language matters here because Google's own layout limits what pagination can do. SerpApi's documentation states plainly that the current Shopping layout ignores the `start` offset parameter, so every request returns the first page of roughly 40 items, and `num` is forwarded but doesn't change the count. If your plan assumed you'd page through 400 results per query, you need to reach deeper coverage through narrower filters (price bands, seller type, on-sale) rather than offsets. ScraperAPI accepts `start` and `num`, and you should verify depth on your own queries before writing pagination logic around it.

Filter parameters are where the two providers converge. SerpApi and Apify both expose `min_price`, `max_price`, `free_shipping`, `on_sale`, `google_domain`, `gl`, `hl` and `device`; SerpApi adds `sort_by` with `1` for low-to-high and `2` for high-to-low, plus `small_business` and a `shoprs` token that can carry a whole filter set and replace `q` entirely.

One detail almost nobody documents upfront: on SerpApi, explicit parameters such as `min_price`, `max_price`, `sort_by`, `free_shipping`, `on_sale` and `small_business` **override** the matching conditions embedded in a `shoprs` token. If you pipe a saved token in and also pass a price floor, the floor wins and the rest of the token still applies. Mixing the two without knowing this produces result sets that look like the filter was ignored.

For volume work, ScraperAPI's async mode takes a batch of query URLs, returns a `statusUrl` per job, and lets you poll until the job reports finished, at which point `response.body` holds the payload. That's the path for nightly runs across a query list rather than one synchronous call per keyword.

## Where the credit math gets expensive

Base cost for a Google Shopping request is 25 credits. Add-ons stack on top:

- JavaScript rendering adds **10 credits**
- premium proxies add **10 credits**
- bot-protection bypass adds **10 credits**
- Ultra Premium adds **30 credits**, and **75** combined with rendering

Geotargeting, sessions, device type, output format and auto parsing cost nothing extra, so the localization parameters above don't inflate your bill. Every response carries a `sa-credit-cost` header telling you what that request actually consumed, and you can check any URL's cost with the Domain Cost Estimator in the dashboard before committing, or set `max_cost` per scrape to cap a single request.

Read that as a warning about defaults. A pipeline that turns on rendering and premium proxies "just in case" moves a Google Shopping request from 25 credits to 45, which cuts your effective monthly capacity by 44%. On Hobby that's 4,000 requests becoming roughly 2,200, for no gain if the Shopping page renders fine without JS.

The two competitors price on different axes, which makes direct comparison awkward. SerpApi caches results for **1 hour** and serves cached searches **free, without counting them against your monthly searches**, so a dashboard refreshing the same twenty queries benefits. Apify bills **per page**, with `max_pages` defaulting to 1 and roughly 40 products per page, and setting it to 0 for unlimited is the fastest way to multiply a bill on a broad query. Neither publishes a per-request figure in the material I could verify, so compare mechanisms rather than headline rates: cached-and-free rewards repetition, per-page rewards narrow queries, and credit-per-request with fixed add-on costs is predictable once you know which add-ons you actually need.

👉 [Check what 25 credits per Google Shopping request leaves you on each plan](https://www.scraperapi.com/pricing?fp_ref=coupons)

## FAQ

**Does Google have an official Shopping search API?** No. The official APIs, Content API for Shopping and its successor Merchant API, manage your own product catalog and report on your own listings. Keyword search across all merchants is only available by scraping Google Shopping results.

**Can I filter by price range and free shipping?** Yes on the SERP-based APIs. SerpApi and Apify both accept `min_price`, `max_price`, `free_shipping` and `on_sale`. On ScraperAPI you can pass Google's own parameters through the endpoint, or filter client-side on `extracted_price` and `delivery_options` after the response lands.

**Is the price a string or a number?** Both, usually. `price` is formatted text and `extracted_price` is a float. Store the float and keep the string for display.

**How does pagination affect cost?** Each request is billed once regardless of how many items come back. Google's current layout returns roughly 40 items per page and ignores offset-based paging, so reaching more products means more requests with different filters, not one long page.

**Can I get CSV instead of JSON?** On ScraperAPI, yes, through `output_format`, at no extra credit cost.

**What happens when I run out of credits?** Scraping pauses until renewal unless auto-upgrade is on, available from Hobby to Professional, or pay-as-you-go is enabled, available from Scaling up.

## Picking a plan without overbuying

Work backward from queries per month, not from credits. One Google Shopping request per query per run, times runs per day, times 30. If you check 50 competitor queries once a day, that's 1,500 requests, comfortably inside Hobby's ~4,000. At 500 queries daily you're at 15,000 requests and need Startup. Add rendering or premium proxies and divide your capacity by roughly 1.8 per add-on.

Two things to confirm before you pay. First, run a handful of your real queries on the free credits and check whether the fields you need (especially discount and delivery data) come back populated for your category, since Google's Shopping layout varies by query. Second, decide now whether you'll ever need pay-as-you-go: it doesn't exist below Scaling, and discovering that mid-month means upgrading a tier rather than topping up.

👉 [Start with the free 1,000 credits and size the plan against your own queries](https://www.scraperapi.com/pricing?fp_ref=coupons)
# google-shopping-api-costs
