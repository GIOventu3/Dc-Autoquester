# Orion

[![Version](https://img.shields.io/badge/v4.11.5-5865F2?style=for-the-badge&logo=discord&logoColor=white)](https://github.com/nyxxbit/discord-quest-completer/releases/latest)
[![Stars](https://img.shields.io/github/stars/nyxxbit/discord-quest-completer?style=for-the-badge&color=faa61a)](https://github.com/nyxxbit/discord-quest-completer/stargazers)
[![License](https://img.shields.io/badge/MIT-green?style=for-the-badge)](LICENSE)

Completes Discord Quests without playing them. It reads the quests you're eligible for, tells Discord you're doing the thing the quest asks for, and waits for Discord to credit the progress.

It handles all five quest types: play a game, stream a game, watch a video, join an activity, and earn an achievement inside an activity. That last one is the reason this project exists, and none of the other tools listed at the bottom of this page do it.

Two ways to run it. A single userscript you paste into DevTools, or a Vencord plugin that starts with Discord. Same engine, kept in sync, in this repo.

> [!CAUTION]
> **Discord has been enforcing against quest automation since April 2026.** People have had system messages land on their account after running automation, any automation, not only this. That is the trade you are making.
>
> The one case reported here with details attached, [#59](https://github.com/nyxxbit/discord-quest-completer/issues/59), was a **14 day quest restriction** after four to five weeks of consistent use: a system notification citing quest automation, quests locked, and nothing else on the account visibly touched. Read that last part narrowly: the datamine account below reports an Account Standing violation attached to these, and a strike on the account record is not something the person holding the account would necessarily see. Orion reads that state (`questAccessSuspendedUntil`) and stops with the date rather than failing quietly. One report is not a pattern, and nothing stops Discord from acting on the whole account instead. Assume it can.
>

## Quick start, userscript

1. Close Discord Fully If Opened
2. Download [Vencord](https://vencord.dev/) available for Win, Mac, Linux, (We don't want the browser version)
3. Install your Vencord, if you need help installing, just ask AI, it's way too easy for me to show you all the steps.
4. After patch, open discord
5.`Ctrl + Shift + I`, Console tab.
6. Type `allow pasting`
7. Paste [`index.js`](index.js) and press Enter.

A quest picker appears. Choose what to run and hit start. `Shift + .` hides and shows the dashboard, and STOP ends the run and undoes everything it patched.

<details>
<summary> Enabling the console on Discord Stable</summary>

Close Discord, edit `%appdata%/discord/settings.json`:

```json
{ "DANGEROUS_ENABLE_DEVTOOLS_ONLY_ENABLE_IF_YOU_KNOW_WHAT_YOURE_DOING": true }

```
Restart Discord.
</details>


## Notice
You only need to put `allow pasting` once, after that just copy all the code in [`index.js`](index.js), enter and start quest, THAT FUCKING EASY.

***




## Compatibility
Works on windows
Works on Mac
Works on Linux (doesn't work on vesktop)

## What it does per quest type

Orion reads Discord's own webpack stores and drives Discord's own authenticated API client. It does not talk to any server of ours; there isn't one.

| Quest task | How it's completed |
|---|---|
| `PLAY_ON_DESKTOP` | Injects a fake running process into `RunningGameStore`, built from the app's real metadata. Discord then sends the quest heartbeats itself and Orion reads the progress off the replies. |
| `STREAM_ON_DESKTOP` | **Does not work.** Same idea, plus a spoofed `getStreamerActiveStreamMetadata`, but Discord checks two other things first and the quest never gets a heartbeat. See below. |
| `WATCH_VIDEO`, `WATCH_VIDEO_ON_MOBILE` | Posts video progress timestamps on a randomized interval, with the fractional values a real player would send. |
| `PLAY_ACTIVITY` | Heartbeats against a voice channel stream key. |
| `ACHIEVEMENT_IN_GAME` | **Not possible.** The achievement is earned in the retail game with the game linked to your account, and nothing in Discord can stand in for that. Named in the log and skipped. |
| `ACHIEVEMENT_IN_ACTIVITY` | Tries the heartbeat first. Discord rejects those with a 403, because the activity backend validates them rather than the client, so it falls back to the OAuth path described below. |

Where the quest's application id lives moved in July 2026, from `config.application.id` to per task at `config.taskConfigV2.tasks.<KEY>.applications[0].id`. Reading the old path fails silently rather than loudly, which is what broke every tool in this space at once. See [#43](https://github.com/nyxxbit/discord-quest-completer/issues/43).

### Stream quests

Before Discord reads the metadata Orion fakes, it requires that you are really Going Live and that at least one other person is in the voice channel with you. Faking the third check alone leaves the first two failing, so no heartbeat is ever opened and the task aborts on its 90 second watchdog instead of completing. Measured on Stable 1.0.9255 and Canary 1.0.1148. The reasoning is in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md#stream_on_desktop-does-not-complete-and-the-spoof-is-why) and the fix is tracked in [#75](https://github.com/nyxxbit/discord-quest-completer/issues/75).

A quest that offers `STREAM_ON_DESKTOP` alongside `PLAY_ON_DESKTOP` or `WATCH_VIDEO` is now driven through the working task instead. It used to be driven as a stream, which meant a quest with a perfectly good path sat there timing out. Only a quest whose sole task is streaming is affected, and Orion still reports the real reason rather than pretending.

## Achievement quests

These can't be faked client side, so completing one means authorizing the quest's app on your account and reporting progress to `discordsays.com` directly. Read the caution at the top before using it.

Discord's renderer blocks requests to `discordsays.com` outright via CSP, so the request has to leave the renderer. Orion tries, in order:

1. The localhost relay on `127.0.0.1:43210`, if it's running. Discord's CSP allows loopback, so this works with no client mod at all.
2. The Vencord plugin's native module, which runs the request in Electron's main process where CSP doesn't apply.
3. A direct `fetch`, which only works on Discord in a browser.

So on desktop you need either the relay or the plugin. There is no renderer-only way around this; that was tested to exhaustion.

Quests for age-gated or delisted activities are skipped instead of retried. Discord answers `/proxy-tickets` with a 403 and code `50165` for those, and they can't be launched by hand either until you age-verify.

## Settings

The userscript asks at start, in the picker: which quests to run, filters by reward type (Orbs, Avatar Decoration, In-Game, Other), auto-enroll (on), auto-claim (off, because claiming often triggers a captcha), a completion sound (off), and randomized idle gaps between quests (off).

Two things are edited in the `CONFIG` object at the top of `index.js` instead:

```js
const CONFIG = {
    HIDE_ACTIVITY: false,   // turn Discord's "Display current activity as a status
                            // message" off while quests run, and restore it after
    MAX_LOG_ITEMS: 60,      // lines kept in the dashboard log
};
```

The plugin has the same options as real settings, plus auto-start, per-type concurrency, and the achievement bypass toggle. They're documented in [`docs/VENCORD-PLUGIN.md`](docs/VENCORD-PLUGIN.md).

## When things go wrong

| Situation | What happens |
|---|---|
| 429 or 5xx | Exponential backoff and re-queue, up to 3 retries. Global and per-endpoint limits are tracked separately. |
| 404 or 403 on enroll | Quest goes on a skip list and the run continues. |
| A quest this client cannot drive | Named in the log with the task keys it offered, then skipped for the rest of the run rather than picked up again on the next cycle. |
| 5 consecutive failures on one task | That task is abandoned, the rest keep going. |
| A game quest gets nothing from Discord for 90s | Aborted with a reason, instead of sitting there until the timeout. A beat that Discord tried and failed is not silence: it resets the wait, and five in a row is what gives up. |
| 25 minutes on one task | Hard stop, next quest. |
| Auto-claim fails | A CLAIM button appears on the task card. |
| A crash | The re-entry lock is released and every patch is reverted, so you can paste again without reloading. |

Stopping is meant to leave nothing behind. The patched store methods are restored, the fake process is removed, the OAuth grant from an achievement quest is revoked, and any Discord setting it changed is put back.

## Compatibility

Vanilla Discord **Stable** is only partly usable. A Stable build changed the webpack runtime so `webpackChunkdiscord_app.push` stopped exposing the live module cache after boot, which the userscript needs ([#20](https://github.com/nyxxbit/discord-quest-completer/issues/20)). Three ways around it: run the userscript with Vencord installed and it uses Vencord's Webpack API instead, install the plugin, or use Canary or PTB where the native path still works.

In a browser or on mobile through a script-injection extension, video and activity quests work. Game and stream quests are filtered out, because they require the desktop client to exist at all.

## Architecture

`index.js` is one IIFE with no build step and no dependencies. The plugin is TypeScript at the repo root, because Vencord's `UserpluginInstaller` clones a repo straight into `src/userplugins` and only reads `index.tsx` and `native.ts` from its top level.

Stores are found by class name (`constructor.displayName`), the Dispatcher by its shape, and the API client by having a `.del` method, so nothing depends on minified paths that change every build. That handles Discord renaming things. It does not handle Discord *moving* things, which is what #43 was.

[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) is the full internal tour.

## Companion plugins

Not alternatives to this, things that run alongside it.

- [Herzchens/QuestUI](https://github.com/Herzchens/QuestUI), a Vencord plugin that adds the quest interface Orion deliberately does not: Quest Home shortcuts, status indicators, and an optional dashboard that can drive Orion. It does not complete, accept or claim anything itself, so it is useful with or without this plugin. Requested in [#48](https://github.com/nyxxbit/discord-quest-completer/issues/48).

  It reads the engine through Orion's control surface rather than poking at its internals, which means Orion stays the source of truth for what is running. Checked live before listing it, with both plugins in one build: every state change made from `/orion` showed up in an already-open QuestUI dashboard, including a per-quest pause it had not issued itself, and the card list grew as the scheduler queued more quests without a reopen. Separate project, separate maintainer, and bugs in it belong in its tracker.

## Other tools

Worth knowing about, and worth knowing their state. All three were last updated before the July 2026 change described above, and none has shipped since.

- [markterence/discord-quest-completer](https://github.com/markterence/discord-quest-completer), a native Windows app that creates dummy executables so Discord's process detection sees a game, without touching the client. Structurally the most durable approach of the four, since it doesn't read Discord internals. Play quests only, Windows only. Last updated March 2026.
- [nicola02nb/completeDiscordQuest](https://github.com/nicola02nb/completeDiscordQuest), a Vencord plugin descended from [aamiaa's original snippet](https://gist.github.com/aamiaa/204cd9d42013ded9faf646fae7f89fbb) that started this whole space. Covers everything except achievement quests. Last updated April 2026 and currently broken by the application id change, tracked in its own issue #23.
- [nvckai/Discord-Web-Auto-Quest-Extension](https://github.com/nvckai/Discord-Web-Auto-Quest-Extension), a Chrome extension, easiest to install, video quests. Last updated April 2026.

Being the one that still works today is a function of being maintained, not of being cleverer. This approach reads Discord's internals, so any Discord update can break it, and one did. The dummy-executable approach doesn't have that failure mode.

## Contributing

Bug reports, PRs and docs all welcome. [`CONTRIBUTING.md`](CONTRIBUTING.md) has the checklist and the code style, and there are issue templates for bugs and feature requests. If you're reporting a bug, the Discord build number from the very bottom of Discord's settings saves a round trip.

---

## Changelog

### v4.11.5
- **The plugin can run only quests that pay Orbs.** Asked for by [@clement-songis](https://github.com/clement-songis) in [#93](https://github.com/nyxxbit/discord-quest-completer/issues/93). The plugin has no quest picker, so farming only Orb quests meant turning auto-enroll off and accepting each one by hand in Discord's Quests page. A new setting, `Orb quests only`, off by default, leaves every quest that pays no Orbs out of the run: not accepted, not started. A quest counts when any of its rewards carries Orbs, not only the first, because Discord can list an in-game item ahead of them. Quests left out are reported in the wrap-up as "left out because they pay no Orbs", not as failures, and a run where every quest was left out no longer ends on "All available quests are completed!" with the done sound. A quest already running when you turn it on is allowed to finish, and turning it off mid-run puts the quests it left out back in on the next cycle. Checked on a live client with two otherwise identical video quests: the one paying 200 Orbs was accepted and started, the one paying none was left out with its reason and never accepted.
- **The userscript picker's ORBS filter no longer hides quests that pay Orbs.** The picker filed each quest by its first reward only, so a quest listing an in-game item before its Orbs landed under IN-GAME and disappeared when you chose ORBS. The payout line already summed every reward entry; the filter was the one place still reading the first. It now files a quest under ORBS whenever any reward pays them, which is also what the plugin's new setting counts, so both engines agree on what an Orb quest is.
- **The caution now says the quest ban reaches the account record.** Datamining reported in August that quest suspensions land alongside an Account Standing violation, the account-wide record, which the person holding the account would not necessarily see. The same entry now says plainly that nobody outside Discord has a settled figure for how long the lock lasts: the leaked client code read up to 10 days, the one detailed report here and the datamine both say 14, and the r/DiscordQuests megathread says 15.
- **A telemetry blocker does not hide quest automation.** The belief is common enough to address in the caution: running under a client mod that blocks analytics does not stop Discord seeing how quests are completed. Blocking analytics silences the product telemetry. Quest progress travels on the quest heartbeat, the functional request that has to arrive for progress to be credited at all, and that is the request carrying the executable fields.
- The plugin's settings table in the docs gains the `Play session tail` row that v4.11.4 left out, and `docs/ARCHITECTURE.md` explains why the Orb-only filter is a separate, revocable skip rather than one of the fixed reasons `questBlocker` returns.
  
----

## Disclaimer

This tool is for **educational and research purposes only**. Automating user actions violates Discord's [Terms of Service](https://discord.com/terms). The developer is not responsible for any account suspensions or bans. Use at your own risk.

---

<div align="center">

Built by [**syntt_**](https://discord.com/users/1419678867005767783)
I, GIO do not own this project, it's a fork and i've credited the main creator

If this helped you, drop a star &mdash; on my fork and the original [repo](https://github.com/nyxxbit/discord-quest-completer)

</div>
