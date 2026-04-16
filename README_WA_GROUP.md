# WhatsApp Native Group-Mention Patches

This fork of [sipeed/picoclaw](https://github.com/sipeed/picoclaw) carries patches
that fix group-chat mention handling in the `whatsapp_native` channel. They are
queued for upstream review; this branch exists to make the working code available
while the upstream review process runs.

Parent commit: `51eecde0` (`Feat/support isolation #2423`), `main` as of 2026-04-15.

## What this fixes

Before these patches, `channels.whatsapp.group_trigger.mention_only: true` in
`config.json` was silently ignored on the `whatsapp_native` channel. The bot
replied to every message in every WhatsApp group, regardless of whether it was
`@`-mentioned. Four compounded defects caused this:

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

The patches handle the LID/PN JID duality that's already rolled out to many
WhatsApp accounts: `Store.ID` (phone-number form) and `Store.GetLID()` (LID
form) are both checked against the mention list.

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

- [ ] Upstream issue filed for `WhatsAppConfig.GroupTrigger` field + mention handling
- [ ] Upstream issue filed for `allow_from` LID handling (separate bug, same channel)
- [ ] Upstream PR opened against `sipeed/picoclaw:main`

## License

MIT — inherited from upstream `sipeed/picoclaw`. See [LICENSE](LICENSE).
