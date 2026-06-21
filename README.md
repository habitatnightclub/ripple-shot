# Ripple Shot

A [Dalamud](https://github.com/goatcorp/Dalamud) plugin for running Habitat Nightclub's Ripple Shot: a `/random` jackpot bar game. A bartender buys shots for players, the plugin watches chat for their rolls, detects wins automatically, and prepares (but never sends) the announcement and payout breakdown.

## What it does

- **Staff-gated access** — only recognised Bartenders or Venue Owners can open the main window. Backed by a Supabase `staff` table, with two hardcoded manager keys as a fallback so nobody gets permanently locked out.
- **Solo or Shared mode** — works fully standalone with zero setup, or synced live (~1.5s) across every bartender running the plugin once Supabase is configured. Fully opt-in; nothing shared-mode-specific is required to use the plugin.
- **Buy shots, track rolls** — set a price, register players (by targeting them in-game), buy them shots, and the plugin matches their `/random` rolls in chat automatically. One purchased shot = one counted roll; extra rolls are logged but never win.
- **Pot grows on roll, not on purchase** — a shot's share of the pot is only added once that shot is actually rolled. This means a winning roll never sweeps up gil for shots a player bought but hadn't gotten to yet.
- **Automatic win detection** — a roll that's a double (`11`–`99`) or a triple (`111`–`999`, all digits identical) wins the whole pot. The pot is claimed and reset the instant a win is detected — no manual confirmation step.
- **Manual chat announcement, never auto-sent** — the plugin composes a ready-to-paste announcement line for whichever bartender's client resolved the win. A person always presses Enter. If a player wins again before the first announcement is sent, the second one queues independently rather than overwriting the first — nothing gets lost.
- **Synced dismissal** — in shared mode, once any bartender dismisses (or sends) a win's announcement, the prominent "Jackpot Won" display clears for every connected bartender, not just that one client.
- **VIP list from Supabase** — VIPs (with higher per-night caps) are resolved automatically against venue's existing VIP table.
- **Per-night archive** — every shot, roll, and payout is recorded. Locally as JSON in solo mode, or shared via Supabase so every bartender can browse every past night, not just ones run from their own computer.
- **Habitat-themed UI** — ported color palette, panel styling, and the Habitat logo in the header.

## How a night works

1. The bartender opens the round (the "Round open" checkbox — rolls only count while this is on).
2. Players are added by targeting them in-game and clicking *Add targeted character* (VIP status is detected automatically from Supabase, if configured).
3. The bartender buys shots for a player at the configured price.
4. The player rolls `/random` in chat. The plugin matches the roll to them by name + home world, counts it against a purchased shot, and adds that shot's share to the pot at that moment.
5. A winning roll claims the pot immediately and resets it to the configured seed value.
6. The plugin shows the win prominently ("Jackpot Won" next to the new "Current Jackpot"), prepares a ready-to-paste chat announcement and a trade-chunk breakdown, and the bartender sends the announcement and pays out the trades by hand.

## Solo vs Shared mode

**Solo mode** is the default — fully local, nothing leaves your machine, no setup required.

**Shared mode** activates automatically once a Project URL and anon key are present in Settings (both are baked in as defaults for this venue, so a fresh install connects with no setup). Once connected, the header shows **Server is Live** instead of **Solo Mode**, and from then on:

- Buying shots, setting the pot, adding/removing/VIP-flagging players, round-open/closed, and win announcements all sync to every bartender within ~1.5 seconds.
- Wins are detected once no matter how many bartenders' clients observe the same `/random` roll in chat — the database deduplicates by the physical roll, so nobody can accidentally consume two attempts (or pay out twice) for one roll.
- The Archive tab shows every past night from Supabase, not just nights run from one computer.

**Limitation to know about:** shared mode needs an internet connection to Supabase for every shot/roll/payout/dismiss action. If a bartender's connection drops, those actions simply fail with an error (check `/xllog`) rather than queuing for later — there's no offline mode.

## Staff authentication

The main window is gated behind a simple "Authenticate Character" check the first time it's opened each session. A character is authorized if either:

- They're listed in the Supabase `staff` table with role `bartender` or `venue owner`, **or**
- Their `name@world` matches one of the two hardcoded fallback keys in `Configuration.SupabaseManagerKeys` (`destiny skyforged@raiden`, `taniri danolnith@raiden`) — this exists so the plugin is never fully unusable if the staff table can't be reached.

Settings has its own, narrower gates:

- The Supabase connection fields and advanced VIP-mapping section are visible only to **Venue Owners** (staff-table role `venue owner`, or the two hardcoded keys).
- The "Open Settings" escape hatch shown on the auth screen *before* authenticating is restricted to just the two hardcoded manager keys specifically — by design, so that if Supabase itself is broken, only Destiny or Taniri can get into Settings to fix it. Everyone else has to wait for one of them.

## Settings reference

| Setting | Default | Notes |
|---|---|---|
| Shot price | 100,000 gil | Per shot. |
| Pot contribution | 80% | Fraction of shot price added to the pot, per roll. |
| Per-night caps | Regular 20 / VIP 30 | Reset at midnigt of the following day. |
| Pot reset value | 0 gil | What the pot resets to after a win. |
| Single-trade limit | 1,000,000 gil | Used to plan the payout's trade-chunk breakdown. |
| Win announcement | on, `/shout` | Template: `{player} ({world}) just won the {amount} Ripple Shot Jackpot with a roll of {roll}! Congratulations!` |
| Round requirement | on | Rolls only count while "Round open" is checked. |

All of the above live in *Settings*; pricing/cap/contribution changes push out to every connected bartender in shared mode.

## Supabase setup

Two tables, both outside this plugin's own schema:

**`staff`** — gates plugin access. Columns: `character_name`, `world`, `role` (`bartender` or `venue owner`). Needs an anon-readable SELECT policy under Row Level Security.

**`vip_list`** *(your existing table)* — used for per-night VIP cap detection. Matched on first + last name + world, case-insensitive (configurable column names in *Settings → Supabase* if yours differ from `character_name`/`world`). Needs the same anon SELECT policy:

```sql
alter table public.vip_list enable row level security;

create policy "anon can read vips"
  on public.vip_list for select
  to anon
  using (true);
```

**Shared mode** (optional) — run `schema_shared_state.sql` once in your Supabase SQL editor. It's idempotent, so re-running it after a plugin update (to apply schema migrations) is always safe, and it never touches `staff` or `vip_list`. It creates the shared pot/players/rolls/payouts tables and the Postgres functions the plugin calls instead of editing the database directly.

Only the public **anon** key belongs in the plugin's config — never the `service_role` key. Every table this plugin reads or writes is expected to be protected by Row Level Security; the anon key alone grants no access without a matching policy.

## Commands

- `/rippleshot` or `/rs` — opens the host window.
- `/rippleshot config` — opens Settings directly.

## Build

1. Install XIVLauncher + Dalamud and run the game once so the dev libraries exist.
2. The project uses **Dalamud.NET.Sdk** — the SDK major version must match your installed Dalamud's API level (currently **15.0.0 = API 15 / .NET 10**, target `net10.0-windows`). If your Dalamud is on a different API level, update both the SDK version in `RippleShot.csproj` and the `TargetFramework` to match.
3. `dotnet build -c Release`.
4. Add the build output folder as a dev plugin under `/xlsettings → Experimental`, or load the produced `RippleShot.json` manifest directly.

To release an update: bump `<Version>` in `RippleShot.csproj`, rebuild, re-zip the output as `RippleShot.zip`, attach it to a new GitHub release, and update `AssemblyVersion` (and `IconUrl`, if not already set) in `pluginmaster.json` on `main` — Dalamud only checks the manifest's version field, not the zip contents, so this step can't be skipped.

## Roll parsing — verify once per data centre/language

FFXIV's `/random` wording can vary. Before going live on a new data centre or client language, do a test roll and confirm the *Rolls* tab picks it up. If it doesn't, adjust the regex in *Settings → Roll parsing* — it must expose named capture groups `name` and `value` (optionally `value_max`). Default:

```
(?:Random!\s*)?(?<name>.+?)\s+rolls?\s+a\s+(?<value>\d+)(?:[.!]?\s*\((?:out of|max)\s*(?<value_max>\d+)\))?
```

Player matching is on `Name @ HomeWorld`. Same-world rolls often carry no world in chat, so the plugin falls back to matching on a unique name; cross-world visitors are matched on name + world.# Ripple Shot
