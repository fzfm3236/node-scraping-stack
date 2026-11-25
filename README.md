# Web scraping Node.js: fetch or Axios, Cheerio or Playwright, and why clean requests still get blocked (with a plan and credit-cost comparison)

A working Node.js scraper needs four things: Node 18 or newer, `fetch` or Axios for the request, Cheerio for parsing, and Playwright for the pages that build their HTML in the browser. Add `p-limit` so you don't fire 500 requests at once. That stack is free, and it handles most public sites.

What isn't free is everything that sits between your request and the HTML: residential IPs, a Chromium fleet, and the maintenance that comes with a target site changing its anti-bot vendor. ScraperAPI's free plan covers the first test of that layer with 1,000 credits a month and 5 concurrent threads at $0, which is enough to find out whether your targets even need it.

> **Quick pick:** start on the free plan (1,000 credits/month, 5 threads, $0). Run the code below against your real URLs, then look at the credit cost per page before paying for anything. The tier with a published monthly rate is Scaling: 5,000,000 credits and 200 concurrent threads for **$475/month**, or **$427.50/month billed annually** ($5,130/year).
>
> 👉 [Compare every plan and open a free account](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, and what each one is for

ScraperAPI lists eight plans. Free and Scaling have published figures; the tiers in between list their credit allowance and rate on the plans page.

| Plan | Monthly credits and listed price | Get started |
| --- | --- | --- |
| Free | 1,000 credits, 5 concurrent threads, $0 | [Open the free plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Allowance and rate on the plans page | [See Hobby's current credits and price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Allowance and rate on the plans page | [See Startup's current credits and price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Allowance and rate on the plans page | [See Business's current credits and price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling (most popular) | 5,000,000 credits, 200 threads, $475/month or $427.50/month annually | [Start on Scaling with annual billing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Allowance and rate on the plans page | [See Professional's current credits and price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Allowance and rate on the plans page | [See Advanced's current credits and price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom quote from sales | [Request an Enterprise quote](https://www.scraperapi.com/pricing?fp_ref=coupons) |

The tiers are ordered by monthly credit allowance, so the selection rule is dull but correct: estimate your monthly page count, multiply by the credit cost of your typical page, and buy the smallest tier that covers it. Two billing behaviours matter more than the tier names.

**Auto-upgrade and pay-as-you-go solve different problems.** Auto-upgrade is opt-in on Hobby, Startup and Business: when credits run out, the account moves up a tier instead of stopping. Pay-as-you-go is available on Scaling, Professional, Advanced and Enterprise, and keeps you scraping past your limit at a fixed per-credit rate up to a monthly cap you set. The cap is a spending limit, not a prepayment, and the dashboard will not exceed it unless you change it.

**Overshooting concurrency is cheap; overshooting credits is not.** Exceed your concurrent threads and you get a 429 telling you to come back within the limit. There's no surcharge, the request is simply refused rather than queued. Exceed your credits and scraping pauses until renewal unless one of the two mechanisms above is on.

Two other details worth knowing before you pay: annual plans hand you the full year's credits upfront, so a seasonal workload can burn them unevenly and renew early for a fresh batch. And there's a 7-day, no-questions refund window, plus a 7-day trial with 5,000 credits that doesn't ask for a card.

## What a request actually costs

This is where most Node.js scraping tutorials stop and where budgets actually get decided. A standard page is 1 credit. Nothing else is.

| Request type | Credits | Notes |
| --- | --- | --- |
| Standard page | 1 | Per request |
| E-commerce (Amazon, Walmart, eBay) | 5 | Per request |
| SERP (Google, Bing) | 25 | Includes subdomains |
| Social (LinkedIn) | 30 | Per request |
| Bot-protection bypass | 10 | Cloudflare, Cloudflare Turnstile, DataDome, PerimeterX/HUMAN, Akamai |
| JS rendering (`render=true`) | 10 | Per request |
| Premium proxies (`premium=true`) | 10 | Per request |
| Screenshot (`screenshot=true`) | 10 | Per request |
| Premium proxies + JS rendering | 25 | Per request |
| Ultra Premium (`ultra_premium=true`) | 30 | Paid plans only |
| Ultra Premium + JS rendering | 75 | Paid plans only |
| Geotargeting, sessions, device type, output format, auto parsing | 0 | No extra cost |

Notice that combinations get their own rows, so credit costs don't simply add. Premium proxies plus rendering is listed at 25, not 20. That's why the dashboard ships a Domain Cost Estimator: check a URL before you build a crawler around it, and set `max_cost` per scrape so no single request can exceed a number you chose. Every response also reports what it cost in the `sa-credit-cost` header, which is the only reliable way to learn your real average.

On Scaling, 1 credit works out to **$0.095 per 1,000 page requests** at $475/month, or roughly $0.086 per 1,000 on the annual rate. A page that needs JS rendering at 10 credits is about $0.95 per 1,000. Run that multiplication against your own mix before you pick a tier, because a crawl of 50,000 plain pages and a crawl of 50,000 Cloudflare-protected rendered pages are two different products.

👉 [Price your target URLs against each tier](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The minimum useful scraper: Axios plus Cheerio

bash
npm i axios cheerio


The request layer deserves more care than a one-line `axios.get`. A bare request carrying Node's default user agent gets filtered on most commercial sites before the HTML is even generated.

js
import axios from 'axios';

const client = axios.create({
  timeout: 15000,
  maxRedirects: 5,
  headers: {
    'User-Agent':
      'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36',
    'Accept':
      'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
    'Accept-Language': 'en-US,en;q=0.9',
  },
});


Then parse with Cheerio and scope every lookup to the row you're iterating. `.text()` on a global selector returns the concatenated text of every match on the page, which is how scrapers end up with prices glued to product names.

js
import * as cheerio from 'cheerio';

const { data: html } = await client.get('https://example.com/products');
const $ = cheerio.load(html);

const rows = [];
$('.product-card').each((_, el) => {
  const card = $(el);
  rows.push({
    name: card.find('.product-name').text().trim(),
    price: card.find('.price').text().trim(),
    href: card.find('a').attr('href'),
  });
});


The hard limit of this approach: Cheerio parses markup and nothing else. Its own documentation states that it doesn't render, doesn't load CSS or external resources, and doesn't execute JavaScript. If the number you want is written into the DOM by a script after load, Cheerio will never see it, no matter how good your selectors are.

## When you have to bring in Playwright

Three signals mean the HTTP path is finished: the HTML you downloaded contains a loading skeleton instead of data, the listing only grows as you scroll, or the page ships an empty shell that a JSON payload hydrates on the client. Playwright drives Chromium, Firefox and WebKit through one API and auto-waits for elements, which removes most of the flaky `sleep` calls.

js
import { chromium } from 'playwright';

const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto(url, { waitUntil: 'domcontentloaded' });
await page.waitForSelector('.product-card');
const html = await page.content();
// reuse `browser` for the next URL, close it once when the whole run ends


Reuse matters. Each browser is a real Chromium process consuming hundreds of megabytes, so launching one per URL will exhaust a small VM within minutes. Launch once, open a page per URL, close the browser at the end of the run.

Before you reach for a browser, check the network tab. If the data arrives as JSON from an internal endpoint, fetching that endpoint directly is faster and cheaper than rendering the page that calls it.

## Retries and concurrency, done properly

Most retry loops in tutorials retry everything, which turns a permanent 404 into three wasted requests and a slower failure.

js
async function getWithRetry(url, attempts = 3) {
  for (let i = 0; i < attempts; i++) {
    try {
      return await client.get(url);
    } catch (err) {
      const status = err.response?.status;
      if (status && status < 500 && status !== 429) throw err;
      const backoff = 2 ** i * 1000 + Math.random() * 500;
      await new Promise((r) => setTimeout(r, backoff));
    }
  }
  throw new Error(`Gave up after ${attempts} attempts: ${url}`);
}


The rule inside that `if`: a 4xx other than 429 means the request itself is wrong (bad URL, auth, gone), so fail fast. A 429 or a 5xx means try again later. The delay doubles each attempt and adds up to 500ms of random jitter, so a hundred workers failing at the same moment don't retry in lockstep and re-create the spike that blocked them.

Concurrency is the other half. `p-limit` keeps the number of in-flight requests fixed instead of letting `Promise.all` open as many sockets as you have URLs.

js
import pLimit from 'p-limit';

const limit = pLimit(5);
const results = await Promise.all(urls.map((u) => limit(() => getWithRetry(u))));


Start at 5 and raise it only while the error rate stays flat. On ScraperAPI the same number is a plan limit, and going past it returns a 429 rather than a bill.

## Why changing the User-Agent won't get you through

Header rotation is the advice every tutorial gives, and it stops working exactly where it matters. Two things a raw HTTP client in Node cannot fake: the TLS fingerprint (JA3/JA4) and the HTTP/2 fingerprint. Node's TLS stack does not look like Chrome's, and anti-bot systems built by Cloudflare, DataDome, PerimeterX or Akamai read that handshake before your headers are ever parsed. A perfect Chrome user agent on a Node TLS handshake is a contradiction the detector resolves in milliseconds.

Rotating IPs solves a different problem. If you were blocked for hitting one address 40 times a second, a residential pool helps. If you were blocked on the handshake, a new IP gets the same result with a different number.

That's the point where the build-versus-buy decision stops being philosophical. A headless Chromium on your own server fixes the fingerprint and costs you a few hundred megabytes per instance plus a queue. Residential proxies fix the IP problem and cost more than the servers. Both together, maintained against vendors that change their checks monthly, is a part-time job.

👉 [Run your blocked URLs through the API on the free tier](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Where ScraperAPI slots into the code you already have

It replaces the request layer, not the parsing. Your Cheerio selectors, your retry wrapper, your `p-limit` fan-out and your storage code all stay. The target URL goes to ScraperAPI instead of the site, and the response body is the HTML you were trying to get.

Two parameters cover most of what the built-in client can't do. `render=true` runs the request through a real browser so client-rendered pages come back fully built. `premium=true` routes through residential IPs. Geotargeting, sessions, device type and output format are listed at no extra credit cost, which matters if your data is region-specific: pricing pages and search results differ by country, and a US-only crawler will quietly return the wrong numbers for everything else.

For repetitive product and article pages there's also `/v1/extract`, which takes a JSON schema and fills the common fields from the page's own structured metadata, cutting out a layer of selector maintenance when a site redesigns its markup.

Treat all of this as capability, not a performance promise. The honest framing is the one above: it removes the fingerprint and IP problems from your to-do list and prices them at 10 credits per bypass or render, so you can decide whether that's cheaper than the hours.

## The real comparison isn't the subscription, it's the mix

A single proxy subscription and a small browser VM usually cost less per month than Scaling at $475, right up until you count the hours spent re-tuning a blocked crawler. The number that decides it is your credit cost per successful record, not the sticker price.

Work it out in this order. Crawl 200 URLs of your real targets on the free plan. Read the `sa-credit-cost` header on each response. Average it. Multiply by your monthly page count. If the average is close to 1 credit, most paid tiers will look inexpensive against the engineering time. If it's 11 (a Cloudflare page) or 25 (Cloudflare with rendering), the same volume costs an order of magnitude more and the cheapest correct answer may be to reduce how many pages you actually need, by caching or by hitting an internal JSON endpoint instead.

Two controls make this safe to try: `max_cost` on any single request, and the monthly spending cap on pay-as-you-go, which the dashboard won't cross unless you change it.

## FAQ

**Cheerio or Playwright?** Cheerio first, always. It's fast, it costs no browser memory, and it handles anything present in the delivered HTML. Move a specific URL to Playwright only after you've confirmed the data isn't in the source.

**Will Node.js scraping get blocked?** On unprotected sites, rarely. On sites running commercial anti-bot, yes, and the reason is usually the TLS and HTTP/2 fingerprint rather than your headers or your IP. Node's TLS handshake is distinguishable from Chrome's.

**What happens if I run out of credits or threads mid-crawl?** Threads: a 429 telling you to come back inside the limit, with no surcharge and no queueing. Credits: scraping pauses until renewal, unless auto-upgrade (Hobby, Startup, Business) or pay-as-you-go (Scaling and above) is enabled.

**Do I need to pay to test it?** No. The trial gives 5,000 credits over 7 days without a card, and the free plan continues at 1,000 credits and 5 threads per month afterwards.

**Can I cancel or get a refund?** Cancelling from the dashboard is free at any time, and there's a 7-day no-questions refund window.

## Which one you should actually buy

Static HTML, no login walls, a few thousand pages a month: your Axios and Cheerio setup is enough, and the free plan is only useful as a fallback for the odd URL that returns a challenge page.

Targets behind Cloudflare, DataDome or Akamai, or listings that only exist after rendering: go straight to the API layer and keep your Node parsing code. Start on the free plan, log the `sa-credit-cost` header for a week, then buy the smallest tier that covers your measured average, with pay-as-you-go enabled if you're on Scaling or above.

Before you enter card details, confirm three things on the page: the credit cost of your specific targets via the Domain Cost Estimator, the concurrency cap of the tier you chose (it's the limit you'll hit first on a wide crawl), and the 7-day refund window, which is short enough that you should test your real URLs inside it rather than after.

👉 [Start on the free plan and upgrade once you've measured your credit cost](https://www.scraperapi.com/pricing?fp_ref=coupons)
