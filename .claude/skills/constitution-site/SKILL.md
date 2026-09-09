---
name: constitution-site
description: Working rules for the รัฐธรรมนุญจำลอง site (one 5MB website_constitution.html). Load before ANY edit to that file. Covers the structural hazard that makes broken nesting invisible, the mandatory validation script, insertion recipes, shell/encoding traps, site conventions, and the commit-and-push-every-change rule.
---

# รัฐธรรมนุญจำลอง — working rules

Fictional worldbuilding site: a simulated Thai constitution, 37 chapters plus a
40-tab dashboard. Invented history, PMs, parties and institutions are
**intentional**. Never "correct" them toward real-world facts.

Almost everything lives in one file: `website_constitution.html`
(~5.5 MB, ~68k lines). Assets in `images/` (107 MB), `audio/`, `js/`, `css/`.

---

## RULE 1 — Commit and push after every change

The user's explicit standing instruction. Do not batch work.

```bash
git add -A && git commit -m "<what changed>" && git push
```

Rationale: one 5 MB file where a bad splice can corrupt the whole site. Every
commit is a restore point. During one session 12 separate pieces of work sat
uncommitted at once — a single bad write would have destroyed all of it.

If a splice goes wrong and the work is committed, recovery is
`git checkout -- website_constitution.html`. If it is not committed, there is
no recovery. **Commit first, then edit.**

---

## RULE 2 — Tag counts do NOT prove the structure is sound

This is the single most important fact about this file.

Damage here consistently takes the form of **balanced `<div>` counts with wrong
nesting order**. Every naive check passes. The page still renders. The bug only
shows as content appearing on tabs it does not belong to.

Real cases:

- A `</div>` closed `chakri-tab` early, leaving the whole ราชวงศ์จักรี body
  outside any `.db-tab-content`. It rendered on **every** tab. Counts were
  11886/11886 — perfectly balanced.
- Two new panels were spliced **inside** `kpptp-tab` instead of beside it.
  Counts balanced (58/58, 93/93, 86/86). Only the overlap check caught it.

The user has said the original damage came from **Antigravity**, which broke the
structure and could not fix it. Treat misnesting as the default suspicion.

### The validation script — run after EVERY structural edit

```python
import re
src = open('website_constitution.html', encoding='utf-8').read()
stack, extra, spans = [], [], {}
for m in re.finditer(r'<div\b([^>]*)>|</div>', src):
    if m.group(0) == '</div>':
        if stack:
            o = stack.pop()
            if o[1]: spans[o[1]] = (o[0], src[:m.start()].count('\n') + 1)
        else:
            extra.append(src[:m.start()].count('\n') + 1)
    else:
        a = m.group(1) or ''
        idm = re.search(r'id="([^"]+)"', a)
        stack.append((src[:m.start()].count('\n') + 1,
                      idm.group(1) if (idm and 'db-tab-content' in a) else None))
sp = sorted((v[0], v[1], k) for k, v in spans.items())
overlaps = [(sp[i+1][2], sp[i][2]) for i in range(len(sp)-1) if sp[i+1][0] < sp[i][1]]
buttons = set(re.findall(r"switchTab\('([^']+)'", src))
print(f"unclosed {len(stack)} | stray {len(extra)} | panels {len(spans)} | overlaps {overlaps}")
print(f"buttons {len(buttons)} == panels: {buttons == set(spans)}")
```

**Expected healthy output:**

```
unclosed 2 | stray 0 | panels 40 | overlaps []
buttons 40 == panels: True
```

- `unclosed 2` is correct and expected — `dashboard-card` and `preamble-section`
  are knowingly left open to EOF; the browser closes them. Do not "fix" these.
- `overlaps` must be `[]`. Any entry means a panel is nested inside another.
- Buttons and panels must match exactly, both directions.

Also verify every panel sits inside the tab container:

```python
ho = src[:src.index('<section id="history_and_pms"')].count('\n') + 1
hc = src[:src.index('</section>', src.index('id="kpptp-tab"'))].count('\n') + 1
outside = [k for k, (s, e) in spans.items() if not (ho < s and e < hc)]
```

### Do NOT use per-line depth counting

A naive line-by-line depth walk reports **false overlaps**, because the file
contains lines like:

```html
</div><div id="independent-organs-tab" class="db-tab-content">
```

One line closes and opens. Always scan in document order with a stack.

Likewise, a slice-and-count of one panel's line range can report 435/434 for the
same reason. Trust the stack walk, not the slice count.

---

## RULE 3 — Insertion recipe (this is where mistakes happen)

Three separate splices went wrong in one session. Follow this exactly.

### Find a tag's real span

```python
def span(marker):
    a = src.index(marker)
    s = src.rindex('<div', 0, a)      # ← the actual opening tag
    d = 0
    for m in re.finditer(r'<div\b[^>]*>|</div>', src[s:]):
        d += 1 if m.group(0) != '</div>' else -1
        if d == 0:
            return s, s + m.end()
```

**The trap:** `src.index('id="kpptp-tab"')` points at the *attribute*, not the
tag. Starting the walk there begins one tag late, so it closes one level early
and everything spliced at that point lands *inside* the panel. Always
`rindex('<div', 0, a)` first.

### Never grab the first `</section>`

`src.index('</section>')` finds the one closing `<section class="hero">` near
line 2083 — nowhere near the tabs. A panel spliced there ends up inside the hero
banner. Tab panels live inside **`<section id="history_and_pms">`**.

Assert containment before writing:

```python
host_o = src.index('<section id="history_and_pms"')
host_c = src.index('</section>', insertion_point)
assert host_o < insertion_point < host_c, "must land inside the tab container"
```

### Adding a new tab — both halves are required

1. **Panel** — insert as a *sibling* after the last panel closes, inside the host
   section: `<div id="NAME-tab" class="db-tab-content"> … </div>`
2. **Button** — beside a related one in the nav:
   ```html
   <button class="db-tab-btn" onclick="switchTab('NAME-tab', this)"
           style="border-color:rgba(R,G,B,0.6);color:#HEX;">EMOJI ชื่อแถบ</button>
   ```
3. Validate. Buttons and panels must both come out at the new count.

Keep button labels short — `🔴 พรรคสีแดง`, not a parenthetical list. Long labels
wrap to two lines and look wrong.

---

## RULE 4 — Shell and encoding traps

Every one of these cost a failed command in real sessions.

| Trap | Symptom | Fix |
|---|---|---|
| PowerShell here-string `@'…'@` in a Bash call | `@` becomes the commit subject line | Use `git commit -F -` with a Bash heredoc |
| Bash heredoc with large Thai/emoji content | `unexpected EOF while looking for matching` | Write the script or HTML to a file with the Write tool, then run/splice it |
| Python printing Thai on Windows | `UnicodeEncodeError: 'charmap' codec` | Prefix every command: `PYTHONIOENCODING=utf-8 python …` |
| `"\\v%d.js"` in a Python path | `\v` is a vertical tab → `Invalid argument` | Use raw strings or `os.path.join` |
| `grep -P` | `-P supports only unibyte and UTF-8 locales` | Use Python `re`, or `grep -oE` |
| ripgrep lookahead `(?!…)` | `look-around … is not supported` | Filter in Python instead |
| Non-greedy `.*?` across the whole file | Matches content in unrelated sections | Scope the regex to one panel's slice first |
| `file://` with Thai path in the browser tool | Cannot open | Validate structurally instead; browser preview is not available for this file |

**Always read a file's real bytes before assuming.** Reading
`website_constitution.html` whole fails (5 MB > 256 KB limit) — use `offset`/
`limit`, Grep, or Python.

---

## RULE 5 — Verify claims against the file, never from memory

The user checks sources. When a figure is used, be able to name the line it came
from. When asked "เอามาจากไหน", answer with line numbers.

Before writing any factual claim, grep for it. Two examples where checking
changed the answer:

- Four places said `จอมพล (กองทัพบก)`. Three were จอมพลคงฤทธิ์; **one at line
  ~40021 was จอมพลปฏิวัติ พิบูลอสงไขย — a different person.** A blind
  replace-all would have corrupted a villain's record.
- PM 4's term count was assumed to be one 8-year term. The site actually
  documents `วาระละ ๘ ปี` for the modern era only; the older constitution used
  4-year terms, making it two terms. The user had to correct this.

**Do not invent numbers unless told to.** When data is missing, say which fields
are missing and ask. The user will often say "คิดขึ้นมาเองได้เลย" — only then
invent, and keep invented figures internally consistent:

- seats ÷ 5 = the stated percentage (500-seat house)
- แบ่งเขต + บัญชีรายชื่อ = total seats
- raw votes ÷ popular-vote% = a turnout that grows sensibly across eras
- respect any ceiling the user sets (e.g. "must not exceed พิธา วาระ ๒" = 76.89% / 384 seats)

---

## RULE 6 — Site conventions

**Numerals.** Thai numerals (๐-๙) for years, counts, article numbers. Arabic is
tolerated inside statistics blocks where the site already mixes them.

**Duplicate function definitions.** `switchTab` and `playAudioMobile` are defined
both inline in the HTML and in `js/constitution.js`. **`js/constitution.js` loads
last (near line 66615) and wins.** Editing the inline copies does nothing. The
two `switchTab` bodies differ.

**Tab panels** are `<div id="X-tab" class="db-tab-content">`. `switchTab` hides
by that class — anything outside it can never be hidden.

**Audio + lyrics block** (reuse verbatim, swap colours):

```html
<div class="audio-box">
  <button type="button" onclick="playAudioMobile(this, 'audio/NAME.mp3')" …>
    <span style="font-size:15px;">▶️</span> กดเล่นเพลงบนมือถือ / Play Audio
  </button>
  <audio controls preload="metadata" playsinline webkit-playsinline src="audio/NAME.mp3" …>
    <source src="audio/NAME.mp3" type="audio/mpeg">
    เบราว์เซอร์ของคุณไม่รองรับการเล่นไฟล์เสียง
  </audio>
</div>
```
Lyrics go below in an italic box with a coloured left border.

**Person cards** (`figures-tab`, `villains-tab`) use `tri-layout` /
`tri-profile-card` / `tri-stage`. Portrait frame is 243.75 × 304.69.

**PM roster rows** (`pms-tab`) are `<tr class="pm-row" data-era="era-N">` with
exactly **7 cells**. Portrait 220 × 275.

**Party tab PM cards** carry a stat card per term: raw votes, Popular Vote %,
ส.ส. ในสภา (n/500), โพลแรก/โพลหลัง, then a 3-column grid (ส.ส. รวม / แบ่งเขต /
บัญชีรายชื่อ), plus a footer line `จัดตั้งรัฐบาลสถาปนานายกฯ คนที่ …`. The right
column carries a ฉายา badge, name, party/years line, then achievements as
bulleted items with emoji headings.

**Images.** Portraits are ratio ~0.80 (e.g. 800×1000). Crop landscape sources
centred, keep full height, save JPEG q92. Check the result with Read before
using it.

---

## RULE 7 — Cross-references must stay consistent

Facts are duplicated across tabs. Changing one means sweeping for the others.

When a new PM is added, update **all** of:
- the roster row in `pms-tab`
- the roster intro count (`ทำเนียบนายกรัฐมนตรีทั้ง ๓๓ คน`)
- the tab button label (`🏛️ ทำเนียบนายกรัฐมนตรี (๓๓ ท่าน)`)
- the previous PM's end year — two PMs cannot both be `ปัจจุบัน`
- the party card in `parties-tab` (`นายกรัฐมนตรีคนปัจจุบัน: …`)
- the democracy timeline era in `timelineData`
- the party tab card, if the party has one

Sweep afterwards:

```bash
grep -c '๒๖๙๘-ปัจจุบัน' website_constitution.html          # must be 0
grep -n 'นายกรัฐมนตรีคนปัจจุบัน' website_constitution.html  # only one Thai PM
```

**`timelineData`** (around line 2396) is an array of
`{title, desc1, desc2, desc3, result}` rendered by `selectTimelineEra(index)`.
It is index-driven with no hardcoded length, so adding entries is safe — but
each entry needs a matching `<button onclick="selectTimelineEra(N, this)">`.
Verify by executing the array with node, not by eye.

---

## Known-good baseline

```
unclosed 2 (dashboard-card, preamble-section) | stray 0 | panels 40 | overlaps []
buttons 40 == panels: True
13 inline <script> blocks, all pass node --check
0 broken local references
pms-tab: 34 rows (1 header + 33 PMs), every row 7 cells
```

Check all inline scripts:

```python
for k, m in enumerate(re.finditer(r'<script(?![^>]*\bsrc=)[^>]*>(.*?)</script>', src, re.S | re.I)):
    open(f'chk_{k}.js', 'w', encoding='utf-8').write(m.group(1))
    # then: node --check chk_k.js
```

Check every local asset resolves:

```python
missing = [f for f in set(re.findall(r'(?:src|href)="([^"]{1,200})"', src))
           if not f.startswith(('http', '#', 'data:', 'mailto:', 'javascript:'))
           and not os.path.exists(f.split('?')[0])]
```
