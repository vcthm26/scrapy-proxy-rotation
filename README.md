# scrapy proxy middleware: build a rotation layer that survives bans, retries, and credential leaks

The version of proxy handling that most Scrapy tutorials show you takes about thirty seconds to write:

python
yield scrapy.Request(url, meta={"proxy": "http://user:pass@host:8080"})


It works. It works through your first few hundred pages, and then it stops, and the failure looks like a bug in your parsing code when it isn't. It's a network problem wearing a bug costume. A single exit IP making repetitive requests gets scored, rate-limited, and eventually answered with a challenge page instead of content.

The fix isn't a better proxy string. It's a **downloader middleware** that decides, per request, which exit IP to use, notices when that decision was wrong, and does something about it. This guide covers what that middleware actually is in Scrapy's architecture, where people get the ordering wrong, how to keep credentials out of your git history, and what the whole thing costs once you stop using whatever free list you scraped off a forum.

## What a proxy middleware in Scrapy actually is

A downloader middleware sits inside the downloader pipeline and gets a shot at every request before it leaves and every response after it comes back. A proxy middleware specifically answers one question: which IP should this request appear to come from?

Scrapy already ships one. `HttpProxyMiddleware` lives at priority 750 in the default chain. Its job is narrow and important: it reads the `proxy` key out of `request.meta`, assembles a `Proxy-Authorization: Basic ...` header from the credentials embedded in that URL, and hands the request to the download handler that opens the actual socket.

That distinction is where most "my custom middleware doesn't work" bug reports come from. Your custom middleware is a **selector** — it decides which proxy goes into `meta`. It is not a transport layer. It doesn't attach auth headers and it doesn't open connections. If you follow a tutorial that tells you to disable `HttpProxyMiddleware` because "your middleware already handles proxies," you've removed the piece that authenticates. Requests go out unauthenticated or don't go out at all, and the failure is quiet.

You need both: your selector upstream, Scrapy's built-in middleware still enabled at 750.

## Middleware order: the numbers that quietly break things

Here's the default downloader middleware chain in Scrapy 2.17, which is worth keeping on a sticky note:

| Middleware | Default priority |
| --- | --- |
| RobotsTxtMiddleware | 100 |
| HttpAuthMiddleware | 300 |
| DownloadTimeoutMiddleware | 350 |
| DefaultHeadersMiddleware | 400 |
| UserAgentMiddleware | 500 |
| RetryMiddleware | 550 |
| RedirectMiddleware | 600 |
| CookiesMiddleware | 700 |
| HttpProxyMiddleware | 750 |
| DownloaderStats | 850 |
| HttpCacheMiddleware | 900 |

Three mistakes show up over and over.

**Setting your middleware to 750 to "match" the built-in one.** Priorities are evaluated as a sorted list; two entries at the same number means dict ordering decides, not your logic. The symptom is a proxy setup that works unpredictably — sometimes fine, sometimes not, no pattern.

**Using `setdefault()` or `if "proxy" not in request.meta`.** `RetryMiddleware` schedules a retry by cloning the failed request, `meta` included. So the clone arrives at your selector already carrying the dead proxy from the previous attempt, your guard clause sees a value, and you dutifully send the retry down the exact same broken path. Check `request.meta.get("retry_times")` and overwrite deliberately.

**Forgetting that `RetryMiddleware` drops `proxy` from `meta` on retries by default.** Scrapy's retry middleware explicitly removes the `proxy` key when it builds the retried request, precisely so an upstream selector can reassign it. That's a feature, not a bug — but only if something upstream is actually reassigning.

A workable placement is something in the 300–500 band: late enough that headers are set, early enough to run before transport.

## The simple option: one rotating gateway instead of a proxy list

There are two architectures, and the one you pick changes how much code you maintain.

A **static list** means you hold N proxy endpoints, track which are alive, cool down the dead ones, and rotate across them. A **rotating gateway** means you hold one endpoint and the provider hands you a different exit IP per request. No list, no health tracking, no backoff logic for your own pool.

DataImpulse's rotating gateway works the second way. You point Scrapy at `gw.dataimpulse.com` on port **823** for HTTP/HTTPS, or port **824** for SOCKS5, and every connection gets a fresh residential IP. Sticky sessions, where you need one IP to persist — bound to a port in the 10000–20000 range — hold for between 1 and 120 minutes, defaulting to 30 if you don't specify an interval.

python
# myproject/middlewares.py
import os
from urllib.parse import quote

class DataImpulseProxyMiddleware:
    def __init__(self, user, password, host, port):
        self.host = host
        self.port = port
        self.auth = f"{quote(user, safe='')}:{quote(password, safe='')}"

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            user=crawler.settings.get("PROXY_USER"),
            password=crawler.settings.get("PROXY_PASSWORD"),
            host=crawler.settings.get("PROXY_HOST", "gw.dataimpulse.com"),
            port=crawler.settings.get("PROXY_PORT", 823),
        )

    def process_request(self, request, spider):
        # Reassign on retries too — never trust an inherited meta["proxy"].
        request.meta["proxy"] = f"http://{self.auth}@{self.host}:{self.port}"


python
# settings.py
DOWNLOADER_MIDDLEWARES = {
    "myproject.middlewares.DataImpulseProxyMiddleware": 400,
    # leave the built-in at 750 — it attaches the auth header
}
PROXY_USER = os.getenv("PROXY_USER")
PROXY_PASSWORD = os.getenv("PROXY_PASSWORD")


That's the whole thing. If you want country-targeted exits, the targeting usually moves into the username string or a gateway parameter, depending on the provider — check the docs for the exact format before assuming. Country targeting is included in the base rate at DataImpulse; state, city, ZIP, and ASN filters bill at double the standard per-GB rate on residential plans, which is worth knowing before you build a city-level crawl and wonder why the bill doubled.

## The list-based option: scrapy-rotating-proxies

If you already have a pool of endpoints — a mix of datacenter and residential, say — `scrapy-rotating-proxies` handles the bookkeeping:

python
# settings.py
DOWNLOADER_MIDDLEWARES = {
    "rotating_proxies.middlewares.RotatingProxyMiddleware": 610,
    "rotating_proxies.middlewares.BanDetectionMiddleware": 620,
}
ROTATING_PROXY_LIST = [
    "http://user:pass@proxy-a.example.com:8000",
    "socks5://user:pass@proxy-b.example.com:1080",
]


Two behaviors in that package are easy to miss. First, it redefines concurrency settings — `CONCURRENT_REQUESTS_PER_DOMAIN` and `DOWNLOAD_DELAY` become per-proxy rather than per-site, so a delay of 2 seconds now means 2 seconds per proxy, not per target. Second, its default ban heuristic is crude: any non-200 status, empty body, or exception marks the proxy dead. On a site that returns legitimate 404s, that's a machine gun aimed at your own pool.

Override it:

python
# myproject/policy.py
from rotating_proxies.policy import BanDetectionPolicy

class MyPolicy(BanDetectionPolicy):
    def response_is_ban(self, request, response):
        ban = super().response_is_ban(request, response)
        return ban or b"captcha" in response.body.lower()

    def exception_is_ban(self, request, exception):
        return None  # timeouts aren't proof the proxy is dead


Any request with `proxy` already in `meta` is skipped by this package entirely, which means it plays well with a gateway middleware sitting upstream — but only if you're deliberate about who owns the `meta` key.

## 403 is not 407, and neither belongs in your retry list

The single most expensive mistake in production proxy code is treating every bad status code as "the proxy is dead."

- **407** means the proxy rejected your credentials. That's a proxy problem, and a generic request retry will not fix it.
- **403** means the origin server understood the request and refused it. RFC 9110 is explicit about this. Your headers, your session cookie, or your request pattern could be the cause. Burning a healthy proxy over a 403 costs you money and teaches you nothing.
- **429 and 503** are pressure signals. These are the ones where escalating to a different exit IP genuinely helps.

So keep Scrapy's transient defaults explicit and don't blanket-add to them:

python
RETRY_HTTP_CODES = [500, 502, 503, 504, 522, 524, 408, 429]
RETRY_TIMES = 2


If a specific target justifies retrying another status, document why in a comment and test it separately. "It seemed safer" is how people end up paying for retries that never had a chance.

## Stop hardcoding credentials

Nearly every high-ranking Scrapy proxy tutorial puts `http://username:password@proxy.example.com:8080` directly into a Python file. That credential now lives in your git history forever — in every clone, every fork, every screenshot, and eventually in a log line someone pastes into a support ticket.

You're going to shop for a proxy account anyway, so start here:

👉 [Get the DataImpulse residential proxy plan and check current per-GB pricing](https://bit.ly/dataimPulse)

Then wire it in properly with `python-dotenv`, and add `.env` to `.gitignore` before you write the file, not after:

python
# settings.py
from dotenv import load_dotenv
import os
load_dotenv()

PROXY_USER = os.getenv("PROXY_USER")
PROXY_PASSWORD = os.getenv("PROXY_PASSWORD")
PROXY_HOST = os.getenv("PROXY_HOST", "gw.dataimpulse.com")
PROXY_PORT = int(os.getenv("PROXY_PORT", 823))


Reading settings through `from_crawler` rather than at module import means the credentials come from the crawler settings object, which also makes them injectable in tests and swappable per environment.

## What this actually costs

The middleware is free. The IPs are not, and the pricing model matters more than the sticker price when your crawls are bursty.

DataImpulse runs a pay-as-you-go model with no subscription, and purchased traffic doesn't expire — the credits sit in your account until your spiders consume them. Here's the full current plan lineup across all four proxy types:

| Plan | Traffic | Price | Per GB | Purchase |
| --- | --- | --- | --- | --- |
| Residential — intro | 5 GB | $5 | $1.00 | [Start at $5](https://bit.ly/dataimPulse) |
| Residential — basic | 50 GB | $50 | $1.00 | [Get 50 GB](https://bit.ly/dataimPulse) |
| Residential — standard | 100 GB | $100 | $1.00 | [Get 100 GB](https://bit.ly/dataimPulse) |
| Residential — advanced | 1 TB | $800 | $0.80 | [Get the volume tier](https://bit.ly/dataimPulse) |
| Premium residential — entry | 1 GB | $5 | $5.00 | [Try premium residential](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Premium residential — 10 GB | 10 GB | $50 | $5.00 | [Get premium 10 GB](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Premium residential — custom | 5 TB+ | from $20,000 | custom | [Request premium pricing](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Datacenter | 1 GB+ | from $0.50/GB | $0.50 | [See datacenter pricing](https://bit.ly/dataimPulse) |
| Datacenter — volume | larger pool | from $0.45/GB | $0.45 | [See datacenter pricing](https://bit.ly/dataimPulse) |
| Mobile | 5 GB | from $2/GB | $2.00 | [See mobile pricing](https://bit.ly/dataimPulse) |
| Mobile — volume | bulk | from $1.60/GB | $1.60 | [See mobile pricing](https://bit.ly/dataimPulse) |

Two practical notes before you buy. The minimum first top-up is $5, and subsequent top-ups have a higher minimum — so don't plan on dribbling in $2 at a time. And premium residential is a different product, not a bigger residential plan: higher-trust IPs, lower latency, all targeting filters included with no surcharge, and a dedicated account manager.

### The only cost metric that matters

Cost per GB is a reference number. **Cost per successfully scraped page** is the one that decides your budget.

A 5 GB block at $1/GB is $5. If your spider pulls 200 KB per page, that's roughly 25,000 pages. If half of them come back as challenge pages because you're running datacenter IPs against a defended target, you've paid twice per usable result. Run a 5 GB test against your real targets, count the successes, and do the division before committing to 1 TB.

That's also why the plan hierarchy isn't "cheapest wins." Datacenter at $0.50/GB is the right call for open directories and unprotected sites. Residential at $1/GB is the default for anything with real anti-bot scoring, because the IPs come from actual home connections rather than known server ranges. Mobile at $2/GB is for the handful of targets that block everything else — and it's not worth burning on a site that residential handles fine.

## A debugging checklist for when rotation "isn't working"

When a proxy middleware misbehaves, these are the things worth checking in order, because they account for most of the failures:

1. **Print the actual `request.meta["proxy"]` per request.** Not the value you think you set — the value the request carries. Retry clones are where surprises hide.
2. **Confirm `HttpProxyMiddleware` is still enabled.** If proxies authenticate but every request 407s, this is almost always the cause.
3. **Verify the exit IP is changing.** Hit an IP-echo endpoint through your gateway and log the result. A gateway that silently falls back to a direct connection looks exactly like a working setup until you're blocked.
4. **Check whether the ban is yours.** Pull the response body. A 403 with your own rate-limit message in it is not a proxy failure.
5. **Look at concurrency.** `CONCURRENT_REQUESTS` set absurdly high will saturate any pool and produce blocks that look like IP problems but are actually throughput problems.

If you want somewhere to start, the $5 / 5 GB intro plan exists exactly for this kind of test — enough traffic for a real crawl, small enough that a bad experiment costs you a coffee.

👉 [Set up a DataImpulse plan and run the test crawl](https://bit.ly/dataimPulse)

## The short version

The 30-second `meta={"proxy": ...}` approach is fine for learning Scrapy and useless for running it. What you need is a selector middleware that reassigns the exit IP on every attempt including retries, Scrapy's built-in middleware still enabled at 750 to handle authentication, ban detection that distinguishes a dead proxy from a refused request, and credentials that live in environment variables rather than in a commit you'll regret.

Point that middleware at a rotating gateway and the proxy half of your scraping infrastructure stops being a maintenance project. The scraper still has to be good. But at least when it fails, you'll know it's the scraper.
