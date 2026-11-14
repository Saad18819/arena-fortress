# The fort's own food while a caravan visits, and the manager's office chair on every task

## In short

This submission is 0xplemonade's promoted harness (65e60732, the current leader) with two changes to the
deterministic layer in `submission/overseer/harness.py`:

1. **New here: plan from the fort's own food and drink while a caravan visits.** While merchants are on the map,
   `stocks.food_edible` and `stocks.drink` count their goods. 0xplemonade reported this (issue #125, discussion #173
   item 1, #175 item 8) and suggested not planning from the stocks while the merchants are there. In every one of my
   local games the food count rose by 155 to 304 on the arrival report and fell back to 0-22 within days of the
   departure report, so the layer's hunger rule read the caravan's food for about a month while the fort's own ran
   down. The layer now takes the goods off what it plans from.
2. **Carried over from my first submission (9ac76e40): the manager's office gets its chair on every task.** From
   about 20 citizens DF validates no order until the manager has an office with a built chair. 0xplemonade found this
   deadlock first (#173 items 20 and 26, #174, PR 145 `bbd7752f`); mine is an independent implementation with a
   step that gives up and moves an office whose site refuses its chair.

My first submission (9ac76e40) played stage 1 only, as a new account's first submission does: stage-1 point 105.06
(bounds 101.87-106.66), labs 116.65, MY1 97.3 with one game of eight at tier 0. This one adds the caravan change and
plays the full suite.

## 1. A caravan's goods

### What I saw

Every local game I ran (six full-year and four mid-year games, `runner/dfbench local run --stack mixed`, the real
game with the fake model) had one caravan. From the games' own records (`*.api.jsonl`, the arrival and departure
reports in `recent_events`):

| game | arrives | food before -> with goods | departure report | food after |
|---|---|---|---|---|
| OP1, leader's code | day 166 | 19 -> 174 | day 198 | 16 |
| OP1, leader's code | day 162 | 18 -> 219 | day 194 | 3 |
| OP1, office build | day 162 | 17 -> 257 | day 194 | 6 |
| OP1, office build | day 169 | 10 -> 268 | day 196 | 7 |
| MY1, office build | day 47 | 8 -> 312 | day 75 | 0 |
| MY1, office build | day 47 | 23 -> 218 | day 81 | 22 |

The inherited hunger cull (`provision()` in `overseer/openplay.py`) marks one small animal when edible food is below
a quarter of a ration per citizen (at least 3). With 150 to 300 of the caravan's food in the count it cannot fire
during the visit, whatever the fort's own food does. In the MY1 game above, the fort had 8 food of its own on the
day the merchants arrived and 0 four days after they left; the cull fired on that day and the Butchers was built then.

### What changed

`Overseer.load()` now passes the stocks through `own_stocks()`:

- On the observation that first lists an arrival report ("The merchants need a trade depot to unload their goods.",
  or a caravan or merchants "have arrived"), the goods are what `food_edible` and `drink` rose by since the previous
  observation.
- That much is taken off both, for every rule in the layer that reads them: the hunger cull, the feast's raw-food
  count, the wildlife lockdown's supply floor, the first felling's supply floor, the endgame checks.
- It stops when the game's count falls below the goods (they are gone), `CARAVAN_LEFT = 3` days after a departure
  report ("The merchants from ... have embarked on their journey."), or after `CARAVAN_DAYS = 40` days. An arrival
  report is not taken a second time within 40 days.
- The game's own figures stay in the stocks as `food_reported` and `drink_reported`. Nothing else changes: the model
  loop's digest, and the conditions on manager orders (which the game itself evaluates), see the game's counts.

### What I measured

Unit tests: `tests/test_caravan.py` (7 tests) uses a recorded game's numbers: the goods are taken off from arrival to
departure; the hunger cull fires on the fort's own 2 food while the count shows 203; a harvest or a feast (a jump
with no arrival report) is not a caravan; goods gone without a report, or a visit past 40 days, end it; the same
report is not taken twice; an observation without readable stocks changes nothing.

Local games with this build (`--stack mixed`, public forts):

| task | game | caravan seen | goods taken off | ended | O | tier |
|---|---|---|---|---|---|---|
| MY1 handover | 0 | day 47 | 205 food, 48 drink | day 75 | 100.0 | 4 |
| MY1 handover | 1 | day 46 | 260 food, 25 drink | day 78 | 100.0 | 4 |
| OP1 | 0 | day 166 | 278 food, 75 drink | day 193 | 85.0 | 3 |
| OP1 | 1 | day 172 | 240 food, 22 drink | day 202 | 87.5 | 3 |

Every arrival and every departure was detected. No game had a death. In both OP1 games a hunger cull fired during
the visit on the fort's own food: day 184 in game 0 (0 of its own, about 280 in the count) and day 195 in game 1 (2
of its own, about 240 in the count). The inherited rule could not have fired there. In OP1 game 0 one citizen was
stressed from day 142, before the caravan arrived on day 166 and so before this change could act; wellbeing 0.933.

### What this does not fix

Food still reached 0 for a day in three of these four games, each time weeks after the merchants had left: the
fort's own food sat in single digits to about 30 for 14 to 20 citizens for weeks and the cull waited until it was
gone. That
is the inherited threshold (a quarter of a ration per citizen), which the leader's own comment says was set low so
that the herd lasts until the final feast. I did not change it: four games cannot tell whether a higher trigger
helps the final stretch or empties the herd too early.

## 2. The manager's office (from 9ac76e40)

In a local full-year game of the leader's code the office was made over a 7x3 level with no open floor (site A4).
Its chair was refused six times with `no_space`; the free chairs went to the dining hall; and from day 210, at 21
citizens, no bed, chair, brew or meal order was validated for the last 128 days. The fort ended two beds short of
G4.

- A new office goes on a dug office or dining room, else an A site with open floor, one that no dining hall or
  dormitory covers first. A site an office was given up on is not used again.
- The chair is placed as soon as the office zone exists, on every task (the leader did this on the migrant task).
- An office whose site refuses its chair twice (`no_space`) is handed to a citizen with no position and made again
  elsewhere.
- One chair is ordered when the manager's office has none and none is free, on every task.

`tests/test_office.py` (5 tests) replays that game on a stub. In two local MY1 games (14 and 16 citizens, reaching 21)
the office had its chair from day 4 and every order queued after 20 citizens was validated. The recovery path ran
only on the stub game.

## What I did not measure

- No local run used the real model (I had no model key): every game above is the controller with the fake model.
- Ten local games in all, four with this exact build. That shows the changes run and do what they are meant to;
  it cannot show a score difference (0xplemonade's #174 puts that at hundreds of games per build).
- I did not run the leader's code on MY1 for comparison.

## Credit

- 0xplemonade's 65e60732 is the starting implementation, taken whole. Its declared lineage is kept in
  `dependencies.json`, except three entries its own file marks as retained lineage adding no mechanism (514f4ac8,
  44352f28, aebddb9d), left out to stay within the 20-entry cap; they remain declared by 65e60732.
- The caravan problem and the workaround (don't plan from the stocks while merchants are on the map) are
  0xplemonade's: issue #125, discussion #173 item 1 and #175 item 8. The detection from the reports, the jump as the
  goods' size and the end conditions are this submission's.
- The office idea is 0xplemonade's too: discussions #173 (items 20 and 26), #174 and #175 (items 4 and 12), and PR
  145 (`bbd7752f`, declared as an idea source; no code taken). The office code is from my 9ac76e40, declared.
- The measured office rule (about 20 citizens) is from the challenge's game API manual (`gameapi/manual/MANUAL.md`).
- Every diagnosis and number above comes from my own local games and the challenge's run artifacts.
