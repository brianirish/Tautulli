# Reporting Exclusions — Design Spec

**Date:** 2026-04-13
**Status:** Approved

## Problem

There is no way to exclude specific Plex users from Tautulli's reporting and statistics. The existing `keep_history` flag only prevents *new* playback from being recorded — it does not filter existing history from home stats, graphs, or history views. Users (e.g., a child's profile) who watch frequently can skew "most popular" rankings and clutter usage reports for the account owner.

## Goal

A global, easily toggleable list of users whose data is kept in the database but excluded from all reporting — retroactively and going forward.

---

## Design

### 1. Schema & Migration

Add a new column to the `users` table:

```sql
ALTER TABLE users ADD COLUMN exclude_from_reports INTEGER DEFAULT 0
```

- Default `0` (not excluded) for all existing users — no behavior change on upgrade.
- Bump `db_version` in `plexpy/__init__.py` and add the migration to the version upgrade block.
- Fully reversible: setting the flag back to `0` immediately re-includes the user in all reports.

### 2. Query Filtering

Apply the exclusion filter at the SQL level in two files:

**`plexpy/datafactory.py`**
- `get_home_stats()` — covers top movies, popular TV/movies/music, top users, top platforms, most concurrent, last watched, top libraries.
- `get_datatables_history()` — the main history table view.
- Any other methods that query `session_history` and aggregate across users.
- Filter condition: `AND (users.exclude_from_reports = 0 OR users.exclude_from_reports IS NULL)`
- Applied via the existing `custom_where` pattern where available; inline otherwise.
- Queries already JOIN `session_history` to `users` on `user_id` in most cases; where they don't, add the JOIN.

**`plexpy/graphs.py`**
- All time-series graph queries (plays by date, play duration, stream type, etc.).
- Extend the existing `_make_user_cond()` helper to append the exclusion filter so all graph queries inherit it automatically.

Filtering is done in SQL, not in Python post-processing. Excluded users have zero effect on counts, totals, rankings, or averages.

### 3. Backend & API

**`plexpy/users.py`**
- `set_config()`: add `exclude_from_reports` as an accepted/saved field, consistent with `keep_history`, `do_notify`, etc.
- New method `get_users_list()` (or extend existing): returns all users with their `exclude_from_reports` status for populating the settings UI.

**`plexpy/webserve.py`**
- New endpoint `set_user_exclude_from_reports(user_id, exclude)`: accepts a user_id and boolean, calls `users.Users().set_config()`, returns JSON success/error. Follows the same pattern as `set_user_config`.

### 4. UI

A new **"Reporting Exclusions"** section on the existing Settings page, placed under the Users grouping.

- Renders a list of all Plex users: avatar, friendly name, and an on/off toggle.
- Toggling fires the new endpoint via AJAX — no page reload.
- Visually consistent with existing toggle patterns in the Settings UI.
- Gives a single global view: all users visible at once, easy to enable/disable exclusion per user.

---

## What This Does NOT Change

- Existing `keep_history` behavior is unchanged (still stops new recording only).
- Excluded users' data remains in the database intact.
- Per-user detail pages and the Users list page are unaffected (those show raw data, not aggregated reports).
- Notifications (`do_notify`) are unaffected.

---

## Non-Goals

- Deleting or anonymizing excluded users' data.
- Filtering excluded users from the active sessions view (that's a live feed, not a report).
- Per-library exclusions.
