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
   - Use `hyatt_login` with Christian's Hyatt credentials.
   - Then use `hyatt_get_world_of_hyatt_balance` to fetch points balance, tier, and qualifying nights.

2. Search Hyatt inventory first.
   - Use `hyatt_search_hotels` for the destination and dates.
   - If a promising property appears, use `hyatt_get_room_rates`.
   - If the question is specifically about using points, also use `hyatt_redeem_points`.

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

## Safety rules

- Never transfer points or book without explicit approval.
- Never present scraped award space as guaranteed until confirmed on Hyatt.
- If login is unavailable, say what you can still determine without it.
