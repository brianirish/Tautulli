# Reporting Exclusions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a global, per-user `exclude_from_reports` flag that hides selected users from all aggregated Tautulli reports and stats while keeping their history data in the database.

**Architecture:** Add `exclude_from_reports INTEGER DEFAULT 0` to the `users` table. Apply a NOT IN subquery filter to all stat/reporting queries in `datafactory.py` and `graphs.py`. Add a `custom_where` filter to the history datatables query (which already JOINs users). Expose a `set_exclude_from_reports` endpoint and a Settings UI section that loads users via AJAX and toggles the flag per-user.

**Tech Stack:** Python, SQLite (via Tautulli's `database.MonitorDatabase`), CherryPy, Mako templates, jQuery/AJAX

---

## File Map

| File | Change |
|------|--------|
| `plexpy/__init__.py` | Add column to CREATE TABLE users; add ALTER TABLE migration |
| `plexpy/users.py` | Add `exclude_from_reports` to `set_config()` and `get_users()` |
| `plexpy/datafactory.py` | Add exclusion filter to all `get_home_stats()` stat blocks + `get_datatables_history()` |
| `plexpy/graphs.py` | Extend `_make_user_cond()` to append exclusion condition |
| `plexpy/webserve.py` | Add `set_exclude_from_reports` endpoint |
| `data/interfaces/default/settings.html` | Add Reporting Exclusions section with AJAX user list + toggle |

---

## Task 1: Schema — Add column to users table

**Files:**
- Modify: `plexpy/__init__.py:710-719` (CREATE TABLE users)
- Modify: `plexpy/__init__.py:2683` (migration insertion point, after the imgur_lookup block)

- [ ] **Step 1: Add column to CREATE TABLE users**

In `plexpy/__init__.py`, find the CREATE TABLE users statement (around line 710). Change:

```python
        "filter_all TEXT, filter_movies TEXT, filter_tv TEXT, filter_music TEXT, filter_photos TEXT)"
```

To:

```python
        "filter_all TEXT, filter_movies TEXT, filter_tv TEXT, filter_music TEXT, filter_photos TEXT, "
        "exclude_from_reports INTEGER DEFAULT 0)"
```

- [ ] **Step 2: Add ALTER TABLE migration**

In `plexpy/__init__.py`, after the imgur_lookup try/except block (around line 2683), insert:

```python
    try:
        c_db.execute("SELECT exclude_from_reports FROM users")
    except sqlite3.OperationalError:
        logger.debug("Altering database. Adding exclude_from_reports column to users table.")
        c_db.execute(
            "ALTER TABLE users ADD COLUMN exclude_from_reports INTEGER DEFAULT 0"
        )
```

- [ ] **Step 3: Verify**

Start the Tautulli server and open a SQLite browser (or run `sqlite3 tautulli.db`) against the database file. Run:

```sql
SELECT user_id, username, exclude_from_reports FROM users LIMIT 5;
```

Expected: column exists, all rows show `0`.

- [ ] **Step 4: Commit**

```bash
git add plexpy/__init__.py
git commit -m "feat: add exclude_from_reports column to users table"
```

---

## Task 2: Backend — users.py

**Files:**
- Modify: `plexpy/users.py:354` (`set_config`)
- Modify: `plexpy/users.py:675` (`get_users`)

- [ ] **Step 1: Add to set_config()**

Find `set_config` (line 354). Change the signature and value_dict:

```python
def set_config(self, user_id=None, friendly_name='', custom_thumb='', do_notify=1, keep_history=1, allow_guest=1, exclude_from_reports=0):
    if str(user_id).isdigit():
        monitor_db = database.MonitorDatabase()

        user = monitor_db.select_single('SELECT username FROM users WHERE user_id = ?', [user_id])
        if user.get('username') == friendly_name:
            friendly_name = None

        key_dict = {'user_id': user_id}
        value_dict = {'friendly_name': friendly_name,
                      'custom_avatar_url': custom_thumb,
                      'do_notify': do_notify,
                      'keep_history': keep_history,
                      'allow_guest': allow_guest,
                      'exclude_from_reports': exclude_from_reports
                      }
        try:
            monitor_db.upsert('users', value_dict, key_dict)
        except Exception as e:
            logger.warn("Tautulli Users :: Unable to execute database query for set_config: %s." % e)
```

- [ ] **Step 2: Add to get_users() query**

Find `get_users` (line 675). Change the SELECT query:

```python
            query = "SELECT id AS row_id, user_id, username, friendly_name, thumb, custom_avatar_url, email, " \
                    "is_active, is_admin, is_home_user, is_allow_sync, is_restricted, " \
                    "do_notify, keep_history, allow_guest, shared_libraries, exclude_from_reports, " \
                    "filter_all, filter_movies, filter_tv, filter_music, filter_photos " \
                    "FROM users %s" % where
```

And add the field to the returned user dict (around line 701, after `'allow_guest': item['allow_guest']`):

```python
                    'allow_guest': item['allow_guest'],
                    'exclude_from_reports': item['exclude_from_reports'],
```

- [ ] **Step 3: Commit**

```bash
git add plexpy/users.py
git commit -m "feat: add exclude_from_reports to users set_config and get_users"
```

---

## Task 3: Webserve endpoint — set_exclude_from_reports

**Files:**
- Modify: `plexpy/webserve.py` (add endpoint near `edit_user` around line 1394)

- [ ] **Step 1: Add the endpoint**

In `plexpy/webserve.py`, after the closing of `edit_user` (around line 1394), add:

```python
    @cherrypy.expose
    @cherrypy.tools.json_out()
    @requireAuth(member_of("admin"))
    def set_exclude_from_reports(self, user_id=None, exclude=0, **kwargs):
        if user_id:
            try:
                monitor_db = database.MonitorDatabase()
                monitor_db.action(
                    'UPDATE users SET exclude_from_reports = ? WHERE user_id = ?',
                    [int(exclude), user_id]
                )
                return {'result': 'success', 'message': 'Updated successfully.'}
            except Exception as e:
                logger.warn("Tautulli WebInterface :: Unable to set exclude_from_reports: %s." % e)
                return {'result': 'error', 'message': 'Failed to update.'}
        return {'result': 'error', 'message': 'Missing user_id.'}
```

- [ ] **Step 2: Verify endpoint exists**

Start Tautulli (if not already running) and confirm the route exists by navigating to:
`http://localhost:8181/set_exclude_from_reports?user_id=1&exclude=0`

Expected: JSON `{"result": "success", "message": "Updated successfully."}` (or an error with the correct structure if user_id 1 doesn't exist).

- [ ] **Step 3: Commit**

```bash
git add plexpy/webserve.py
git commit -m "feat: add set_exclude_from_reports API endpoint"
```

---

## Task 4: datafactory.py — Exclusion filter for home stats (media cards)

**Files:**
- Modify: `plexpy/datafactory.py` (`get_home_stats`, multiple stat blocks)

This task covers: `top_movies`, `popular_movies`, `top_tv`, `popular_tv`, `top_music`, `popular_music`.

All six use the same inner subquery pattern:
```sql
FROM session_history
WHERE session_history.media_type = 'X' %s %s
GROUP BY %s
```

- [ ] **Step 1: Define where_exclude variables at top of get_home_stats()**

In `get_home_stats()`, after the `where_id` / `where_id_args` setup block (after line 391), add:

```python
        where_exclude = ("AND session_history.user_id NOT IN "
                         "(SELECT user_id FROM users WHERE exclude_from_reports = 1 "
                         "AND user_id IS NOT NULL) ")
        where_exclude_sh = ("AND sh.user_id NOT IN "
                            "(SELECT user_id FROM users WHERE exclude_from_reports = 1 "
                            "AND user_id IS NOT NULL) ")
```

(`where_exclude_sh` uses the `sh.` alias needed by `most_concurrent` in Task 5.)

- [ ] **Step 2: Patch top_movies inner WHERE clause (around line 410)**

Change:
```python
                            "   WHERE session_history.media_type = 'movie' %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "GROUP BY shm.full_title, shm.year " \
                            "ORDER BY %s DESC, sh.started DESC " \
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, group_by, sort_type, stats_count, stats_start)
```

To:
```python
                            "   WHERE session_history.media_type = 'movie' %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "GROUP BY shm.full_title, shm.year " \
                            "ORDER BY %s DESC, sh.started DESC " \
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 3: Patch popular_movies (around line 464)**

Same change — add `%s` to the inner WHERE and `where_exclude` to the format tuple:

```python
                            "   WHERE session_history.media_type = 'movie' %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 4: Patch top_tv (around line 516)**

```python
                            "   WHERE session_history.media_type = 'episode' %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 5: Patch popular_tv (around line 571)**

```python
                            "   WHERE session_history.media_type = 'episode' %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 6: Patch top_music (around line 623)**

```python
                            "   WHERE session_history.media_type = 'track' %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 7: Patch popular_music (around line 677)**

```python
                            "   WHERE session_history.media_type = 'track' %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe, where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 8: Commit**

```bash
git add plexpy/datafactory.py
git commit -m "feat: filter excluded users from home stats media cards"
```

---

## Task 5: datafactory.py — Exclusion filter for remaining home stats

**Files:**
- Modify: `plexpy/datafactory.py` (stat blocks: `top_libraries`, `top_users`, `top_platforms`, `last_watched`, `most_concurrent`)

These five use a different WHERE pattern: `WHERE %s %s` where the first `%s` is `where_timeframe[4:]` (strips leading "AND ").

- [ ] **Step 1: Patch top_libraries inner WHERE (around line 734)**

Change:
```python
                            "   WHERE %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe[4:], where_id, group_by, sort_type, stats_count, stats_start)
```

To:
```python
                            "   WHERE %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe[4:], where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 2: Patch top_users inner WHERE (around line 823)**

Change:
```python
                            "   WHERE %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe[4:], where_id, group_by, sort_type, stats_count, stats_start)
```

To:
```python
                            "   WHERE %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe[4:], where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 3: Patch top_platforms inner WHERE (around line 892)**

Same change as top_users:
```python
                            "   WHERE %s %s %s " \
                            "   GROUP BY %s) AS sh " \
```
```python
                            "LIMIT %s OFFSET %s " % (where_timeframe[4:], where_id, where_exclude, group_by, sort_type, stats_count, stats_start)
```

- [ ] **Step 4: Patch last_watched inner WHERE (around line 987)**

The inner WHERE clause reads:
```python
                            "   WHERE (session_history.media_type = 'movie' " \
                            "           OR session_history.media_type = 'episode') %s %s " \
```

Change to:
```python
                            "   WHERE (session_history.media_type = 'movie' " \
                            "           OR session_history.media_type = 'episode') %s %s %s " \
```

The `query` assignment ends with a `% (...)` format call applied to the whole concatenated string. That format tuple (around line 995) currently reads:
```python
                            "LIMIT %s OFFSET %s" % (watched_threshold,
                                                    where_timeframe, where_id, group_by, watched_where,
                                                    stats_count, stats_start)
```

Change to (add `where_exclude` after `where_id`):
```python
                            "LIMIT %s OFFSET %s" % (watched_threshold,
                                                    where_timeframe, where_id, where_exclude, group_by, watched_where,
                                                    stats_count, stats_start)
```

`monitor_db.select(query, args=where_timeframe_args + where_id_args)` is unchanged — `where_exclude` has no `?` placeholders.

- [ ] **Step 5: Patch most_concurrent base_query (around line 1082)**

The `most_concurrent` query uses `sh.` alias. Change `base_query`:

```python
                    base_query = "SELECT sh.started, sh.stopped " \
                                 "FROM session_history AS sh " \
                                 "JOIN session_history_media_info AS shmi ON sh.id = shmi.id " \
                                 "WHERE %s " % where_timeframe[4:].replace('session_history.', 'sh.')
```

To:

```python
                    base_query = "SELECT sh.started, sh.stopped " \
                                 "FROM session_history AS sh " \
                                 "JOIN session_history_media_info AS shmi ON sh.id = shmi.id " \
                                 "WHERE %s %s " % (
                                     where_timeframe[4:].replace('session_history.', 'sh.'),
                                     where_exclude_sh
                                 )
```

- [ ] **Step 6: Commit**

```bash
git add plexpy/datafactory.py
git commit -m "feat: filter excluded users from remaining home stats and most_concurrent"
```

---

## Task 6: datafactory.py — Filter history table

**Files:**
- Modify: `plexpy/datafactory.py:134-200` (`get_datatables_history`)

The history datatables query already LEFT OUTER JOINs `users` on `session_history.user_id = users.user_id`. We can use the existing `custom_where` mechanism. The `custom_where_union` (for the live activity union) is built from `custom_where` *before* our insertion point, so it won't affect the sessions UNION.

- [ ] **Step 1: Add exclusion to custom_where after the include_activity block**

In `get_datatables_history`, find the `if include_activity:` block (lines 134-138). After it closes (after the `group_by_union = ['session_key']` line), add:

```python
        # Exclude users flagged for report exclusion (applied after union to avoid sessions table schema conflict)
        custom_where.append(['users.exclude_from_reports', [0, None]])
```

The `build_custom_where` function handles `[0, None]` as a list, generating:
`(users.exclude_from_reports = 0 OR users.exclude_from_reports IS NULL) AND`

This correctly shows rows where:
- `exclude_from_reports = 0` (not excluded)
- `exclude_from_reports IS NULL` (user not found via LEFT OUTER JOIN — keep those too)

And hides rows where `exclude_from_reports = 1`.

- [ ] **Step 2: Verify the code position**

Read `get_datatables_history` from line 134 to ~145 to confirm `custom_where_union` is set before your new line, not after.

Expected structure after edit:
```python
        if include_activity:
            table_name_union = 'sessions'
            custom_where_union = [[c[0].split('.')[-1], c[1]] for c in custom_where]
            group_by_union = ['session_key']
            columns_union = [...]

        # Exclude users flagged for report exclusion
        custom_where.append(['users.exclude_from_reports', [0, None]])

        try:
            query = data_tables.ssp_query(...)
```

- [ ] **Step 3: Commit**

```bash
git add plexpy/datafactory.py
git commit -m "feat: filter excluded users from history datatables query"
```

---

## Task 7: graphs.py — Filter excluded users from all graphs

**Files:**
- Modify: `plexpy/graphs.py:1233-1244` (`_make_user_cond`)

- [ ] **Step 1: Extend _make_user_cond()**

Find `_make_user_cond` (line 1233). Replace the entire method:

```python
    def _make_user_cond(self, user_id, cond_prefix='AND'):
        """
        Expects user_id to be a comma-separated list of ints.
        """
        user_cond = ''
        if session.get_session_user_id():
            user_cond = cond_prefix + ' session_history.user_id = %s ' % session.get_session_user_id()
        elif user_id:
            user_ids = helpers.split_strip(user_id)
            if all(id.isdigit() for id in user_ids):
                user_cond = cond_prefix + ' session_history.user_id IN (%s) ' % ','.join(user_ids)

        user_cond += ('AND session_history.user_id NOT IN '
                      '(SELECT user_id FROM users WHERE exclude_from_reports = 1 '
                      'AND user_id IS NOT NULL) ')
        return user_cond
```

Note: This always appends the exclusion with `AND`. It is safe because `_make_user_cond()` is only ever used after an existing WHERE condition (e.g., `WHERE session_history.stopped >= timestamp`), so `AND` is always a valid prefix.

- [ ] **Step 2: Commit**

```bash
git add plexpy/graphs.py
git commit -m "feat: filter excluded users from all graph queries"
```

---

## Task 8: Settings UI — Reporting Exclusions section

**Files:**
- Modify: `data/interfaces/default/settings.html` (around line 257, before Updates section)

- [ ] **Step 1: Add the HTML section and JavaScript**

In `settings.html`, find the closing `</div>` of the "History Logging" section (around line 257, just before `<div class="padded-header"><h3>Updates</h3></div>`). Insert the following block:

```html
                        <div class="padded-header">
                            <h3>Reporting Exclusions</h3>
                        </div>

                        <p class="help-block">
                            Users checked below will be hidden from all reports, statistics, history tables,
                            and graphs. Their data is kept in the database and can be re-included at any time.
                        </p>
                        <div id="reporting-exclusions-list">
                            <p class="text-muted"><em>Loading users...</em></p>
                        </div>
```

- [ ] **Step 2: Add the JavaScript**

Find the `<script>` block in `settings.html` (near the bottom of the file). Add the following inside it (e.g., before the closing `</script>` tag or after the last existing function):

```javascript
// Reporting Exclusions
function loadReportingExclusions() {
    $.ajax({
        url: 'get_full_users_list',
        cache: false,
        async: true,
        success: function(data) {
            var container = $('#reporting-exclusions-list');
            container.empty();
            if (!data || data.length === 0) {
                container.html('<p class="text-muted"><em>No users found.</em></p>');
                return;
            }
            var html = '<ul class="list-unstyled">';
            $.each(data, function(i, user) {
                var thumb = user.thumb || '/images/default_user_thumb.png';
                var name = user.friendly_name || user.username;
                var checked = user.exclude_from_reports ? 'checked' : '';
                html += '<li style="margin-bottom: 8px;">' +
                    '<label style="font-weight: normal; cursor: pointer;">' +
                    '<input type="checkbox" class="exclude-from-reports-toggle" ' +
                        'data-user-id="' + user.user_id + '" ' + checked + ' ' +
                        'style="margin-right: 6px;"> ' +
                    '<img src="' + thumb + '" width="20" height="20" ' +
                        'style="border-radius: 50%; margin-right: 6px; vertical-align: middle;" ' +
                        'onerror="this.src=\'/images/default_user_thumb.png\'"> ' +
                    name +
                    '</label></li>';
            });
            html += '</ul>';
            container.html(html);

            // Bind toggle handler
            container.on('change', '.exclude-from-reports-toggle', function() {
                var userId = $(this).data('user-id');
                var exclude = $(this).is(':checked') ? 1 : 0;
                $.ajax({
                    url: 'set_exclude_from_reports',
                    data: { user_id: userId, exclude: exclude },
                    cache: false,
                    async: true
                });
            });
        },
        error: function() {
            $('#reporting-exclusions-list').html('<p class="text-danger">Failed to load users.</p>');
        }
    });
}

loadReportingExclusions();
```

- [ ] **Step 3: Verify in browser**

Start Tautulli, navigate to Settings > General. Scroll to the "Reporting Exclusions" section. Confirm:
- User list loads with avatars and names
- Checking a user's box sends an AJAX request (check browser Network tab)
- Refreshing the page preserves the checked state (because `get_full_users_list` returns the current `exclude_from_reports` value)

- [ ] **Step 4: Commit**

```bash
git add data/interfaces/default/settings.html
git commit -m "feat: add Reporting Exclusions section to Settings UI"
```

---

## Task 9: End-to-end verification

No automated tests exist in this project. Verify manually.

- [ ] **Step 1: Set up test state**

In Settings > General > Reporting Exclusions, check the box next to "Norah" (or whichever test user). Then navigate to the home page.

- [ ] **Step 2: Verify home stats**

Confirm that:
- "Most Watched" movie/TV/music cards no longer show titles that only Norah watched
- "Most Popular" cards no longer count Norah in `users_watched`
- "Most Active Users" card does not include Norah
- "Most Active Platforms" and "Most Active Libraries" counts reflect only non-excluded users

- [ ] **Step 3: Verify history table**

Go to History. Confirm Norah's play history rows are no longer visible.

- [ ] **Step 4: Verify graphs**

Open any play-count graph (e.g., "Plays by Date"). Confirm Norah's plays are not counted.

- [ ] **Step 5: Verify individual user page is unaffected**

Navigate to Norah's user page directly (Users > click Norah). Confirm her history still shows there — the exclusion only applies to global/aggregate views.

- [ ] **Step 6: Verify toggle is reversible**

Uncheck Norah in Settings > Reporting Exclusions. Refresh the home page and confirm her data reappears everywhere.

- [ ] **Step 7: Final commit (if any cleanup needed)**

```bash
git add -p
git commit -m "fix: reporting exclusion cleanup after verification"
```
