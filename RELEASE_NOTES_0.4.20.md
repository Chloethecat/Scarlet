# Scarlet 0.4.20

A feature release focused on **spotting the right people faster** and **reacting to new instances faster**, plus a readability pass on the player-list colours.

> Version header is provisional (0.4.20) — rename if you'd rather cut this as something else.

## Trust ranks

The player list now has a **Rank** column showing each player's VRChat trust rank — Visitor, New User, User, Known, Trusted — plus **Nuisance** for troll-flagged accounts. It's derived from the account's own tags, so it costs no extra API calls. VRChat only returns a complete tag set for some users, so treat it as a strong hint rather than a guarantee; in practice it's right for the large majority of joins.

Two rank-based alerts come with it:

- **Nuisance (troll-flagged):** on by default. It's flagged in the advisory column and sorted to the top of the list, the bundled **siren-chirp** alert plays, and the join callout speaks "Nuisance rank." The sound is one of your `BL_SFX_*` effects, rendered to WAV and shipped inside the build (extracted to a scratch temp file for playback, never the data folder) so it plays on every route including Discord voice. Turn the pieces off with `advisory_flag_nuisance_rank` (row/advisory) and `tts_announce_nuisance_rank` (alarm + callout).
- **Visitor:** opt-in, off by default. An optional row advisory (`advisory_flag_visitor_rank`) and an optional spoken callout (`tts_announce_visitor_rank`) for owners who want to watch for brand-new accounts. Visitors are usually fine, so this is quiet unless you ask for it.

## Faster follow-into-new-instance (~90s → ~30s)

When *Launch on Instance Create* is enabled, the bot used to take roughly a minute and a half to notice a freshly created instance, because it waited for the group **audit log** — which is limited both by the poll interval and by VRChat's own ingest lag before an entry even appears.

Scarlet now also watches the group's **live open-instance list** on a short cadence and cold-boots the client the moment a new instance shows up. It's built to stay well clear of anything VRChat would flag:

- one lightweight call per cycle, at a **tunable interval** (`instance_follow_fast_poll_seconds`, default **30s**, hard **15s** floor),
- only while *Launch on Instance Create* is on,
- backs off a full minute on any error or rate-limit response,
- deduplicated with the existing audit path, so an instance is never launched into twice.

Net effect: detection drops from ~90s to ~30s (tunable) without hammering the API.

## Readability: advisory colours

The player-list advisory colours were raw, over-saturated primaries that were harsh on a dark theme and rough to read over a Discord screenshare. They've been retuned to a calmer, higher-contrast set that reads the same at a glance but is much easier on the eyes — and **Nuisance** and **Visitor** ranks now each have their own distinct colour. This is UI-only; Discord embed colours are unchanged.

## Headless fix

When Scarlet finds more than one data folder and there's no interactive console (a headless service), the "which folder?" prompt no longer opens a blocking read on stdin — it logs the candidates and deterministically loads the first, so a headless bot can't stall at startup.

## Bans no longer logged as standalone kicks

Old bug, going back to the legacy `0.4.12`-era builds. When you ban someone who's currently in an instance, VRChat auto-issues an instance-kick alongside the ban. Scarlet is meant to recognize that kick as part of the ban and bundle it under the ban's thread rather than posting it as a separate kick.

The detection compared the new kick against the target's "most recent prior action" — but an inverted date comparison made that resolve to the target's *oldest* action instead of the newest. So for any user with prior moderation history, the co-occurring ban was missed and its auto-kick got logged as a normal kick. Fixed the comparison (it now correctly picks the newest prior action, which also corrects the "Most recent" line in the moderation embed).

Note for testing: there's a separate, intermittent timing factor — the ban's Discord thread is created a beat after the ban is posted, so if the auto-kick is processed in the very same audit poll it can occasionally still slip through before the thread exists. If you still see the odd ban-as-kick after this fix, that's the cause and I'll harden the timing.


## Outstanding-moderation re-ping fixed

The reminder that re-pings a moderator when they log an action without a reason was completely non-functional: it was never called from the run loop, and its "is this outstanding?" test was inverted. Both are fixed — it now runs, and correctly flags only actions with **no tags and no description** (skipping redacted entries and the auto-kick that gets bundled under a ban).

Per community request, **a message posted in the moderation thread now counts as a valid log** — the first message a moderator sends in the thread is captured as that action's reason, so the re-ping clears without needing tags. (Heads-up: this uses an in-memory thread→entry map, so it captures messages in threads created since the bot last started; a message in a pre-restart thread won't be captured. And the whole feature only posts if you've configured an "Outstanding Moderation" channel and enabled the relevant *Ping on outstanding…* toggles.)


## Report profile pictures / user icons to VRChat

Community idea (KyootFox): reporting in-game profile / sticker / emoji content. Sticker, emoji and print reporting already existed — Scarlet surfaces a pre-filled VRChat T&S report link on each one's in-instance spawn embed. The gap was **profile pictures and user icons**, which had a report-URL builder in the code but were never shown anywhere.

Moderation embeds now include **Report profile picture** and **Report user icon** links, pre-filled with the target user and your configured report email (Settings → "VRChat Help Desk report email"). Scarlet only assembles the report; a human clicks through and files it with VRChat T&S — no automated reporting under the bot account.


## Moderation/report data no longer drops out when the API is flaky

The bug behind "sometimes the report data doesn't appear, and it's inconsistent." The moderation embed assumed VRChat always returns the full target user, and dereferenced it in several places (name, image, join date, pronouns, status). When the API didn't — rate limits, a private profile, or a transient failure, the same flakiness behind the avatar/trust-rank gaps — the embed threw and the entire moderation log (with its report links) silently failed to post. Which user it hit was luck of the draw, so it looked random across people.

Now every user field is null-guarded and falls back to the audit entry's own IDs. The log and the pre-filled report links always post; any field VRChat didn't return is just omitted (with a short note), instead of taking the whole message down with it.
