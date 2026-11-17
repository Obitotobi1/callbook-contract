# CallBook (Intelligent Contract)

A public track record for predictions. Write a prediction, the web page that will settle it, and exactly what counts as a **hit** and a **miss**. The conditions are locked on-chain, so nobody can move the goalposts later. When the time comes, **anyone** can ask GenLayer validators to read the page and settle the call. Every wallet builds a **public hit rate** that other contracts can read.

Anyone can say "I called it". CallBook makes that a fact many independent validators agreed on, with the exact words from the page as proof.

- **Contract (GenLayer Studio):** `0x0C2ede87B616219cF9487648f3D6F92D4C9128ca`
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
4. If the page shows neither outcome yet, or cannot be read, the call stays OPEN and can be tried again (up to 8 attempts).
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
| Spam | 30 calls per wallet, no duplicate call on the same page, 8 settle attempts per call, bounded text and page size. |

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

It covers input and link refusal, duplicates and caps, HIT and MISS with a real quote, made-up quotes and garbage output staying open, unreachable pages and retries, the resolve limit, prompt isolation, track records and the feed (13 tests).

## Limits

- The contract has no clock, so a call can be settled as soon as the page shows an outcome. Choose a page that only shows the outcome after the event.
- It only sees what validators fetch. Pages that need a login or build their text with scripts are not supported.
- The hit rate counts settled calls only and does not weigh how bold a call was.
- Studio test network only. No payments or enforcement. A web app for it lives in its own project repository.
