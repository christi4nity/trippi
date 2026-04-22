---
name: hyatt-strategy
description: Hyatt-first hotel strategy for Christian. Use when a destination may have useful Hyatt options, when comparing Hyatt cash vs points, or when checking if Chase UR should be transferred to Hyatt.
version: 1.0.0
---

# Hyatt Strategy

Christian is unusually likely to want the Hyatt angle first.

Use this skill when:
- a trip involves hotels and Hyatt might be relevant
- comparing cash vs points for a Hyatt stay
- deciding whether to transfer Chase UR to Hyatt
- checking World of Hyatt balances, rates, or award availability
- evaluating Park Hyatt, Alila, Andaz, Miraval, Thompson, all-inclusive, or high-value Hyatt redemptions

## Default posture

Do not start with generic hotel metasearch if Hyatt is plausibly in play.
Start by checking Hyatt directly, then compare Hyatt against other options.

## Preferred workflow

1. If account-specific context is needed, log into Hyatt first.
   - Christian's Hyatt credentials are intended to be pulled from 1Password at runtime, not stored in plaintext env.
   - First verify runtime credential plumbing if login context seems missing:
     - check that `op` is installed and signed in
     - confirm the helper script exists at `/home/hermes/.hermes/scripts/hyatt_1password_credentials.py`
     - run the helper with `--shell` and verify it emits `HYATT_USERNAME` and `HYATT_PASSWORD`
   - Use `hyatt_login` with Christian's Hyatt credentials once verified.
   - Then use `hyatt_get_world_of_hyatt_balance` to fetch points balance, tier, and qualifying nights.

2. Search Hyatt inventory first.
   - Use `hyatt_search_hotels` for the destination and dates.
   - If a promising property appears, use `hyatt_get_room_rates`.
   - If the question is specifically about using points, also use `hyatt_redeem_points`.
   - Current MCP quirk: `hyatt_search_hotels` and `hyatt_get_hotel_details` can be flaky or return Hyatt "We're sorry" wrapper data even when the property ID is valid. Do not treat empty search/detail results alone as proof that a property or award is unavailable.
   - In that failure mode, go straight to the likely hotel ID and use `hyatt_redeem_points` / `hyatt_get_room_rates` directly.
   - For availability hunts like "find N award nights in a row this year," brute-force date windows directly instead of relying on search. Use batched parallel calls across consecutive start dates and capture every positive hit.
   - For known Hyatt properties, do not rely on `hyatt_search_hotels` alone. If search is flaky or empty, try the likely hotel slug/ID directly with `hyatt_redeem_points` and `hyatt_get_room_rates`.
   - Be aware the Hyatt MCP can fail asymmetrically: search/details may return empty results or Hyatt's "We're sorry" wrapper even while `hyatt_redeem_points` still returns structured availability results. In that case, treat the award endpoint as the most useful signal, but label confidence accordingly.
   - If the user asks for date-finding (for example "find me 4 award nights in a row this year"), brute-force the date range instead of sampling a few dates. Prefer batched parallel checks over weekly spot checks.

3. For each promising Hyatt option, calculate whether the redemption is good.
   - Use Hyatt's conservative floor from the toolkit: 1.4 cpp.
   - Compute cpp as:
     cpp = ((cash_price - taxes_or_fees_still_paid_on_award) * 100) / points_required
   - Above 1.4 cpp = good.
   - Well above 1.4 cpp = strong Hyatt use.
   - Below 1.4 cpp = usually pay cash unless there is a strategic reason not to.

4. Always compare against Chase UR transfer logic.
   - Chase UR transfers 1:1 to Hyatt according to `data/transfer-partners.json`.
   - Treat Hyatt as one of the best routine Chase transfer uses.
   - If Hyatt redemption materially beats portal value, say so plainly.

5. Compare Hyatt against non-Hyatt alternatives only after the Hyatt angle is clear.
   - If Hyatt looks weak for the destination or dates, then move to broader hotel comparison.
   - If Hyatt looks strong, recommend it directly rather than burying it in a long list.

## Output style

Always give:
- best Hyatt option
- points required
- cash price if known
- implied cpp
- verdict: use Hyatt points, transfer Chase to Hyatt, or pay cash
- any caveats: poor availability, weak cpp, resort fee issues, room-type mismatch, or speculative data

## Tailoring for Christian

Bias toward Hyatt when:
- the property quality is materially better than peers
- the redemption is at or above floor value
- the property is a premium Hyatt brand Christian is likely to care about
- Chase transfer is the cleanest path

Be skeptical when:
- only premium room awards are available
- cash price is low and cpp is mediocre
- the stay is better booked through FHR/Edit or another luxury channel
- the tool data looks scraped/uncertain and needs confirmation on Hyatt's site

## Tool reliability notes

- Hyatt MCP can degrade in a specific way: `hyatt_search_hotels` may return empty results even for obvious cities/properties, and `hyatt_get_hotel_details` may return Hyatt's generic `We’re sorry.` wrapper instead of real hotel data.
- Hyatt may also actively bot-block requests. Common signals: terminal requests returning HTTP 429, browser pages showing Hyatt's `We’re sorry.` page, error code `E6020`, or blank pages that are actually Kasada challenge shells containing `window.KPSDK` / `ips.js` rather than real Hyatt HTML.
- If that happens, do not treat empty search/detail/rate results as proof the property is unavailable or invalid.
- In that failure mode, try known property IDs/slugs if you have them and use `hyatt_redeem_points` directly for date checks.
- If you do not know the property slug/code, use DuckDuckGo HTML search from the terminal (`https://html.duckduckgo.com/html/?q=site:hyatt.com <property query>`) to surface public Hyatt URLs and extract the hotel code/slug from those URLs.
- If Hyatt is bot-blocking direct browser automation, fall back to MaxMyPoint for directional Hyatt award discovery instead of brute-forcing Hyatt. For known properties, search DuckDuckGo for the MaxMyPoint hotel page (for example `site:maxmypoint.com <property name>`), open it in the browser tool, and use the rendered calendar/list view to identify candidate dates. This is the current practical workaround for Andaz Papagayo.
- On MaxMyPoint pages, the calendar/list are JS-rendered. Use the browser tool (not raw terminal HTTP) and, if needed, switch number-of-nights via DOM (`document.querySelector('select').value = '4'` plus a change event) or the visible controls, then read the page text with `browser_console` / `browser_snapshot`.
- Treat MaxMyPoint as directional but useful: good for finding candidate Hyatt dates, not final booking proof. Use Hyatt itself only for sparse final verification when possible.
- If you need a cash benchmark for Hyatt cash-vs-points math and Hyatt's own rate tools are degraded, use Trivago as a directional fallback: first resolve the property with `trivago_search_suggestions`, then run `trivago_accommodation_search` for the exact stay dates and parse `Price Per Stay` / `Price Per Night` from the returned property card. Be aware the Trivago MCP may persist a very large result blob to `/tmp/hermes-results/...`; in that case, extract the specific hotel's JSON with terminal or read_file rather than trying to read the whole payload in chat.
- Trivago prices from this tool may come back in EUR even when searching from the US. For cpp math, convert to USD using a live FX source (for example the ECB daily EUR reference rate) and label the cash price as a directional benchmark, not a guaranteed checkout total.
- Be aware the current `@striderlabs/mcp-hyatt` package hardcodes brittle URL patterns like `/en-US/hotel/{id}` and `/en-US/hotel/{id}/rooms?...&usePoints=true`, while real Hyatt canonical URLs are often brand- and geography-specific. This can make `hyatt_get_hotel_details` fail systematically and can make `hyatt_redeem_points` return false empty results if it lands on a blocked or wrong page and simply finds zero room cards.
- The upstream Hyatt MCP also launches its own raw Playwright Chromium session instead of using Hermes browser backends. That means Hermes improvements like Browserbase, Browser Use, Camofox, or a connected CDP browser do not automatically help Hyatt MCP unless the MCP itself is patched to use them.
- When troubleshooting, distinguish four cases explicitly: real no-space, wrong-page parsing, anti-bot blocking, and browser-stack mismatch (Hyatt MCP bypassing the better Hermes browser backend). Empty arrays across multiple obvious properties (for example city Hyatt Places) are a red flag for tool failure, not credible inventory.
- In this environment, the host/browser egress IP can still inherit Christian's Tailscale exit node correctly while Hyatt blocks anyway. A successful Tailscale/home IP check does not prove the browser path is viable if the browser fingerprint still shows obvious automation signals like `HeadlessChrome`, `navigator.webdriver = true`, zero plugins, or missing `window.chrome`.
- If you need to remediate locally, patch the Hyatt MCP browser layer so it detects Hyatt block/error pages (`We’re sorry`, `Your browser did something unexpected`, `E6020`) and throws explicit errors instead of returning empty results. Also patch it to support canonical property paths/URLs rather than assuming `/en-US/hotel/{id}` works for every Hyatt brand.
- The next architectural fix after block-page detection is to patch Hyatt MCP so it can use an external browser backend instead of raw Playwright Chromium — ideally via CDP (`BROWSER_CDP_URL`/`HYATT_BROWSER_CDP_URL`) or another pluggable backend. Without that, stealth improvements elsewhere in Hermes will not materially help Hyatt searches.
- For bot-sensitive Hyatt work, the strongest agentic paths are: (1) persistent real Chrome over CDP, (2) Browserbase with residential proxies + advanced stealth, or (3) Camofox with persistence. Tailscale/home-IP routing helps reputation but is not sufficient by itself.
- Persistence matters. Fresh ephemeral sessions get challenged more often. If using Camofox or another self-hosted backend, prefer a persistent profile/session over one-off ephemeral launches.
- In Trippi's environment, a patched local Hyatt MCP can live at `/home/hermes/.hermes/mcp-servers/hyatt-patched`, and the travel profile config can point `mcp_servers.hyatt` to `node /home/hermes/.hermes/mcp-servers/hyatt-patched/dist/index.js`.
- The patched local Hyatt MCP now supports external browser handoff via `HYATT_BROWSER_CDP_URL` (preferred) or `BROWSER_CDP_URL`. When one of those env vars is set, Hyatt MCP connects to that browser over Chrome DevTools Protocol instead of launching its own raw local Playwright Chromium. This is the clean path for using a stealthier persistent browser backend.
- After changing MCP server config or browser env for Hyatt, remember Hermes loads MCP servers at startup, so the running chat will not use the patched server until Hermes is restarted.
- A headed Chrome session connected over CDP can materially improve the browser fingerprint versus local headless Playwright (`webdriver=false`, non-Headless UA, plugins present, `window.chrome=true`), but Hyatt/Kasada may still serve a blank challenge shell instead of real hotel content. Better fingerprint is necessary, not sufficient.
- Treat `hyatt_redeem_points` results during that failure mode as directional, not definitive, unless corroborated elsewhere.
- If you need to answer a question like “are there any 4-night award runs this year?”, a reusable fallback is to sweep weekly or daily windows across the requested range and report the findings plainly, while labeling the result as non-final if the other Hyatt tools are degraded.
- A stronger workflow for date-finding is:
  1. discover candidate dates from a third-party Hyatt calendar source such as MaxMyPoint
  2. verify only the promising dates in a real Hyatt browser session or the strongest available agentic browser backend
  3. prefer Christian's own local Chrome profile for final truth if server-side verification remains blocked
- For date-finding tasks like “find me 4 nights at Andaz Papagayo,” a good reusable workflow is: find the MaxMyPoint property page via DuckDuckGo (`site:maxmypoint.com <property>`), use that for candidate windows, then confirm those windows on Hyatt. Do not represent MaxMyPoint alone as final bookable truth.

## Safety rules

- Never transfer points or book without explicit approval.
- Never present scraped award space as guaranteed until confirmed on Hyatt.
- If login is unavailable, say what you can still determine without it.
- If Hyatt tooling is degraded, explicitly distinguish between a directional scan and a definitive availability answer.
