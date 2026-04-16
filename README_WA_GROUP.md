# WhatsApp Native Group-Mention Patches

This fork of [sipeed/picoclaw](https://github.com/sipeed/picoclaw) carries patches
that fix group-chat mention handling in the `whatsapp_native` channel. They are
queued for upstream review; this branch exists to make the working code available
while the upstream review process runs.

Parent commit: `51eecde0` (`Feat/support isolation #2423`), `main` as of 2026-04-15.

## What this fixes

This fork addresses two related-but-distinct bugs in the `whatsapp_native`
channel, both affecting accounts that have been migrated by Meta to the new
LID (Linked-Device ID) identity format.

### Bug 1 — `group_trigger.mention_only: true` is silently ignored

Before these patches, the bot replied to every message in every WhatsApp group,
regardless of whether it was `@`-mentioned. Four compounded defects caused this:

1. **`WhatsAppConfig` was missing the `GroupTrigger` field.** Every other
   channel config struct had it. Without the field, `encoding/json` had nowhere
   to place the `group_trigger` key during unmarshal and silently dropped it.

2. **`NewWhatsAppNativeChannel` didn't forward `WithGroupTrigger`** (or
   `WithReasoningChannelID`) to `channels.NewBaseChannel`. Even with the config
   field present, the trigger never reached the base channel implementation.

3. **`handleIncoming` had no mention detection.** It never read
   `ExtendedTextMessage.ContextInfo.MentionedJID` (WhatsApp's protocol-level
   mention metadata).

4. **`handleIncoming` never gated through `ShouldRespondInGroup`.** Compare with
   Discord, Line, IRC, QQ — all call `ShouldRespondInGroup(isMentioned, content)`
   before forwarding to the agent bus. The native WhatsApp channel didn't.

The patches handle the LID/PN JID duality: `Store.ID` (phone-number form) and
`Store.GetLID()` (LID form) are both checked against the mention list.

### Bug 2 — `allow_from` silently drops device-index-drifting LIDs

On LID-migrated accounts, sender JIDs arrive as `<user>:<N>@lid` where `:N` is
a device/agent index that can shift during normal WhatsApp session
housekeeping. The existing `identity.MatchAllowed` does exact-string comparison
against the stored allow-list entry, which silently breaks the moment `N`
changes.

The added `lidBaseParts` helper and a short new branch at the top of
`MatchAllowed` handle this: when an allow-list entry is a bare LID
(`<user>@lid`, no colon), the sender's device suffix is stripped before the
base part is compared. Allow-list entries that include an explicit device
suffix continue to use exact-match — fully backwards compatible for existing
configs.

Unit tests cover the new behavior plus backwards-compat scenarios.

### Bug 3 — zombie socket: whatsmeow claims connected while inbound events silently stop

In production we observed a state where the whatsmeow client's
`IsConnected()` method returned `true` but WhatsApp's servers had
silently stopped delivering inbound events. The existing reconnect
supervisor bounced off `"websocket is already connected"` every 5
minutes indefinitely; group messages were dropped for 30+ minutes
until a manual launcher restart.

Two complementary changes address this:

1. **`reconnectWithBackoff` now escapes the "already connected" loop.**
   When `client.Connect()` returns an error containing
   `"already connected"`, the supervisor treats it as a signal that
   whatsmeow's internal state is stuck on a half-open socket. It calls
   `Disconnect()` to clear the stale handle, then retries `Connect()`
   immediately (no backoff).

2. **A new `healthWatchdog` goroutine detects silent zombies.** It
   tracks `lastEventAt` (updated in `eventHandler` on every whatsmeow
   event — messages, presence, appstate, keepalive). Every 2 minutes
   it checks: if `IsConnected()` is true but no events have arrived
   for 10 minutes, it calls `Disconnect()` so the normal
   `*events.Disconnected` → reconnect flow takes over.

10 minutes is a balance for family-chat workloads — quiet periods are
common but rarely exceed 10 min of total silence across all event
types. False-positive reconnects are cheap (~8 s, session preserved
via SQLite) and strictly better than silent delivery failure.

## Building

```bash
go build -tags goolm,stdjson,whatsapp_native ./cmd/picoclaw
```

Or use the upstream Makefile target:

```bash
make build GO_BUILD_TAGS=goolm,stdjson,whatsapp_native
```

Requires Go 1.21+. See the upstream [README](README.md) for full prerequisites.

## Status

- [x] Upstream issue filed for `allow_from` LID handling — [sipeed/picoclaw#2540](https://github.com/sipeed/picoclaw/issues/2540)
- [x] Upstream issue filed for `WhatsAppConfig.GroupTrigger` + mention handling — [sipeed/picoclaw#2541](https://github.com/sipeed/picoclaw/issues/2541)
- [ ] Upstream issue to file for zombie-socket reconnect supervisor (Bug 3)
- [ ] Upstream PR opened against `sipeed/picoclaw:main`

## License

MIT — inherited from upstream `sipeed/picoclaw`. See [LICENSE](LICENSE).
