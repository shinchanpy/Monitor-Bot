# Telegram Adbot

Telegram account manager and forwarding engine for multiple accounts. The
application has two layers:

- `launcher.py` is the global menu for projects and shared settings.
- `main.py` runs one selected project and its account forwarding engine.

## First-time setup

1. Create a Telegram application at <https://my.telegram.org> and copy its
	numeric **API ID** and 32-character **API hash**.
2. Create a bot with `@BotFather` if channel logging is needed. Copy the bot
	token once; treat it like a password.
3. Install dependencies:

	```text
	python -m pip install -r requirements.txt
	```

4. Start the global menu:

	```text
	python -X utf8 launcher.py
	```

5. Open `/settings` and enter the API ID, API hash, and logging bot token.
	Values are entered through hidden prompts. Addlist URLs can be entered on
	the same screen.
6. Create or open a project, use `/add` to log in accounts, then configure
	`/msg`, `/groups`, and `/logs` before enabling forwarding.

## Where data is stored

| Data | File or directory | Purpose |
| --- | --- | --- |
| Telegram API ID/hash and logging bot token | `workspace/shared.json` | Shared by all projects. This file is local secret material. |
| Telegram account owner mapping | `workspace/owners.json` | Maps Telegram user ID to `[project, account_id]`. |
| Project registry | `workspace/projects.json` | Maps project names to their directories. |
| Account settings | `data/config.json` or `projects/*/data/config.json` | Phone metadata and forwarding settings. |
| Account state | `data/state.json` or `projects/*/data/state.json` | Skip lists, waits, bans, and delivery state. |
| Login sessions | `sessions/` or `projects/*/sessions/` | Telethon session files; these are equivalent to passwords. |
| Topic lists | `groups/*.txt` or project `groups/*.txt` | Destination chats grouped by topic. |

### Owner ID behavior

There is no owner ID to type into configuration. When an account logs in,
Telethon returns its Telegram user ID. The application records that ID in the
account config and claims it in `workspace/owners.json`. The claim prevents
the same Telegram identity from being assigned to two projects. A user ID is
not a bot token and cannot log an account in by itself.

### Secret handling

Keep `workspace/shared.json`, `workspace/owners.json`, all `sessions/`
directories, and project `data/` directories private. `.gitignore` excludes
these files and generated reports, but it cannot remove secrets already copied
elsewhere. Never paste a bot token into source code, README files, issue
reports, or chat. If a token is exposed, revoke it with `@BotFather` and issue
a new one immediately. A Telethon session file also requires immediate
revocation by logging that Telegram session out.

The current workspace contains credentials in `workspace/shared.json`; rotate
the bot token before sharing or uploading this folder.

## How the application works

`launcher.py` initializes the workspace, creates/selects projects, and starts
`main.py` with `ADBOT_PROJECT` set. `workspace.py` owns project locks, shared
settings, owner claims, account allocation, and crash recovery. `main.py`
loads settings, connects each Telethon session, validates source messages and
destination permissions, then forwards eligible messages on the configured
cycle. `destination_permissions.py` maintains media-specific allowlists, and
`log_destination.py` verifies logging channels through the Bot API.

The app keeps account state separate, uses locks to prevent concurrent project
changes, archives removed sessions rather than deleting them, and records
temporary waits or failures so a retry does not create duplicate sends.

## Useful commands

- `/projects`, `/create`, `/settings`, `/replacements`, `/recover`, `/exit`
- `/add`, `/groups`, `/msg`, `/logs`, `/routes`
- `/status`, `/start`, `/stop`, `/help`

Run the offline tests with:

```text
python -X utf8 -m unittest discover -s tests -v
```

The audit and probe scripts are optional maintenance tools. They produce
reports that can be deleted and regenerated; they are not required to start
the main application.

Start with `python -X utf8 launcher.py`. Reopen existing windows to load code changes.

The global screen uses one animated cfonts block header and a double-line input box.

## Everyday controls

- Global: `/projects`, `/create`, `/settings`, `/replacements`, `/recover`, `/exit`.
- Replacement menu: `/add`, `/remove`, `/accounts`, `/allocate`, `/back`. Account management opens its own window. Close replacement and destination windows before allocation; the application enforces session locks.
- Project setup: `/add` → `/groups` → `/msg` → `/logs`. `/msg` guides you through account selection, source message, destination list, and a save preview. `/routes` keeps advanced rotation separate.
- Run: `/status`, `/start`, `/stop`. `/help` groups commands by purpose.
- Scroll output with the mouse wheel, PageUp/PageDown, or Ctrl+Home/Ctrl+End. The last 5,000 logical lines are retained; individual long lines are wrapped rather than truncated.
- `/effects`: rainbow, ultra, normal, calm, off. `ADBOT_REDUCED_MOTION=1` or `NO_COLOR` disables effects. Smaller terminals use a compact header.

Phone login accepts Ctrl+V / Shift+Insert and pasted spaces, parentheses, hyphens, or a `00` international prefix. A missing `+` is added automatically; the number must still include its country code.

Every successful `/add` or `/login`, including an existing account, now replaces imported addlists with the globally configured lists. Incoming links are validated before old folders are removed. Old-only chats are left; incoming chats and message-source chats are protected. Cleanup errors stop import and are reported. `python -X utf8 sync_all_addlists.py` applies this to all active configured sessions; close project windows first. Results are saved in `addlist-sync-results.json`. Archived/removed sessions are not reactivated.

In `/profile`, enter a new first name to replace the full old name; an omitted new last name clears the old surname and keeps the Adbot marker. Bio text replaces the existing bio; `-` clears it. A new photo is validated and staged, old profile photos are removed, and the new photo is installed and checked. Partial upload/cleanup failures are reported explicitly. Blank unrelated fields remain unchanged.

## Logging

`/logs` accepts a channel username, public channel link, private `t.me/c/ID` channel link, or numeric `-100…` ID. Private invitation links cannot be resolved by the logging bot. Add the bot as a channel administrator first.

Addresses are verified through Bot API `getChat` and stored as canonical IDs, preserving separation between projects. Enter `-` to disable logging; blank keeps the saved destination. Logs use escaped `<b>` HTML with explicit HTML parse mode. When message links are off, normal groups link to the group; forum topics link to their resolved topic. Source-list category names are not appended as if they were Telegram topics.

## Topic import and permission audit

The provided Downloads/Groups.txt and FORUMS.txt are imported with platform-specific forums first, followed by common groups and remaining existing entries. Duplicate targets are collapsed. TikTok and Twitter/X map to TikTok-X; OnlyFans maps to OFM. Other unmatched forum sections are not mixed into unrelated topics.

`python -X utf8 audit_destinations.py --inspect` checks existing configured sessions without joining chats or sending test posts. Close project windows first. The audit stops on Telegram FloodWait and saves its deadline. Re-run after that deadline to finish a partial audit. `--import-lists` repeats the import and creates backups first.

Per-project outputs:

- `permissions/ACCOUNT.json`: checked permissions, timestamps, and unknown/error reasons.
- `media_targets/ACCOUNT/TYPE/TOPIC.txt`: separate text, photo, video, document, GIF, sticker, voice, audio, round-video, game, and poll lists.
- `permissions/removed-TIMESTAMP/`: original topic files before removing destinations whose posting permissions were denied for every inspected project session.
- `review_targets/ACCOUNT/TOPIC.txt`: unresolved destinations retained for review.

The root `destination-audit-summary.json` records completion and counts. Not-a-participant, inaccessible private chats, failed lookups, and connection errors remain **unknown**; they are not evidence of a global posting ban. Permission checks cannot detect every moderation bot rule or guarantee that an administrator will accept an advertisement.

`/msg` previews the media type when the selected account can read the message. The sender checks it again before forwarding and periodically refreshes edited sources. Media sending uses the matching account/type allowlist and verifies new or stale entries; unknown media permissions prevent dispatch. A media-specific rejection does not skiplist the destination for text or another media type.

## Retry behavior

Telegram FloodWait is an account deadline; SlowModeWait is a destination deadline and does not block the rest of the target list. Repeated temporary destination failures back off from 15 minutes toward the configured maximum; a successful send clears that history. These deadlines make a destination eligible again; they do not bypass the configured cycle interval. Network retries grow exponentially while preserving the original delivery ID. Message and cycle delay settings are lower bounds.

Account restrictions still require review. This update does not clear existing bans, spam flags, or Telegram-mandated waits.

## Explicit posting tests

`probe_destinations.py` is different from the read-only audit: it sends temporary text/photo permission checks to unverified group/forum destinations and deletes successful tests. Run it only when posting tests are intended. It never joins new chats. A wait or account restriction stops testing; failed cleanup leaves a `permissions/ACCOUNT-test-cleanup.json` record and stops further tests.

After the probe has exited, `python -X utf8 finalize_destinations.py` applies recorded denials, backs up affected files, and regenerates topic/media/review lists. Shared topic removal requires unanimous evidence from available sessions. Results are in `final-destination-results.json`; individual send outcomes are in `destination-probe-summary.json`.

## Validation and recovery

Run `python -X utf8 -m unittest discover -s tests -v`.

Audit and import scripts may create temporary backups while they run. Existing
credentials and sessions are not replaced by normal startup. A passing offline
suite is not a guarantee against changing Telegram permissions or future
server errors.

Permission semantics follow [Telegram's rights documentation](https://core.telegram.org/api/rights) and [forum topic documentation](https://core.telegram.org/api/forum).
