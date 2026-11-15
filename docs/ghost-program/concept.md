# Ghost program: concept

Status: **plan**, nothing built yet. Written 2026-11-15 for leadership review.

## Why

Every member keeps one dedicated alpha alt, a **ghost**, staged in **Amo** for
**highsec home defense**. When war targets hit our highsec structures and the
mains are deployed elsewhere, members log in their ghosts and defend. It works
like The Initiative's mandatory ghost program, which keeps an alpha army staged
in their home region.

Every member of FL33T and BUILD is expected to have a ghost. Leaders need to
see who has a ready ghost and who doesn't.

## What a member does

1. Create a new account through EVE's recruit-a-friend link from the main
   account, so the ghost gets the free skill points. They hold the skill points
   until the skill plan is installed.
2. Create an alpha character and add it to my.minmatar.org from the main's
   account. The Basic token is enough: it already covers assets, skills and
   clones.
3. Apply to the ghost corporation, naming the main and the main's corporation
   in the application.
4. Travel to Amo, dock in the staging structure, and set the medical clone
   there.
5. Buy at least one ghost-doctrine ship and leave it docked in the staging
   structure.
6. Train the ghost skill plan.

The site shows the member their own ghost status, with these steps and what
is still missing.

## When a ghost counts as ready

A ghost is ready when all four checks pass:

| Check | Passes when | Data the site already has |
| --- | --- | --- |
| Corporation | The character is in the ghost corporation | `EveCharacter.corporation_id`, refreshed every 30 minutes |
| Clone | The medical clone is in the Amo staging structure | `medical_clone_location_id`, synced by `backend/eveonline/helpers/characters/clones.py` |
| Ship | The character owns a ghost-doctrine hull docked in the staging structure | `EveCharacterAsset` (ships only), the doctrine's hulls from `EveDoctrineFitting` and `EveFitting.ship_id` |
| Skills | Skill-plan progress is at or above the minimum | `EveCharacterSkill` against the ghost `EveSkillset` |

Assets, skills and clones sync every 4 hours, so no new ESI scopes are needed.
A check whose data is missing because the token lacks a scope, or because the
character has never synced, shows as **can't check**, not as a failure.

Which character is the ghost: any of the member's linked characters that is in
the ghost corporation. Members don't pick one. If a member has several in the
corporation, the most ready one counts.

Each member gets one status:

- **Ready:** a ghost passes all four checks.
- **Partial:** a ghost is found, and the dashboard lists what is missing.
- **No ghost:** no linked character is in the ghost corporation.
- Members **on leave** are listed separately and don't count as missing.

## Who sees what

- **A member** sees their own ghost card on their profile.
- **A CEO or director** sees every member of their own corporation, so they
  can follow up with them.
- **Alliance leadership** sees the whole of FL33T and BUILD, with counts
  (Ready, Partial, No ghost) and filters by corporation and status.

Sketch of the dashboard:

```text
Ghost program · 412 members
Ready 188 · Partial 97 · No ghost 127 · On leave 14

Member        Corp   Ghost          Corp  Clone  Ship  Skills
Pilot A       MEC    Pilot A Ghost  ✓     ✓      ✓     100%
Pilot B       FOSFO  —              —     —      —     —
Pilot C       TDT    Pilot C Alpha  ✓     ✗      ✓     74%
```

## How it would be built

A new small backend app, `ghosts`, keeps the program's rules apart from tribes
and member status.

- **Settings:** one row, edited by leadership. It holds the ghost corporation
  (empty until it exists), the Amo staging location (an `EveLocation`), the
  ghost doctrine (an `EveDoctrine`), the ghost skill plan (an `EveSkillset`) and
  the minimum skill progress (default 100%). Leaders can change them without a
  deploy.
- **Readiness:** one row per member holds the ghost character, the four
  results, the skill percentage, the status and when it was checked. A Celery
  task rebuilds the rows after the character sync and on a beat. The dashboard
  only reads them, so it stays fast for the whole alliance.
- **The checks:**
  - **Ship:** works like the tribe asset requirement in
    `backend/tribes/helpers/requirements.py`.
  - **Skills:** the app computes progress itself, from the skill-plan text.
    The stored `EveCharacterSkillset.progress` can't be used: its maths in
    `compare_skills_to_skillset` is unreliable and usually reads 0 or 100.
- **API:**
  - Endpoints: one for the member's own ghost, and a dashboard endpoint
    filtered by corporation.
  - Schemas: Ninja response schemas, with the frontend types regenerated from
    them.
  - Access: leadership's view is a new pilot feature in
    `backend/groups/features/registry.py` and
    `frontend/app/src/json/feature_gates.json`. CEOs and directors are limited
    to their corporation by the existing corporation permission helpers.
- **Frontend:** a dashboard page (an htmx partial table, with Alpine filters
  over the rows already on the page), a ghost card on the profile, and the
  step-by-step guide above.

It ships in two pull requests after this concept: first the backend
(settings, checks, task, API and tests), then the frontend.

## Not in the first version

- Locking anything on the site behind having a ghost.
- A Discord role for ghosts, or ghost fleet pings.
- Telling alpha from omega: ESI doesn't expose clone state.
- Removing or kicking anyone.

## Later

- A Ghosts Discord role and a fleet audience, so FCs can ping ghosts for a
  defense fleet.
- Reminders to members who have no ghost, or whose ghost is only partly ready.

## Known gaps

- A ship inside a container has the container as its location, so it doesn't
  count. Members should leave the ship in the hangar.
- The site only sees characters added to it. A ghost that was never added
  counts as no ghost.
- Data is up to 4 hours old, so a member who has just set their clone waits
  for the next sync.

## Decisions for leadership

1. Create the ghost corporation in FL33T, so it is part of the alliance's
   highsec wars. Name and ticker.
2. Which Amo structure is the staging structure.
3. The ghost doctrine: which hulls, which must be flyable on an alpha clone.
4. The ghost skill plan, and the minimum progress for ready (100%, or lower so
   new ghosts count sooner).
5. What happens to members without a ghost: only shown on the dashboard, or
   followed up by CEOs.
