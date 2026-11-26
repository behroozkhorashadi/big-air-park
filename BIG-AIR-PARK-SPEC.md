# Big Air Park — Game Spec (current state, v79)

> Sound: in soccer and all sports (`sportsQuiet()`), the "Whoa!" scream and the in-air space music are off (the kid found them annoying); the park, bowl and Sky Zone keep them.

A browser 3D trick-and-sports game built with and for a 7-year-old (requests came by voice, one feature at a time). This document describes how the game works **today** so another agent can extend it without breaking it.

- **File:** `big-air-park.html` (single file, ~3,900 lines, ~300 KB). Everything — HTML, CSS, JS, textures (drawn on canvases), music (Web Audio) — is inline.
- **Library:** Three.js r128 from `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`. No build step, no other dependencies.
- **Published at:** https://claude.ai/artifact/2M8CQ7nYTGrZr18UAZh3Jo (republish by publishing the same file path).
- **Audience/tone:** a 7-year-old. On-screen text is short, excited, CAPS for big moments ("TOUCHDOWN!!!"). Everything should be forgiving and fun; "harder" was requested only for soccer AI and volleyball.

---

## 1. Big picture

The player rides one of five "rides" (snowboard, skis, skateboard, scooter, BMX) or goes **on foot**, does tricks for points, and can travel between **zones**. All zones live in one Three.js scene, far apart in world space; switching zones just teleports the player and changes theme/camera rules.

| Zone (`S.zone`) | Where in the world | How you get there |
|---|---|---|
| `park` | centered at origin, ±200 m (`LIM`) | start / **R** / Back buttons |
| `sky` (Sky Zone) | x ≈ 1000 | ride into the purple portal on the start hill |
| `bowl` (Giant Bowl) | (0, 3000) | **Giant Bowl** button |
| `soccer` | x = −1000 | **Soccer** button |
| `sport` + `S.sport` ∈ football / baseball / tennis / volleyball / basketball / bowling / golf | x = −2000 / −2600 / −3200 / −3800 / −4400 / −5000 / −5600 | orange **More sports** button → picker (basketball, bowling, golf are 🔒 until won as math prizes, in that order: `LOCKED_SPORTS`, `sportUnlocked()`, `nextLockedSport()`) |

`LIM` (park half-size) is a `let` (200, +50 per "Bigger park" prize, max 400). `heightAt(x, z)` routes ground height by region: `x > 500` → Sky Zone ramp; `z > 1500` → bowl; `x < −500` → flat (all sports); otherwise park ramps/pipes/start hill.

`bounds()` returns the walls for the current zone; the camera is clamped per zone so it never leaves the area or goes inside a ramp.

---

## 2. Core state & loop

- **`S`** (global state object, exposed as `window.__game`): position `x,y,z`, `vy`, air velocity `avx/avz`, `heading` (travel direction), `vis` (visual facing, eased toward heading), `speed`, `grounded`, `zone`, `sport`, `mode` (ride), `onFoot`, `fakie`, `score`, `best`, `planet`, `human` (outfit), `looks` (ride colors), trick state (`cur`, `queue`, `air`, `gtr` ground trick, `grind`, `bail`), `flying`, `diving`, etc.
- **Loop:** `requestAnimationFrame(loop)` → `update(dt)` (physics, input, zone logic) → `render(dt, time)` (camera, animation, minimap).
- **Gravity:** `grav()` → Sky Zone 3.2, soccer & sports 24, otherwise the current planet's `GRAV`.
- **Rider model hierarchy:** `player` → `spin` → `flip` → `content` (y −1) → `board` + `body` (`buildRider(mode)`); globals `armL`, `armR`, `riderWings`. `restPose()` resets all trick poses. Snowboard & skateboard riders are built rotated 90° (`MODES[mode].sideways`).
- **Never stuck:** a watchdog (`S.wd`) hops you to a clear spot if you push for 2.5 s without moving 4 m. Rails and the soccer friend are solid.

---

## 3. Controls

### Keyboard
| Key | Action |
|---|---|
| ↑ / W | speed up (baseball base running: **sprint**) |
| ↓ | slow down; hold when stopped = **fakie** (ride backwards); baseball running: run back to last base; baseball pitching: slow ball |
| ← → (or D for right) | steer; tennis: aim shot; baseball pitching: aim |
| Space | jump / 360 spin; with wings in the air: **flap/fly**; soccer with ball: shoot; baseball running: **pause/resume** |
| A | soccer: **shoot** (alternates normal shot ↔ **rabona**); sports: the **action button** (hike-throw-to-me / swing / pitch / hit / bump / pass) |
| S | **DIVE** (forward superman dive; with wings in the air: dive-bomb down) |
| M backflip · B ride toss · V 360 · 1 diagonal backflip toss · 2 drop & chase · 3 buttercup · 4 drop & crazy backflips · 5 free wheelie · X solo double backflip · Z soccer air pass · N slide under | tricks |
| C or 7 | bicycle kick · 8 skill moves · 0 slide tackle (soccer) |
| T | next planet · K Colors & Stickers panel · H help · R back to start |

Also: on-screen touch buttons (`touchKeys`) and gamepad support (`GP_MAP`; A jump, X backflip, L1/Y toss, B spin, L2+R2 crazy flips, R1 change ride, Start change color; d-pad/sticks steer).

Input dispatch: `keydown` → `TAP[e.code]()` for one-shot actions; held keys read from `keys[...]` in `update`. (Testing note: dispatch synthetic keyboard events to `document` only, not also `window`, or TAP fires twice.)

---

## 4. Rides, tricks, scoring

- **Rides** (`MODES`): snowboard, ski, skate, scooter, bmx — each has its own model, max speed (23–27), and trick names (e.g. scooter "Tailwhip", "Scooter Toss"). **On foot** button hides the ride.
- **Air tricks** (`TRICKS` table, keys above) each have a duration and points (300–3000). Landing a trick adds points; landing mid-trick = **WIPEOUT** (`S.bail`). Some tricks (X, Z, Y) auto-fit their duration to the predicted landing.
- **Ground tricks** (`S.gtr`): buttercup, wheelie, slide tackle, kick, skill moves, rabona, dive.
- **Grinds:** ride into yellow rails (`RAILS`); bowl coping gives a **Feeble Grind**.
- **Score / best** shown top-left; best saved in localStorage.

---

## 5. Zones in detail

### Park (`park`)
Outer-space park with low gravity (planet-based). High **START hill** with drop-in and kicker (`START`); the landing lane in front of it is kept permanently clear (`KEEP_CLEAR`). Ramps (`RAMPS`), halfpipes (`PIPES`), rails, plus procedurally placed extras that respect `KEEP_CLEAR`. The **Sky Zone portal** sits on top of the start hill.

### Planets (T / purple button)
`PLANETS`: Moon (g 9), Mars (13), Uranus (20), Earth (24, trees + snow), Sun (34), Meteorite (4), and **Star Planet** (`star`, won with a big-trick math challenge: dark starry sky `starSky` + nebula, glittery `starTex` ground, gravity 4.5 low or 34 high as chosen; added to `PLANET_ORDER` only once unlocked). `applyTheme()` swaps sky, fog, ground texture, lights and decorations. Saved in localStorage.

### Sky Zone (`sky`)
One giant ramp; super low gravity (3.2); the higher you fly the farther you go (`skyBoost`), tuned so you **always reach the finish line** at `SKY_FINISH` (18,500 m) — finish arch + timer. **Steering in the Sky Zone is gentle** (kid complaint: you zig-zagged so fast you never got across): on the ground turning is ×0.45; in the air the arrows only ease a small sideways drift (`S.avx` → ±7 m/s), forward `avz` is never rotated, `skyBoost` multiplies only forward motion, and `takeoff()` scales only `avz` and clamps `avx` to ±4. Original "space" music plays while airborne (copyrighted songs were requested and replaced with original tunes).

### Giant Bowl (`bowl`)
Huge lathe-shaped bowl (lip radius 75, walls 30 m). You can't leave: at the lip you auto-turn **180** and drop back in ("180 DROP IN!" +300). Shallow-angle coping contact = feeble grind.

### Soccer (`soccer`) — `field` object (`window.__field`)
Full match: you + 2 blue teammates + green goalie vs 3 red + red goalie; scoreboard and 3-minute clock. You keep the ball at your feet (possession), **A/Space shoot** (alternating normal / rabona curl), **C/7 bicycle kick** (auto-turns your back to goal), 8 skill moves, 0 slide tackle. Goalies (`keepers()`) predict the line crossing, react, dive, catch and throw out; tuned so **far shots and far bicycle kicks are saved**, close corner shots can score. Red AI is deliberately tough (steals, double-team, aims corners).

### More sports (`sport`) — `window.__sport = { fb, bb, tn, vb, sp, goSport, sportAction }`
Shared pieces: `sp.balls[sport]` meshes, `sp.vel`, yellow **landing ring** `sp.ring`, arrow `sp.arrow`, held gear (`sp.gear`: bat on `sp.batPivot`, `sp.racket`), helpers `lob()` (aim a parabola), `landAt()` (predict landing), `walkTo()`, `autoMove()` (moves *you* automatically). Blue teammates in every sport are **look-alikes of the player** (`mateKid()` / `dressMate()` rebuild them from the player's outfit, refreshed whenever looks change). Arms go **up when a ball is coming to you** (`catchArms`).

**Football** (`fb`, field 100×240 m, 2.2 m per "yard")
- Plays **start by themselves**: "HUT! HIKE!" after ~2 s → teammates run routes, blockers knock defenders down (stun), defense covers (3 defenders follow *you*, 2 per teammate, 1 deep).
- **A** = thrower passes to you (passes home in on you). If you don't press A within ~3.5 s, the thrower hits the most open teammate.
- After a catch you're nearly tackled within ~1 s unless you **pass (A)** to a teammate; **A again** = they pass it back. Teammates jump for high balls ("LEAPING CATCH"); you can jump-catch (Space).
- ~35% of passes to you are **bad throws** (no homing, land 4–6 m off) → run to ring and **S dive** ("SUPER DIVING CATCH" +1500).
- 4 downs; touchdown +7; first to 28 wins.
- **Red offense / interceptions** (phases `r_set`, `r_live`, `r_air`, `r_run`, `r_dead`; functions `frStart`, `frSet`, `frUpdate`, `frEnd`): red gets the ball after a turnover on downs, after intercepting you, or on the kickoff after your touchdown. Red runs up to **3 plays** toward your end zone: QB throws to one of 3 red receivers (often the one nearest you, long floaty pass with a yellow ring). **You** play safety: get to the ball = **INTERCEPTION** (+2000, S dive = bigger reach), or tackle the catcher by running into him (+500). Your look-alike teammates cover receivers and sometimes intercept; on a catch only the 2 closest chase (7.6 m/s, slower than the runner) and each tackle works only 30% of the time — a miss knocks the teammate down for 1.6 s ("He broke the tackle! YOU get him!"), so the player usually has to make the tackle. Red TD = +7 red; after 3 plays red punts; then you're back on offense.

**Baseball** (`bb`, bases 30 m apart, fence 100 m; test hooks `window.__bbx`)
- **Batting:** realistic side-on stance, bat over shoulder, keyframed swing (`BAT_KEYS`, `batPose()`), catcher's-view camera, white strike-zone box. Some pitches are balls — don't swing: 4 balls = walk; 3 strikes = out. **A** = swing; timing sets direction/power.
- **Running (after a hit):** your teammates already on base run too (animated, staying ahead of you; `bb.br`). You run the bases automatically; **↑ sprint** (15 vs 10 m/s), **Space pause/resume** (pausing near a base snaps onto it = safe), **↓ run back**. Fielders field the ball (visible, with arrow), throw to the base you're heading to, and chase; tagged off-base = out. Reaching home = inside-the-park HR; over the fence = HR trot.
- **Switch sides** after **3 outs or 3 home runs** (both teams). Then **you pitch** (A; ← → aim, ↑ fastball, ↓ slow ball) with your look-alike team in the field; the red batter decides to swing; on contact red **visibly runs the bases** (arms pumping) and outcome = how far they got. 3 innings; scoreboard shows YOU vs RED, balls/strikes/outs, HR x/3.

**Tennis** (`tn`, doubles): you + blue teammate vs 2 red. **You move to the ball automatically**; press **A** to hit (← → aim, high ball = smash). Steady TV camera behind your baseline. 15/30/40/game, first to 3 games.

**Basketball** (`bk`, unlockable; FULL COURT 26×46 m, `RIM_B` = your hoop at the far +z end, `RIM_R` = red's hoop at the near −z end): **3 on 3** — you + 2 look-alike teammates vs 3 red, man-to-man defense. You dribble automatically. Controls: **A = shoot** at the far hoop (make chance by distance and nearest defender; 3 pts beyond 6.75 m); **Space then A near the far hoop = SLAM DUNK**; **7 = pass** to your most open teammate, and when a teammate has it **7 = they pass it back** (`bkPassKey`); when red has it they drive to the near hoop and **S = dive + STEAL** (`bkSteal`: works within 6 m, lunges you at the ball carrier, 85% inside 3 m / 65% farther; a "Press S to STEAL!" hint shows when you're close). After every basket the other team brings it up from half court. **Energy**: running drains your ⚡ (shown on the scoreboard; tired = max speed 6 + 8×energy), teammates tire too; **B or the 🔄 SUB button = substitution**: 2 fresh teammates come off the bench (`bk.bench`, seated by the sideline) and your energy refills. Red subs automatically when tired. First to 15.

**Bowling** (`bw`, unlockable): 10 pins in a triangle at the end of a lane (`LANE`, `HEAD`). ← → slides you left/right behind the foul line; **A = roll** (hold ← or → while rolling to curve it). Random wobble so strikes aren't automatic; pins knock other pins; gutter balls. 2 rolls per frame, 10 frames; simplified scoring: pins + strike 20 / spare 15. Strike or spare → math question.

**Golf** (`gf`, unlockable; was Mini Golf): 4 long holes (`HOLES`, course 20×88 m: wall to hit over par 3, zig-zag par 4, **pond** par 3, blocks + pond par 4). ← → aim (yellow arrow), **A** starts the power meter, **A again** = hit. Strong shots (power > 50%) **fly in the air** (up to ~3.7 m, over the 0.6 m blocks), bounce and roll; **hold ↑ while hitting = DRIVE** (35% faster + higher). Landing in water = SPLASH, +1 stroke, replay from the last spot. **F or ⏩ button = fast-forward** the rolling ball 3×. Golf camera follows behind the ball toward where it's going; the ball has a pulsing yellow halo, the cup a yellow ring and a tall flag. 9-stroke pick-up; par or better → math question.

**Volleyball** (`vb`, beach): you + teammate vs 2 red. **You move automatically**, can't cross the net, steady camera. Turns alternate: on your teammate's turn they dig and **set** for you; on your turn your **A bump** goes up to your teammate, who usually sets it back. When the ball is set, **press A once = auto jump + SPIKE** (+800). Red is tough (digs many spikes, spikes back). First to 7.

---

## 6. Customization ("Colors & Stickers", K)

- Per-ride colors + stickers (`S.looks`).
- **Human** section (`S.human`): sweater color, outfit (Normal, Soccer kit #10, **Football kit** #88 with pads & face-mask helmet, **Baseball kit** #7 pinstripes + cap/glove, **Tennis kit** polo/headband/wristbands, **Volleyball kit** knee pads/headband, Hoodie; plus kits that appear only once their sport is unlocked (`OUTFITS[].lock`): **Basketball kit** #23 tank top/long shorts/headband, **Bowling kit** retro striped collared shirt, **Golf kit** polo/khakis/cap/glove), helmet on/off, **wings** (13: Angel, Dragon, Butterfly, Rainbow, Bat, Fairy, Bee, Phoenix, Ice, Robot, Gold, Galaxy, Parrot) and **wing color** (13 incl. Normal; see-through/glowing wings keep that property).
- **Wings let you fly:** in the air, hold/tap Space to flap, arrows steer, ↑ faster; you can fly with the soccer ball and shoot it.

---

## 7. LEARNING MODE (`window.__learn`)
- **Toggle:** `📚 Learning on/off` button (top right), saved in `bigair-learn`. Off = no counting, no questions, chip hidden; the park plays exactly as before.
- **Counting:** every landed trick or grind in the park, bowl or Sky Zone counts (`countMove(n, big)` from `land()`; bails never reach it). HUD chip `#mathChip` shows `🧮 n/10 ⭐ solved`.
- **Level 1 (every 5 landed tricks — `QUIZ_EVERY = 5`):** addition 1–9 + 1–9 with answers up to 12, or subtraction 4–12 minus 1–6 (never negative) — made easier at the kid's request. Rewards: 🔓 **NEW SPORT** (the next locked sport: basketball → bowling → mini golf; offered at both levels while any remain), 🎨 **new color for your current ride** (pick from 4 reward colors in `COLORS` marked `reward: true`; won colors also appear in the Colors panel), 🛹 **new ramp** (placed in front of you, rideable through the normal `RAMPS`/`heightAt` logic), plus the kid's extras 🛤️ new grind rail and ⚾ extra turn at bat.
- **Level 2 (after landing a BIG trick):** `BIG_TRICKS = ['X', 'Q', 'D']` (solo double backflip, drop & crazy backflips, diagonal backflip toss). Tens-and-ones with **no carrying or borrowing** (e.g. 30 + 6, 20 + 30, 23 + 5, 27 − 4; the kid found 27 + 37 too hard) (never negative; answers can be 3 digits). Rewards: 🪐 **Star Planet** (choose LOW or HIGH gravity, travel there immediately; picking again changes gravity), 🔓 next locked sport, ⭐ Super ride (gold, +6 speed), 🏞️ Bigger park.
- **Football TOUCHDOWN CHALLENGE:** crossing the goal line (you or a teammate carrying) pauses for a level-1 question *before* the TD counts (`fbScore` → `tdQuiz`). **One try:** right → `fbTouchdown()` (+7) and the prize picker; wrong → `fbFumble()`: quiz closes, "FUMBLE!!! The answer was N", red gets the ball (`frStart`). Only when learning is on.
- **Scoring in sports (level 1):** `scoreQuiz(what)` fires ~1.8 s after you score: soccer goal, baseball getting on base (hit, walk, inside-the-park or over-the-fence HR, via `bbAdvance`), tennis/volleyball point won by you, basketball basket, bowling strike/spare, mini golf hole at par or better. Red scoring never asks. Title reads e.g. "🎉 TOUCHDOWN! MATH TIME!"; same rewards as level 1.
- **Overlap rule:** a big trick that is also the 5th trick → one big challenge, counter resets to 0.
- **Answering:** the game pauses (`S.quiz`; `update()` returns early) with **unlimited time**. Type digits (keyboard captured while open; Enter/Backspace) or use the on-screen keypad. **Wrong → a different question at the same level** (unlimited tries) with help: counting dots for small numbers, tens bars + ones dots for big numbers. Right → +1000 (level 2: +2500) and the reward picker; the reward applies immediately and play resumes.
- **Prize picker (`showPrizes`, `givePrize`):** level 1 offers next locked sport, new color (`showPrizes('color')`), new ramp, new rail, extra bat, **new sticker** (`showPrizes('sticker')`; reward stickers Rainbow/Checkers/Hearts/Zigzag/Diamonds/Sunburst in `STICKER_LIST` with `reward: true`, unlocked in `S.unlocks.stickers`, hidden in the panel until won), **MEGA ramp** (`placeNewRamp(true)`: len 15, h 7, `big` flag saved as `a[6]`), **Super speed** / **Super jump** for 60 s (`S.pow.speed/jump`, ticked down in `updateTrail`, badge `#powChip`; jump power ×1.6, max speed +14), **Sparkle trail** (`S.unlocks.trail`), **+5,000 points**. Level 2 adds **Rainbow trail**, **3 MEGA ramps**, **Super powers** (speed + jump 120 s). The prize grid scrolls (`max-height: 52vh`).
- Verified (headless): 9 tricks = no question, 10 = question; wipeout doesn't count; big trick → level-2 question and counter reset; wrong answers give new same-level questions; color and ramp apply immediately and the ramp launches you; Star Planet low gravity jump ≈ 6.2 s air / 21.7 m vs high ≈ 0.8 s / 2.8 m; learning off = no questions.
- All rewards persist in `bigair-unlocks` (park size, super rides, basketball, colors, star planet gravity, added ramps/rails, extra bats, solved count) and are rebuilt on load.

---

## 8. Persistence (localStorage, all wrapped in try/catch)
`bigair-best`, `bigair-planet`, `bigair-looks`, `bigair-human`, `bigair-sound`, `bigair-pad`, `bigair-skytime`, `bigair-flight`, `bigair-learn` (on/off), `bigair-unlocks` (math rewards: park size, super rides, basketball, colors, star planet, added ramps/rails, extra bats, solved count).

---

## 9. Testing hooks & workflow
- `window.__game` (S), `__mus`, `__ramps`, `__rails`, `__field`, `__sport`, `__bbx`, `__fbx` (`frStart`), `__learn` (`openQuiz(level)`, `quizType`, `givePrize`, `countMove`, `setLearn`, `showPrizes`, `makeQuestion`, `bk`), `__mates`.
- `window.__step(dt, n)` runs `n` update+render steps headlessly (now ≈15–40 ms each with the bigger scene — keep a single test call under ~45 s, e.g. ≤ ~600 steps; split longer tests).
- Local preview: a static server on port 8765 (`.claude/launch.json` config "game"); add `?v=N` to bust the browser cache after edits.
- Typical test: click `#go` to start, `__sport.goSport('football')`, call `__sport.sportAction()` or dispatch key events on `document`, step, then read `#callMain`/`#callSub` (the big on-screen message) and `#board` (scoreboard).
- The preview pane has often been hidden, so visuals were mostly verified by numbers, not screenshots.

---

## 10. Difficulty (v73: the kid asked for "way harder")
- Soccer: red runs 10.5–11.5 m/s, grabs from 1.3 m, steals at dt×4.5; red goalie faster (6.3) with more reach; your goalie weaker; you dribble ≤ 11.
- Football: defenders 13.6–14.4 m/s; 50% bad throws; catch radius 2.4 (dive 5.0); 0.45 s grace after a catch; red intercepts from 1.5 m; blocks work 50%. Red offense: carrier 11 m/s, teammates tackle only 15%, your interception radius 2.0 (dive 4.2).
- Baseball: pitches 0.62–0.8 s; smaller hit window and sweet spot; fielders react in 0.3 s and run 6.2–9.5; throws faster; red batters swing and connect more.
- Tennis: red runs 9.2, rarely misses (3%), hits 22–28 m/s to the lines; your reach 2.7, auto-move 7.2.
- Volleyball: red digs 82% of your spikes, spikes 62% of the time (faster), reaches farther; your auto-move 5.8 (4.3 vs spikes).

## 11. Conventions to keep
- One file; Three.js r128 only; canvas-drawn textures; no external assets.
- Kid-friendly feedback via `call(main, sub)` (big center message) and `hint(text)` (small line).
- New zones go far away on the x axis (x < −500 is flat ground) with their own `bounds`, camera clamp, and `applyTheme` handling.
- Don't recreate copyrighted songs/characters; use original tunes/designs.
- Balance: forgiving for the player, but the kid has asked for tougher opponents in soccer and volleyball; goalies save far shots.

---

## 12. Learning game: ideas not built yet
Math-every-10-moves is built (section 7). Other ideas offered to the child: reading/spelling, planet facts, questions before pitches/kicks, flying to grab the floating right answer, and a difficulty setting (bigger numbers).

### People, talking and the Race (`race` zone) — v79
- **People in the park** (`PEOPLE`, built with `makeKid` + `labelSprite` name tags): Coach Rio stands on the start hill next to the spawn; Mia, Leo, Ava and Sam are auto-placed on flat open ground (`clearSpot`, no ramps/rails within 3 m). They turn to face you and wave when you're close (`updatePeople`). Within 5.5 m a yellow **"💬 Press E to talk"** button appears (`talkHint`, clickable); **E** (`talkKey`) opens `talkBox` (Next/Bye; E = next line). **Escape** closes it; riding 9 m away closes it.
- **Coach Rio** offers a race: E / "Let's race!" → `goRace()`.
- **Race zone** at x 0, z −3400 → −2200 (`RACE`, half-width 14), routed in `heightAt` (`z < −1500`, |x| < 500 → `raceHeight`), `bounds()`, camera clamp, `applyTheme` (canyon orange), gravity 20. Pink ramps across the track (`RACE_RAMPS`, full-width / two side-by-side / one big side kicker), ~75 **rocks** (`ROCKS`) between them: hitting one = `bail()` + "CRASH!". Three racers (`RACERS`: Zoe, Max, Kai) on boards with their own simple physics, dodge rocks, fly off ramps doing a 360; speed is scaled to your ride's max and rubber-banded slightly in your favor. 3-2-1-GO countdown freezes `update()`; `#raceChip` shows place / time / meters to go. Finish → place points (5000/3000/2000/1000), best place saved (`bigair-race-best`), back to the park next to the coach after 6 s (`leaveRace`). **R** also leaves. Debug handle: `window.__race`.
- **Fence grind:** the park fence now sits at `LIM + 0.3` with a shiny metal top bar; `FENCE_RAILS` (4 sides, h 1.13, `fence: true`, wide `catch: 3.2`) are in `RAILS`, repositioned by `placeFence()` (also called by `growPark`). Ride along the edge or jump onto it to grind; riding into it head-on still bounces.
