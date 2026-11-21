# CallBook (Intelligent Contract)

A public track record for predictions. Write a prediction, the web page that will settle it, and exactly what counts as a **hit** and a **miss**. The conditions are locked on-chain, so nobody can move the goalposts later. When the time comes, **anyone** can ask GenLayer validators to read the page and settle the call. Every wallet builds a **public hit rate** that other contracts can read.

Anyone can say "I called it". CallBook makes that a fact many independent validators agreed on, with the exact words from the page as proof.

- **Contract (GenLayer Studio):** `0xf144E1D206ccD6bB644C8Da39e886a5892ae119c` (v1.1; the earlier instance `0x0C2ede87B616219cF9487648f3D6F92D4C9128ca` is v1.0)
- **Contract source:** [`callbook.py`](./callbook.py)
- **Tests:** [`tests/test_callbook.py`](./tests/test_callbook.py)

## How it works

```
make_call(statement, url, hit_when, miss_when) --> OPEN --resolve--> HIT   (final)
                                                     |   \--------> MISS  (final)
                                                     +-- page shows neither / unreachable: stays OPEN
```

1. `make_call` stores the prediction (8 to 200 characters), a public https page, and two fixed conditions: what the page must show for a HIT and what it must show for a MISS.
2. `resolve(id)` (**anyone**): validators fetch the page (`gl.nondet.web.render`), a model reads it against the two conditions, and validators must agree on the verdict (`gl.eq_principle.prompt_comparative`, the `verdict` field must be identical).
3. A HIT or MISS is only kept if the model's **quote really appears on the page**. A made-up or unmatched quote turns the verdict into OPEN. A settled call is final.
4. If the page shows neither outcome yet, or cannot be read, the call stays OPEN and can be tried again. Attempts are limited per wallet (3 each), never globally, so no one can use up a call's chance to settle: a different wallet can always try again.
5. `get_record(author)` returns a wallet's hits, misses, open calls and hit rate in percent.

## What it protects against

| Risk | How CallBook handles it |
|---|---|
| Moving the goalposts after the event | Hit and miss conditions are fixed when the call is made. |
| Author picking a convenient moment to settle | Anyone can settle, not only the author. |
| A model inventing proof | A HIT or MISS needs a quote that appears word for word on the page, checked in code. |
| Prompt injection from the page or the text fields | The statement, both conditions and the page are each wrapped in their own tag with `<` and `>` escaped, and the prompt says never to follow instructions inside them. |
| A flaky page deciding a call | An unreadable or empty page never settles anything. |
| Validators fetching internal addresses | Only `https` links to public hosts. `localhost`, private and link-local IPs, `.local`/`.internal`, credentials and IPv6 literals are refused. |
| Spam | 30 calls per wallet, no duplicate call on the same page, 3 settle attempts per wallet per call (no shared cap, so early or failed attempts can never block settlement), bounded text and page size. |

## Read API (composable)

- `get_call(call_id)` returns the call as JSON, including the verdict quote
- `get_calls_by_author(author)` returns a wallet's calls
- `get_record(author)` returns hits, misses, open calls and `hit_rate_percent` (`-1` while nothing is settled)
- `get_recent(limit)` returns the newest calls from everyone
- `is_hit(call_id)` is `true` only for a settled HIT

## Try it in Studio

1. Deploy `callbook.py` (no constructor arguments).
2. Run `make_call` with a prediction, a public https page, a hit condition and a miss condition.
3. Run `resolve` with the call id. Wait for consensus: the call becomes `HIT` or `MISS` with a quote from the page, or stays `OPEN` if the page shows neither.
4. Read it back with `get_call`, `get_record(author)` and `is_hit`.

## Tests

`tests/test_callbook.py` runs the contract's real logic against a small stand-in for the `genlayer` library, with the web fetch replaced by a dictionary of pages and the model by a scripted reply. No node is needed:

```
python3 tests/test_callbook.py
```

It covers input and link refusal, duplicates and caps, HIT and MISS with a real quote, made-up quotes and garbage output staying open, unreachable pages and retries, the per-wallet attempt limit, 12 failed attempts by different wallets followed by a successful settlement, settled calls staying final, prompt isolation, track records and the feed (15 tests).

## v1.1: settlement can no longer be used up

A steward review found that v1.0 counted every `resolve` attempt, including early or failed ones, against a global cap of 8. Eight early attempts would have made a call impossible to settle later. v1.1 changes the lifecycle:

- There is no shared cap. A call stays resolvable for as long as it is OPEN.
- Each wallet may try a call up to 3 times (`MAX_TRIES_PER_CALLER`), tracked in the call's `tries` field. A different wallet can always try again.
- HIT and MISS are still final, and still need a quote that appears word for word on the page.

Live test of v1.1 (Studio, validators on):

| Step | Result |
|---|---|
| make_call 1, loose condition ("contains the words Example Domain"), resolve | validators answered OPEN with no quote; call stays OPEN, resolves 1, tries {wallet: 1} |
| make_call 2 ("This domain is for use in documentation examples"), resolve | HIT, quote "This domain is for use in documentation examples without needing permission.", tries {wallet: 1} |

Local tests added: one wallet uses all 3 tries and another wallet still settles the call; 12 failed attempts by 12 different wallets, then a successful settlement; a settled call cannot be resolved again.

## Limits

- The contract has no clock, so a call can be settled as soon as the page shows an outcome. Choose a page that only shows the outcome after the event.
- It only sees what validators fetch. Pages that need a login or build their text with scripts are not supported.
- The hit rate counts settled calls only and does not weigh how bold a call was.
- Studio test network only. No payments or enforcement. A web app for it lives in its own project repository.
