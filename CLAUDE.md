# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this repo is

A single-purpose automation: it fetches sunrise/sunset times for a fixed location
(Plano, TX — `LAT 33.0198`, `LNG -96.6989`) from the public
[sunrise-sunset.org](https://api.sunrise-sunset.org/json) API and writes an
iCalendar feed to `docs/sun.ics`. A GitHub Actions cron job regenerates and
commits that file every 6 hours so calendar apps subscribed to the raw file URL
stay current.

There is no application, no server, no package, and no test suite. The entire
product is one Python script plus one workflow.

## Layout

```
sun_calendar.py                          # the whole program (~110 lines, no modules)
.github/workflows/update-sun-calendar.yml # cron + manual trigger, commits the output
.github/workflows/.gitignore              # Python ignore rules (misplaced, see Gotchas)
docs/sun.ics                              # GENERATED OUTPUT — do not hand-edit
README.md                                 # end-user setup + subscription instructions
```

## How the generator works

`sun_calendar.py` runs top to bottom in `__main__`:

1. `build_ics()` writes VCALENDAR headers and a hardcoded `VTIMEZONE` block for
   `America/Chicago` (US DST rules: 2nd Sunday of March, 1st Sunday of November).
2. It loops `DAYS_AHEAD` (400) days starting from *today in UTC*, calling
   `get_sun_times()` once per day — one HTTP request per date, with
   `time.sleep(0.1)` between calls as rate limiting.
3. API results come back as ISO-8601 UTC (`formatted=0`) and are converted to
   `LOCAL_TZ` for the `DTSTART;TZID=America/Chicago:` value.
4. `add_event()` emits a `VEVENT` per sunrise and per sunset — 800 events total —
   each with `TRANSP:TRANSPARENT` (shows as free time) and a `VALARM` firing at
   `TRIGGER:-PT15M`.
5. The result is written to `docs/sun.ics`, creating `docs/` if needed.

Per-date API failures are caught, printed, and **skipped** — the run still
succeeds and produces a calendar with holes. If you change error handling, decide
deliberately whether a partial calendar should still be committed.

## Running it

```bash
pip install requests pytz     # only dependencies; there is no requirements.txt
python sun_calendar.py        # writes docs/sun.ics
```

Expect the run to take **at least ~40 seconds** (400 sequential HTTP requests plus
sleeps) and to require outbound network access. Do not run it casually just to
"check" a change — it hits a free public API 400 times. To smoke-test a change to
the ICS format, temporarily lower `DAYS_AHEAD` locally and **revert it before
committing**.

There is no linter, formatter, type checker, or test runner configured. Keep the
existing style: module-level constants in caps, small top-level functions, plain
`lines.append(...)` string building, f-strings.

## The generated file

`docs/sun.ics` is build output that happens to be committed (that is the delivery
mechanism — subscribers fetch it from the repo). Treat it as artifact, not source:

- Never edit it by hand; change `sun_calendar.py` and regenerate.
- Never resolve a merge conflict in it manually — regenerate instead, or take
  either side and let the next scheduled run overwrite it.
- Regenerating rewrites all 800 events, so any local run produces a ~200 KB diff.
  Avoid committing an unrelated regeneration alongside a code change.

## The workflow

`.github/workflows/update-sun-calendar.yml` runs on `cron: '0 */6 * * *'` (UTC)
and on `workflow_dispatch`. It checks out, installs Python 3.11 + `requests pytz`,
runs the script, and commits as `GitHub Action <action@github.com>` with the
message `Update sun calendar [skip ci]`.

Rules when touching it:

- Keep `[skip ci]` in the commit message. Without it, the bot's own push can
  retrigger workflows.
- Keep `permissions: contents: write` — the job pushes to `main`.
- Dependencies are installed inline via `pip install requests pytz`. If you add a
  dependency, update **both** that line and the README, or introduce a
  `requirements.txt` and switch the workflow to it.
- Nearly every commit is the bot's `Update sun calendar [skip ci]`. To find human
  changes, filter them out:
  `git log --oneline --invert-grep --grep='Update sun calendar'`. In a shallow
  clone (what CI and remote sessions get) that can legitimately return nothing —
  the truncated history is all bot commits. Deepen with
  `git fetch --unshallow` before concluding a change isn't there.

## Gotchas

- **The output always differs.** `DTSTAMP` is set from `datetime.now(timezone.utc)`
  on every run, so the workflow's `git diff --quiet` guard almost never short-
  circuits — it commits every 6 hours even when sun times are unchanged. If asked
  to stop the commit churn, that field is the cause; the fix is to make `DTSTAMP`
  deterministic (or diff ignoring it), not to change the schedule.
- **Event UIDs embed the emoji.** `uid = f"{date_str}-{title.replace(' ', '').lower()}@sun-calendar"`
  and `title` is `"🌅 Sunrise"` / `"🌇 Sunset"`, so UIDs look like
  `2026-08-21-🌅sunrise@sun-calendar`. Changing the emoji or wording in `SUMMARY`
  changes every UID, which makes subscribed calendars show duplicate events
  instead of updates. Decouple the UID from the display title before rewording.
- **Events have no `DTEND`/`DURATION`.** They are point-in-time by design. Clients
  render them as zero-length; don't "fix" this without a reason.
- **Line endings are `\n`, not CRLF.** RFC 5545 specifies CRLF and 75-octet line
  folding; neither is implemented. Current lines are short and mainstream clients
  accept the file, so this is a known, tolerated deviation.
- **`VTIMEZONE` is hardcoded** to US DST rules and `America/Chicago`. Changing
  `LAT`/`LNG` to another region requires updating `LOCAL_TZ` *and* rewriting that
  block — the coordinates alone are not enough.
- **There is no root `.gitignore`.** The Python ignore rules sit at
  `.github/workflows/.gitignore`, where they only apply to that directory. The
  README tells users to add a root `.gitignore`; the repo never did. Move it to
  the root if you touch this area.
- **README is written for someone forking the project**, with
  `YOUR_USERNAME/YOUR_REPO_NAME` placeholders rather than this repo's real URLs.
  Keep that framing unless asked to change it.

## Conventions

- Default branch is `main`; the workflow commits directly to it.
- Feature work goes on a branch and is pushed with `git push -u origin <branch>`.
- Don't open a pull request unless explicitly asked.
- Keep changes minimal and in the existing single-file style — this repo does not
  want a package structure, a config file, or a framework.
