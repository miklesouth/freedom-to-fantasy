# Freedom to Fantasy — Draft Wheel: Randomness & Fairness

*A short methodology note for the mathematically curious (hi, Mitch).*

## TL;DR

The draft order is a **uniformly random permutation** of the ten teams: every one of the
10! = 3,628,800 possible orderings is equally likely. Each team's own draft slot is marginally
**Uniform{1, …, 10}** with expected value 5.5, **regardless of when that team spins** (first,
last, or anywhere between). Spinning early or late confers no advantage, and no one — including
the commissioner — can predict or reproduce the outcome in advance.

The one honest caveat: the generator is the browser's built-in PRNG (`Math.random`), which is
statistically uniform but **not cryptographic and not seed-auditable after the fact**. If you
want provable-after-the-fact fairness, see "Optional upgrades" at the bottom. For a church-league
draft, the property above is more than enough.

---

## 1. Where the randomness comes from

Every consequential draw is a single call to JavaScript's `Math.random()`, evaluated in the
spinner's own browser. `Math.random()` returns a double-precision float uniform on the interval
[0, 1). In V8 (Chrome/Edge) and modern Safari/Firefox it's an `xorshift128+` generator, which
passes the standard empirical randomness batteries (e.g., TestU01) for non-cryptographic use.

There is **no fixed seed**. The engine seeds it from the environment at page load, so outcomes
are neither predictable beforehand nor reproducible afterward.

## 2. The selection algorithm — sampling without replacement

When a manager spins, the page selects a slot uniformly at random from the slots that are
**still open**, then records it. That's the entire algorithm:

```js
const taken = Object.keys(current).map(Number);        // slots already claimed
const open  = [];
for (let p = 1; p <= N; p++) if (!taken.includes(p)) open.push(p);  // N = 10
if (!open.length) return;                              // board full → abort

const slot = open[Math.floor(Math.random() * open.length)];  // uniform over remaining
current[slot] = myTeam;                                // claim it
```

`Math.floor(Math.random() * k)` yields an integer in {0, …, k−1}. Each successive spin draws from
a strictly smaller pool, so the same slot can never be handed out twice — this is
**sampling without replacement**, done incrementally as people arrive.

### Rounding/modulo bias is negligible

`Math.random()` draws from 2^53 equally likely values. `floor(x·k)` partitions [0,1) into `k`
buckets; each bucket holds either ⌊2^53 / k⌋ or ⌈2^53 / k⌉ of those values, so the maximum
relative bias between buckets is ≤ k / 2^53. For k ≤ 10 that's about **1.1 × 10⁻¹⁵** — roughly
fifteen orders of magnitude below anything a ten-person draft could detect.

## 3. Why the result is a uniform permutation

Fix any arrival order of the ten teams (whoever happens to click first, second, …). Assigning
each arriving team a uniformly random *unused* slot is exactly the incremental construction of a
random permutation (the same principle as a Fisher–Yates shuffle). For any specific complete
assignment, the probability telescopes:

```
P(assignment) = (1/10) · (1/9) · (1/8) · … · (1/1) = 1 / 10!
```

Since **every** complete assignment has probability 1/10!, the distribution over orderings is
uniform. Two consequences worth stating explicitly:

- **Arrival order doesn't matter.** The 1/10! result holds for *any* fixed arrival order, and the
  actual arrival order (human click timing) is independent of the RNG. A mixture of identical
  uniform distributions is still that uniform distribution — so who spins when has no effect on
  the distribution of the final order.
- **No timing advantage.** By symmetry, each team's marginal slot is Uniform{1,…,10} no matter its
  position in the arrival sequence. The manager who deliberately spins last is, marginalized over
  everyone else's draws, still equally likely to land on any of the ten slots. You cannot improve
  your expected pick by waiting or by rushing.

## 4. The spinning animation is cosmetic

The slot is decided **first**, by the code in §2. The reel animation is then *told* which number
to land on and simply scrolls to it; the filler numbers it flashes on the way are throwaway and
never touch the result:

```js
const cells = [];
for (let i = 0; i < 32; i++) cells.push(1 + Math.floor(Math.random() * N)); // decorative only
cells.push(finalNum);   // finalNum was already chosen by the transaction
```

So there's no "stopping the wheel at the right moment," no skill, and no way for the visual to
override the draw.

## 5. Concurrency: no collisions, even on simultaneous spins

This isn't randomness, but it's the other half of "fair," so it's worth documenting. The
selection in §2 runs inside a **Firebase Realtime Database transaction**. Transactions are applied
atomically and serialized by the server: if two people spin in the same instant, one commits and
the other's transaction function automatically re-runs against the updated board (now with one
fewer open slot) and draws again. The net effect is identical to the two spins having happened one
after the other, so two managers can never be assigned the same pick.

## 6. Honest limitations

- **Not a CSPRNG.** `Math.random()` is not cryptographically secure. Nobody here is adversarially
  attacking a PRNG's internal state to steal pick #1, so this is immaterial in practice — but it's
  true, so it's stated.
- **Not reproducible / not independently auditable.** Because there's no recorded seed, you can't
  recompute the draw later to prove it wasn't tampered with. The integrity guarantee is
  operational (the live shared board everyone watched fill in), not cryptographic.
- **Team selection is on the honor system.** The page trusts each person to pick their own team
  from the dropdown; nothing technically stops someone from selecting another team. Social
  enforcement only.

## 7. Optional upgrades (if Mitch insists)

**Cryptographic RNG.** Swap `Math.random()` for `crypto.getRandomValues` with rejection sampling
to get an unbiased integer with a CSPRNG source:

```js
function randInt(n) {                       // unbiased uniform integer in [0, n)
  const limit = Math.floor(0x100000000 / n) * n;   // largest multiple of n ≤ 2^32
  const buf = new Uint32Array(1);
  let x;
  do { crypto.getRandomValues(buf); x = buf[0]; } while (x >= limit);  // reject the tail
  return x % n;                             // now exactly uniform, zero modulo bias
}
```

**Provable fairness (commit–reveal).** If you want after-the-fact verifiability, publish a hashed
commitment to a seed before the draw, reveal the seed afterward, and derive the whole permutation
deterministically from it — anyone can then recompute and check. Overkill for this league, but the
door is open.

---

*Generated for the Freedom to Fantasy 2026 draft. The draw itself is in `draftwheel.html`; the
relevant lines are the `ref.transaction(...)` block.*
