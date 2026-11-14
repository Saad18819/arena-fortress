# The manager's office gets its chair on every task

## In short

This submission is 0xplemonade's promoted harness (65e60732, the current leader) with one change: the steps that
give the manager an office. From about 20 citizens Dwarf Fortress validates no work order until the manager has an
office zone with a built chair, assigned to them (the game API manual records this as measured in game). The order
for the office's own chair is a work order too, so a fort that reaches 20 citizens without that chair in place can
not get it any more: from then on no new order is validated at all.

I found this in a local full-year game where it cost the last 128 days of the year. The change makes the office
land on floor that can take a chair, puts the chair in as soon as the zone exists on every task, and recovers when
an office's site refuses its chair.

**This is not a new finding.** 0xplemonade found the same deadlock first and wrote it up in discussion #173 (item 26,
"From about 20 citizens the manager validates no orders without an office chair", and item 20, an office over a farm
level whose chair got `no_space` at every try), listed it as test triggers in #175 (items 4 and 12), and shipped an
office set up from the start, on a dug room, never on a farm level, with its chair first, in PR 145 (`bbd7752f`, see
#174). I read those threads after diagnosing my own game. This submission is an independent implementation of the
same idea on top of the leader's code, with one addition: an office whose site still refuses its chair is given up,
handed to another citizen, and made again elsewhere. No code is taken from `bbd7752f`.

## What I saw

I ran the leader's harness locally with `runner/dfbench local run --stack mixed` (the real game with the fake model,
so only the deterministic controller in `overseer/` makes decisions) on the public OP1 start fort, seed 12345, twice.

| game | citizens at day 336 | Hamlet | placed beds | O | tier |
|---|---|---|---|---|---|
| OP1-0 | 13 | no | 15 | 87.5 | 3 |
| OP1-1 | 21 | day 209 | 19 | 89.7 | 3 |

OP1-1 missed G4 (placed beds >= citizens) by two beds. Following the game's own records:

- Day 117, 16 citizens: `office()` made the office with `make_zone office` over site A4, a 7x3 A site at z 105
  (88,92) with no open floor, then assigned it to the manager on day 118.
- Days 129-148: `place_furniture chair` on A4 was answered six times with `no_space`: "no free floor tile in
  88,92,105 (7x3): every tile is wall, undug, built on or designated". The `office_chair` key then ran out of
  attempts. Two free chairs waited from day 148.
- Day 208: the second migrant wave brought the fort to 21 citizens; the furnishing step placed the two free chairs in
  the dining hall and queued 5 beds, 3 tables and 3 chairs (orders 14-16).
- Day 210 to the end: orders 14-16 stayed `validated: false, active: false`. So did every order added afterwards:
  the daily brew, barrel and meal orders queued on day 273 (orders 17-19) and a meal order on day 329 (order 20). The
  manager's office reported `{"chair": false}` throughout. Beds stayed at 19 for 21 citizens for 128 days.

In OP1-0, which stayed at 13 citizens, every order was validated within two days. The orders already standing before
day 210 kept running, so drink and food held, and supply and wellbeing still scored 1.0 in that game. A fort whose
standing brew or meal order finishes, or needs replacing, after the lock would have no way to queue another one.

Why the office went there: the inherited `office()` picks `self.dug_rooms("office") + self.dug_rooms("dine") +
self.kind("A")` and takes the first. No room had those roles, so it took the first A site, whatever its open floor.
When the chair was refused it kept retrying the same site. The chair was placed straight after `make_zone` only on
the migrant task (`self.surge`), and the extra office chair was ordered only through the bed-only queue of the
migrant task (`beds_first`).

## What changed

All in `submission/overseer/harness.py`:

1. **Where a new office goes** (`office_site()`): a dug office or dining room as before; else an A site with
   `open_tiles > 0`, preferring one that no dining hall or dormitory covers (an office laid over the dining hall would
   take its chairs as the office's), then the most open tiles. A site an office was given up on is skipped.
2. **The chair right away, on every task**: after an accepted `make_zone office`, a free chair is placed in it on
   every task, not only the migrant task. This is i34-9's office-first sequence (fa9c13cc), which the leader kept
   behind the migrant flag.
3. **Recovery** (`office_chair()`, `drop_office()`): a chair refused with `no_space` counts against that office;
   after `OFFICE_FAILS = 2` refusals the office is given up. If the manager still owns it, it is assigned to an
   adult citizen with no position, so the manager has no office on the next look, and a new one is made on another
   site and given its chair. A given-up office is never assigned back to the manager. If nobody can take it over, a
   new office is made beside it.
4. **The chair is ordered on every task**: when the manager's office has no chair and no chair is free,
   `supply()` queues one `make_chair` (the leader did this only in the migrant task's bed-only queue). It uses the
   leader's own `office_chair_needed()` test, so it stops as soon as a chair is free or placed.

Nothing else in `submission/` changes apart from a new section at the top of `submission/README.md`, the new test
file and the dependency declaration. No prompt, model-loop, digest or open-play rule is touched.

## What I measured

Unit tests: `tests/test_office.py` (5 tests) replays the recorded failure on a stub game: the no-floor site, two
refusals, the hand-over to a citizen with no position, the new office on open floor and its chair. It also checks
that a new office prefers open floor away from the dining hall, that the dining level is only the last choice, and
that nothing happens below 14 citizens or once a chair is in. All 28 tests in `tests/` pass.

Local games with this build, all `--stack mixed` (real DF 53.16, fake model), public forts:

| task | game | citizens | office chair in by | orders validated after 20 citizens | O | tier |
|---|---|---|---|---|---|---|
| OP1 | 0 | 13 | (never needed) | (never reached 20) | 87.5 | 3 |
| OP1 | 1 | 12 | (never needed) | (never reached 20) | 87.5 | 3 |
| MY1 handover | 0 | 14 -> 21 on day 64 | day 4 | all | 100.0 | 4 |
| MY1 handover | 1 | 16 -> 21 on day 90 | day 4 | all | 100.0 | 4 |

In both MY1 games the office was made on day 0 on open floor, its chair was planned in the same pass, and the
manager's office reported `chair: true` from day 4. Every order queued after the fort passed 20 citizens was
validated within 0-9 days.

## What I did not measure

- No local run used the real model: I had no model key. The fake-model runs exercise the controller only.
- The recovery path (giving up an office) ran only on the stub game. None of my four local games with this build
  put an office on a site with no floor, so it was not triggered in game.
- Two OP1 games and two MY1 games are far too few to show a score difference. The MY1 games may have scored 100
  with the leader's code as well; I did not run the leader on MY1 for comparison. This is a robustness fix for a
  failure I recorded, not a measured score gain.
- Migrant waves vary: the two OP1 games here reached only 12-13 citizens, so the office rule never applied in them.

## Something I noticed but did not change

In one OP1 game with this build, edible food (`food_edible`) fell to 4-6 between days 224 and 280 with 12 citizens, recovering
with the autumn harvest (194 at day 336). The leader's official run lists two OP1 games at tier 1 out of 16. I
cannot see the official games, but a summer food dip like this one is a candidate for some of those. I left it alone
here so that this submission changes one thing.

## Credit

- 0xplemonade's 65e60732 is the starting implementation, taken whole. Its declared lineage is kept in
  `dependencies.json`, except two entries its own file marks as adding no mechanism (514f4ac8, 44352f28), left out
  to stay within the 20-entry cap; they remain declared by 65e60732.
- The idea of setting the office up on floor that takes a chair, chair first, is 0xplemonade's: discussions #173
  (items 20 and 26), #174 and #175 (items 4 and 12), and their PR 145 (`bbd7752f`, declared in `dependencies.json`
  as an idea source; no code taken). The diagnosis and the measurements in this note are from my own local games.
- The office rule this changes is the leader's `office()`, which carries i34-9's office-first chair placement
  (fa9c13cc) and GordoAR's earlier office repair (5dfdcf15). Both are in the retained declarations.
- The measured rule that DF stops validating orders from about 20 citizens without an office with a chair is from the
  challenge's game API manual (`gameapi/manual/MANUAL.md`, "The manager's office").
- The diagnosis used the challenge's own run artifacts (`*.api.jsonl`, `*.daily.json`) from local mixed-stack games.
