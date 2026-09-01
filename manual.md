# EveStructureBot User Manual

EveStructureBot is a Discord bot that monitors your EVE Online corporation's structures and starbases (POS) and posts alerts to your Discord server. It watches for attacks, low fuel, moon mining events, war declarations, and more — so your corp finds out in Discord before someone logs in and discovers the bad news.

---

## Quick Start

1. **Invite the bot** to your server (see the [README](README.md) for the invite link) and make sure it can view and send messages in the channel you want alerts in.
2. **In the alert channel, run `/auth`.** The bot replies (privately) with a *Log in using Eve Online Single Sign-On* button. Click it and log in with an EVE character that has the right in-game roles (see below). That's it — the bot detects the character's corporation automatically and links it to the channel.
3. **Run `/info`** to confirm the bot is now tracking your corp in that channel.
4. **Optionally run `/configure`** to choose which alert types this channel receives, and `/set_ping` to have a Discord role @-mentioned on fuel or attack alerts.

Have several members authorize characters — the bot spreads its API polling across all authorized characters, so **more authorized characters means faster checks**.

### Which in-game roles do characters need?

The bot can only see what the authorizing character can see:

| EVE corp role | What it enables |
|---|---|
| **Director** | Corporation notifications (attacks, wars, moon mining, POS alerts — POS notifications are only sent to Directors) and starbase lists |
| **Station Manager** | The corporation structure list (Upwell structures, fuel levels) |

For full coverage, authorize at least one Director and at least one Station Manager (one character can be both).

When you authorize, EVE SSO requests these ESI scopes:

- `esi-corporations.read_starbases.v1`
- `esi-corporations.read_structures.v1`
- `esi-corporations.read_corporation_membership.v1`
- `esi-characters.read_notifications.v1`
- `esi-characters.read_corporation_roles.v1`

---

## Configuring a Channel

All configuration is **per channel**. A corporation can be linked to multiple channels (run `/auth` in each), and each channel independently chooses what it wants to see — for example a quiet `#structure-fuel` channel and a loud `#defense-pings` channel.

### `/configure` — choose what this channel receives

Running `/configure` shows five toggle buttons. Green = ON. All five default to **ON** for a new channel:

| Toggle | Controls |
|---|---|
| **POS Fuel Alerts** | Starbase (POS) fuel/resource warnings (e.g. "POS Needs Resources") |
| **POS Status Alerts** | Starbase attack and status events ("POS Under Attack") |
| **Structure Fuel Alerts** | Upwell structure fuel warnings, low-power/abandonment warnings |
| **Structure Status Alerts** | Structure attacks, shield/armor loss, reinforcement, anchoring/unanchoring, destruction, POCO and Skyhook events, sov hub reinforcement |
| **Mining Updates** | Moon mining extraction started/finished, laser fired, auto-fracture (includes estimated chunk composition) |

Click a button to flip it; the change saves immediately.

> Note: war-related notices (War Declared, War Inherited, etc.) are always posted to linked channels and are not affected by these toggles.

### `/set_ping` — who gets @-mentioned

`/set_ping type:<Fuel|Attack> role:<@role>`

Sets the Discord role that gets pinged when an alert of that type fires in this channel:

- **Fuel** — pinged on low fuel, low power, services offline, POS resource, and abandonment warnings.
- **Attack** — pinged on attacks, shield/armor loss, reinforcement, destruction, POCO/Skyhook attacks, and sov hub reinforcement.

Run the command **without the `role` option to remove** the ping for that type.

---

## Command Reference

### Everyday commands

| Command | What it does |
|---|---|
| `/auth` | Get an EVE SSO login link to authorize a character (also used to **re**-authorize when a token expires). Private reply. |
| `/info` | Show what's tracked in this channel: corp details, member/character counts, structure counts, and effective check rates. |
| `/fuel name:<structure>` | Fuel status for a single structure (name autocompletes). |
| `/refuel system:<system>` | Fuel status for **all** structures in a system, sorted soonest-expiring first. Handy for planning a refuel run. |
| `/checkauth` | List characters whose authorization has expired and ping their owners to re-run `/auth`. Also cleans up corps with no members left. |
| `/whois name:<character>` | Show which Discord user owns a given EVE character. |
| `/configure` | Toggle which alert categories this channel receives (see above). |
| `/set_ping` | Set/clear the role pinged on Fuel or Attack alerts (see above). |
| `/hello` | Simple liveness check — the bot says hello back. |

### Data management

| Command | What it does |
|---|---|
| `/remove` | **Deletes all stored data for this channel** (unlinks the corps from it). Asks for confirmation with a button; times out after 1 minute if unconfirmed. |

### Debug / admin commands

These exist mainly for troubleshooting and bot administration; most users never need them.

| Command | What it does |
|---|---|
| `/debug` | Live status panel showing countdowns to the next notification/starbase/structure checks and which character will be used. Updates every few seconds until you press *Turn Debug Mode Off*. |
| `/debug_reprocess duration:<all\|one-week>` | Rewind the notification cursor so old notifications are processed again on the next poll (can repost old alerts). |
| `/reload command:<name>` | Re-register a slash command with Discord (bot admin tool). |
| `/test` | Parse a local test notifications file (developer tool; does nothing useful on a normal server). |

---

## How Monitoring Works

The bot polls the EVE ESI API on a schedule and posts anything new to every channel the corp is linked to (subject to each channel's `/configure` toggles):

- **Corporation notifications** (attacks, wars, mining, POS alerts) are checked roughly every **10 minutes** per corp. EVE caches each character's notifications for 10 minutes, so the bot rotates through your authorized **Directors** — with 5 Directors authorized, it can effectively check every ~2 minutes.
- **Structure and starbase lists** (fuel levels, states) are checked roughly every **hour** per corp, rotated across authorized **Station Managers** (structures) / **Directors** (starbases) the same way.
- `/info` shows your corp's current effective check rates.

### Fuel warnings

Besides relaying CCP's own "Structure low on fuel" notification, the bot watches fuel timers itself and warns when a structure drops below:

- **7 days** of fuel remaining (low fuel warning), and again at
- **2 days** remaining (urgent warning).

It also announces fuel timer *changes* (e.g. someone refueled, or fuel info disappeared) as structure status messages.

### Alert types posted

- **Structures (Upwell):** under attack, shields/armor lost, destroyed, online, low/high power, services offline, anchoring, unanchoring, low fuel, impending abandonment (assets at risk).
- **Starbases (POS):** under attack (with aggressor and shield/armor/hull %), needs fuel/resources.
- **POCOs:** attacked, reinforced.
- **Skyhooks:** deployed, online, under attack, lost shields, destroyed.
- **Moon mining:** extraction started/finished, laser fired, automatic fracture — with estimated ore composition.
- **Wars:** declared, inherited, HQ destroyed, retracted by CONCORD, no longer war eligible, corp joined an alliance at war.
- **Sovereignty:** sov hub reinforced.

Attack alerts include the aggressor's name (linked to zKillboard), corp, alliance, remaining shield/armor/hull, and reinforcement exit times as Discord relative timestamps.

---

## Keeping It Healthy

- **Tokens expire.** When a character's ESI token can no longer be refreshed, the bot marks it as needing re-auth. Run `/checkauth` periodically (or when alerts seem quiet) — it pings owners of expired characters to re-run `/auth`.
- **Coverage depends on roles.** If you see notifications but no structure fuel data, you're missing an authorized Station Manager (and vice versa for Directors).
- **More characters = faster alerts.** Polling is divided among authorized characters, so encourage a few Directors to authorize.
- **Leaving?** `/remove` unlinks the channel and deletes its channel settings. Members can also revoke the bot's ESI access at any time from EVE's [third-party application support page](https://community.eveonline.com/support/third-party-applications/).

## Getting Help

Join the *EVE Apps by Lak Moore* Discord: https://discord.gg/9xgRvQf5A
