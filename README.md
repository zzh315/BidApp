# Bid Ladder

A one-page auction bidding assistant. You tell it the highest bid on the floor and what the room is
doing; it tells you whether to bid, what to bid, how many bids you have left before your limit, and
what that bid actually costs you once stamp duty and every other fee is added.

Built for Victorian residential auctions and the first-home-buyer stamp duty scale. No build step, no
dependencies, no server — one `index.html`.

---

## Why it exists

An auction gives you a few seconds to decide, and the two mistakes that cost real money are not
mistakes of arithmetic:

1. **Overshooting your limit** because the auctioneer's increments carry you past it.
2. **Running out of bids** — arriving at your ceiling with one move left while a rival still has four.

So the page leads with a hard stop pinned to the top of the screen and a count of bids remaining, and
puts the suggested number underneath. The recommendation is the smallest part of it.

---

## Deploying to GitHub Pages

This directory is already a git repository on branch `main`.

```bash
# 1. Create an empty repository on GitHub (no README, no .gitignore), then:
git add -A
git commit -m "Bid Ladder: auction bidding assistant"
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Then in the repository on GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a
branch → Branch: `main` / `/ (root)` → Save.**

The site appears at `https://<your-username>.github.io/<repo-name>/` within a minute or two.

**Open it once on wifi before you need it.** The page itself is self-contained, but the three
webfonts load from Google Fonts. Once cached it works with no reception; if the fonts never load the
page still works and falls back to system faces.

> **Do not enable Pages on a repository that holds anything private.** Pages serves *every file in the
> published branch* as a public URL. Pages from a private repository also requires a paid plan, and the
> published site is public regardless unless you are on GitHub Enterprise Cloud with access control.
> This repository deliberately contains nothing but the app.

---

## Where your numbers live

The committed defaults are a live setup, not a placeholder, and are what a fresh browser starts from.
Anything you change after that is kept in that browser's `localStorage`.

⚠️ **Committed defaults are public** — this repository included, because Pages on a free plan requires
a public repo. A bid ceiling in the defaults is a bid ceiling anyone can read. If that matters for a
given auction, reset the defaults to round placeholders and carry the real figures by prefill link.

Two ways to load your own figures:

- **Setup screen (⚙).** Enter them once; they persist in that browser's `localStorage`.
- **Private prefill link.** In Setup, tap **Copy private prefill link**. You get a URL with your
  numbers as query parameters. Bookmark it on your phone and the app opens ready to go. The app strips
  the parameters from the address bar after reading them.

Keep that link to yourself. It contains your limit, which is the one number you never put in writing.

Supported parameters: `name`, `open`, `reserve`, `target`, `ceiling`, `step`, `loan`, `gov`, `funds`,
`mtg`, `other`.

Unlike the form, **URL parameters are in whole dollars** (`ceiling=710000`, not `710`). The Copy button
writes them for you, so this only matters if you hand-edit a link.

---

## What you enter

**Every money field is in thousands.** Type `670` for $670,000 and `1` for a $1,000 increment;
decimals work, so `7.3` is $7,300. Each field echoes what it means in dollars directly underneath, and
turns red past $50m — so typing `670,000` out of habit shows `= $670,000,000` rather than silently
disabling your hard stop. Displays throughout the app stay in full dollars.

Government equity is a percentage and mortgages to register is a count; both are marked accordingly.

| Field | Meaning |
|---|---|
| **Opening bid** | Where the auctioneer starts, or the lowest bid you'd consider |
| **Reserve estimate** | Your read on the vendor's reserve. Drives the whole strategy — see below. `0` if unknown |
| **Target** | The number you want to win at |
| **Hard stop** | The number you will not cross. Never suggested past this |
| **Smallest increment** | The smallest step you think the auctioneer will accept, typically $1,000 |
| **Loan drawn** | Your loan amount |
| **Government equity %** | Shared-equity share of the price, e.g. `30` for Help to Buy on an existing home. `0` if not using a scheme |
| **Funds available** | Total cash you can deploy. Used to catch bids you cannot actually fund |
| **Mortgages to register** | Lender's mortgage, plus a second mortgage if a shared-equity scheme registers one |
| **Other costs** | Conveyancing and disbursements, building and pest, rates and water adjustments, first-year insurance, lender settlement fees, contingency |

---

## The decision logic

Checked in priority order. The first rule that matches wins.

| Condition | Verdict |
|---|---|
| Floor is at or past your hard stop | **STOP** |
| The highest bid is yours | **SAY NOTHING** — never bid against yourself |
| Last bid was a vendor bid, reserve not met | **HOLD** — that is the auctioneer bidding for the vendor, not a competitor |
| Suggested bid leaves your funds short | **STOP** — your real limit is below your hard stop |
| Suggested bid is past target, no live rival | **HOLD** — don't cross your target on momentum alone |
| Suggested bid is past target, rival is live | **BID**, flagged amber |
| Otherwise | **BID** |

### How the increment is sized

Three regimes, because the right step depends entirely on whether the property can currently sell.

**Pass-in play** — below your reserve estimate, one rival or fewer. Below the reserve the property
*cannot be sold*, so the only prize on offer is being the highest bidder when it is passed in, which
carries the exclusive right to negotiate. Step: the minimum increment. Bid as slowly and cheaply as
the auctioneer allows.

**Shake-out** — below the reserve with two or more live rivals. Now small steps work against you:
being ground upward in $1,000s just walks the vendor to their reserve for free. Step: confident and
round, about 0.7% of the current bid.

**Real contest** — reserve met, announced on the market, or two-plus rivals above the reserve. It will
sell. Step: sized so roughly four more bids fit between here and your hard stop, because the bidder
with rungs left wins the last one.

In the pass-in and contest regimes a suggested bid landing on a round multiple of five increments is
nudged down by two increments. That is not superstition about round numbers rattling anyone — it is
increment arithmetic. From $700,000 with a $710,000 ceiling, $10,000 steps give you one more bid;
$2,000 steps give you five.

---

## The cost model

**Stamp duty** — Victorian first-home-buyer principal-place-of-residence scale: nil to $600,000,
tapered from $600,001 to $749,999, full general duty above. The taper is
`general_duty(price) × (price − 600,000) ÷ 150,000`, which makes the marginal duty rate inside the
taper band roughly 25–32c per extra dollar of price — the most expensive money in the schedule.

General duty for the $130,001–$960,000 band is `$2,870 + 6% of the excess over $130,000`.

**Land transfer registration** — `$104.30 + $2.34 per whole $1,000`, capped at `$3,614.00`
(Land Services Victoria, 2026‑27 schedule).

**Mortgage registration** — `$129.20` per mortgage, electronic lodgement.

Then:

```
all-in cost   = price + duty + transfer registration + mortgage registration + other costs
your cash out = (price − loan − government equity) + all of those costs
cash left     = funds available − your cash out
```

`Cash left` turns amber below 12% of your funds and red when negative. A negative value overrides the
verdict to **STOP**.

Fee schedules change on 1 July each year, and duty scales change with state budgets. Check the figures
against the State Revenue Office and Land Services Victoria before relying on them for a real bid.

---

## Not advice

Guidance computed from numbers you type in. It has no knowledge of the property, the contract, the
vendor, or your finances beyond those inputs, and it cannot see the room. Nothing here has been
reviewed by a lawyer, conveyancer or licensed adviser.
