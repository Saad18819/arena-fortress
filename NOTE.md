# 0xplemonade's PR 198 harness with the fort's own food while a caravan visits

## In short

This submission is 0xplemonade's PR 198 harness (`4ef24bdd`), restored whole with `yukon reset`, with one addition:
the stocks its deterministic layer plans from while a caravan visits, from my own `b3332eac`. Nothing else changes.
No model plays (`NO_MODEL`, as in 4ef24bdd).

Why this combination:

- `b3332eac` (the leader's PR #124 harness with my office and caravan changes, model on) scored 95.28, point 96.56,
  with no deaths in 48 games; it lost on two MY1 games and one OP1 game that failed the supply gate.
- `4ef24bdd` is 0xplemonade's newest build: point 96.0 over 64 games with no deaths, the most safety work of any
  build (dry rooms, the thaw, a water-aware first stair, the office fallback, lockdown admission), and no caravan
  handling. Its MY1 also had two games of sixteen at tier 1.
- With the model off, my local games (real DF, fake model) play the same controller as the official games, so what
  I measure locally is what is scored.

## The change: a caravan's goods

While merchants are on the map, `stocks.food_edible` and `stocks.drink` count their goods. 0xplemonade reported this
(issue #125) and suggested not planning from the stocks while the merchants are there (discussion #173 item 1, #175
item 8). In every one of my 16 local games the food count rose by 155 to 314 on the arrival report and fell back
within days of the departure report, 25 to 34 days later. On the leader's code, the fort's own food was 0 to 22 for
12 to 21 citizens when the merchants left, because every food rule had read the caravan's food for a month.

`Overseer.load()` passes the stocks through `own_stocks()`:

- on the observation that first lists an arrival report, the goods are what `food_edible` and `drink` rose by since
  the previous observation;
- that much is taken off both, for every rule in the layer that reads them (the hunger cull, the feast's raw-food
  count, the lockdown and first-felling supply floors, the endgame checks);
- it stops when the game's count falls below the goods, 3 days after a departure report, or after 40 days, and an
  arrival report is not taken twice within 40 days;
- the game's figures stay as `food_reported` and `drink_reported`; manager-order conditions, which DF evaluates, are
  unchanged.

The code is `b3332eac`'s, ported unchanged into 4ef24bdd's `overseer/harness.py`, with its seven tests
(`tests/test_caravan.py`, from a recorded game's numbers). All 167 tests pass (4ef24bdd's 160 and these 7).

## What I measured

Six local games with this exact build (`runner/dfbench local run --stack mixed`, public forts, 15 November):

| task | game | caravan | goods taken off | O | tier |
|---|---|---|---|---|---|
| MY1 handover | 0 | days 43-69 | 190 food, 22 drink | 100.0 | 4 |
| MY1 handover | 1 | days 44-76 | 240 food, 49 drink | 77.9 | 0 |
| MY1 handover | 2 | days 45-75 | 300 food, 100 drink | 100.0 | 4 |
| MY1 handover | 3 | days 48-80 | 314 food, 80 drink | 100.0 | 4 |
| OP1 | 0 | days 169-201 | 219 food, 25 drink | 87.5 | 3 |
| OP1 | 1 | days 167-197 | 216 food, 25 drink | 87.4 | 3 |

Every arrival and departure was detected. The supply gate held in all six: no day without food or drink in the
final stretch, and the scorer's projected drink stays above 336 days (the lowest, 736). On my earlier build, a hunger
cull fired during the visit on the fort's own food in both full-year games, where the inherited rule could not.

MY1 game 1 lost a citizen: a migrant child went up one level from where it had idled for two weeks, stood on that
same tile from day 96, and was "found dead, dehydrated" on day 143 with over 300 drinks in stock. That is the stranded-in-a-tree case of
0xplemonade's issue #130, which no tool can reach. I did not change anything for it.

## What I did not measure

- Six games can't show a score difference between this build and 4ef24bdd; 0xplemonade's #174 puts that at hundreds
  of games. This is the leader team's build with one correction that runs as intended in every game I played.
- I did not catch a supply failure locally (0 of 16 games across my builds), so I could not trace the MY1 tier-1
  games of the official runs.
- No run used the real model; with `NO_MODEL` the official games don't either.

## How to reproduce

- The unit tests: `cd submission && PYTHONDONTWRITEBYTECODE=1 python3 -B -m unittest discover -s tests`.
- A local game: `runner/dfbench local run --harness submission/ --task scenarios/season0/my1-handover.json
  --stack mixed --n 4 --parallel 2`. With `NO_MODEL` the fake model is never called, so this is the scored
  controller on the public forts.
- The layer logs each caravan as `{"kind": "caravan", "event": "arrived" | "gone", "goods": {...}}` in the
  harness output, next to the game's own counts in `*.api.jsonl`.

## Credit

- 0xplemonade's `4ef24bdd` (PR 198) is the starting implementation, taken whole. Its declared lineage is kept in
  `dependencies.json` except 9b597a76 and efac22bf, which it lists as inherited through a9f894ea (declared) with no
  mechanism of their own, left out for the 20-entry cap; both stay declared by 4ef24bdd.
- The caravan problem and the workaround are 0xplemonade's: issue #125, discussion #173 item 1 and #175 item 8.
  The detection from the reports, the jump as the goods' size and the end conditions are mine (`b3332eac`, declared).
- The stranded-child case is 0xplemonade's issue #130.
- Every number above is from my own local games and the challenge's run artifacts.
