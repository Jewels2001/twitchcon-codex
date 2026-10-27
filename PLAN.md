# Twitch Chat Overlay Bot — Project Plan

**Status:** Draft for review. This document describes proposals; implementation does not begin until the user approves the plan and gives the go-ahead to code.

**Purpose:** Build a useful, creative Twitch chatbot for small streamers. Chat commands trigger restrained visual effects in an OBS browser source. The first art pack is Halloween, built on a theme system that can also support a neutral theme and later seasonal packs.

## Working agreements

- Keep this file as the project’s living plan and decision log. Record each approved change with its date, reason, and impact. Mark suggestions as **Proposed** until approved; record rejected or deferred options briefly so they are not accidentally reintroduced.
- Before coding, present material design choices and risks here for review. Proceed only after the user approves them and says to begin implementation.
- Do not put OAuth tokens, client secrets, or private channel details in this file.
- This is a documentation-only planning pass. No application code has been changed.

## Starting point

- `bot.js` is a Node.js ES module. It loads settings from `.env`, validates a Twitch token, opens an EventSub WebSocket, subscribes to `channel.chat.message`, and can send a reply through Twitch’s Send Chat Message API.
- The current example behavior replies to `HeyGuys` with `VoHiYo`. There is no command parser, HTTP server, overlay page, browser-to-bot event channel, theme configuration, or custom-art support yet.
- The current WebSocket handler covers the welcome and notification messages. It does not yet implement EventSub reconnect/recovery behavior, subscription revocation handling, or duplicate-message protection.
- `.env` is present. It was not opened during this planning pass. Before sharing or publishing the project, confirm that secrets are excluded from version control and logs. No usable Git metadata was detected in this workspace, so this Markdown file is the initial durable project record.

## Product goals

1. Let a streamer add a transparent browser source to OBS and see small, timely reactions to chat.
2. Make the default experience polished and quiet: compact placement, brief animations, no sound by default, and no obstruction of gameplay, captions, alerts, or face cam.
3. Give streamers a small set of safe viewer commands and separate streamer/moderator controls for persistent settings.
4. Make seasonal art replaceable. Halloween is the first complete pack; a neutral/general pack should always be available as a fallback. Winter and spring packs can follow without changing command or overlay plumbing.
5. Allow a streamer to replace default art with their own assets without requiring them to write application code.
6. Keep setup understandable for a small streamer and keep credentials off the browser page.

## Proposed first release

### Include

- One channel at a time, running locally on the streamer’s computer.
- The existing Twitch chat connection, extended with a strict command allowlist and the reliability work below.
- A local Node.js service that serves the overlay page and relays approved effects to it.
- An OBS Browser Source that loads a loopback URL such as `http://127.0.0.1:<port>/overlay`. OBS Browser Source supports loading a URL and setting the viewport dimensions; the overlay should use a transparent page background.
- One neutral theme and one Halloween theme. The Halloween art direction should be festive and friendly rather than jump-scare-heavy.
- A few short, restrained effects, for example a small ghost drift, bat pass, or pumpkin bob in a screen corner.
- Local configuration for theme selection and basic placement/scale settings. Streamer/moderator chat commands may offer the same small set of controls if approved.
- A documented folder and manifest convention for replacing included art with a streamer’s own local art.

### Keep out of the first release

- A hosted multi-channel service, public account dashboard, or one-click installer. These may be valuable later, but they introduce account onboarding, deployment, and credential-storage decisions that should be made after the single-channel prototype is understood.
- Viewer-uploaded files, remote image URLs, arbitrary CSS/HTML from chat, or user-authored scripts.
- An economy, points system, bits/subscription triggers, sound effects, or broad moderation features.
- Automatic calendar-based theme switching. Let the streamer choose the theme; seasons and holidays vary by preference and location.

## Proposed user experience

1. The streamer starts the local bot/service and sees a clear status for Twitch connection and the local overlay URL.
2. The streamer adds that URL as an OBS Browser Source and sets its size to match the stream canvas (for example, 1920×1080). The page itself remains transparent.
3. A viewer uses one of a few documented, exact commands. The bot validates the command, caller permissions, and cooldowns, then sends a small typed effect event to the overlay. It does not echo every trigger into chat.
4. The effect appears briefly, then fades away. The overlay caps simultaneous effects and queues or drops excess events according to a documented rule.
5. A broadcaster or moderator may change the active theme, hide/show the overlay, or reset it. Viewer commands cannot alter persistent configuration.
6. If the OBS source reloads, it receives the current theme/settings snapshot. Expired effects are not replayed.

## Proposed architecture

```text
Twitch chat
    │ EventSub WebSocket
    ▼
Node service ── command parser / permissions / cooldowns
    │                         │
    │ current config/state    └── validated effect event
    ▼                                  ▼
HTTP overlay page  ◀── local WebSocket or SSE relay ── OBS Browser Source
```

### Responsibilities

- **Twitch adapter:** token validation, EventSub subscription, chat-message normalization, connection status, reconnection, and optional chat replies.
- **Command layer:** parse exact command names and arguments; reject unknown or malformed input; apply role rules, per-user/channel cooldowns, and duplicate detection before taking action.
- **Overlay bridge:** serve static files only on loopback and broadcast a small JSON message format. Never expose Twitch tokens to the browser page.
- **Overlay client:** render the current theme and only the received, allowlisted effects. On initial connection or refresh, request a current-state snapshot.
- **Configuration:** keep persistent theme and visual preferences separate from temporary effects. Start with a local config file and theme manifests; avoid a dashboard in the first release.
- **Theme packs:** define theme ID, palette, asset mapping, effect names, and animation settings independently from Twitch command handling.

### State and event distinction

- **Persistent state:** selected theme, enabled/hidden state, placement, scale, and other approved preferences. Persist locally so a restart or OBS reload has a predictable result.
- **Transient event:** a particular ghost/bat/pumpkin effect with an event ID and expiry. Do not persist it as a replayable action.
- **Recovery:** Twitch EventSub notifications may be delivered more than once, so ignore duplicate message IDs. When the Twitch socket is lost, reconnect and restore subscriptions. When OBS reconnects, send a fresh state snapshot rather than replaying the event history.

## Proposed command and permission shape

The names and cooldown values below are examples for review, not approved behavior.

| Command/action | Proposed callers | Result |
| --- | --- | --- |
| `!ghost`, `!bat`, or `!pumpkin` | Any chatter, subject to cooldown | Trigger one small effect from the active theme |
| `!theme <name>` | Broadcaster and moderators only | Change the persistent theme to an installed theme |
| `!overlay on\|off` | Broadcaster and moderators only | Show or hide the browser overlay |
| `!overlay reset` | Broadcaster and moderators only | Clear active effects and restore the default state |
| `!help` | Any chatter | Optional concise command list; decide whether it replies in chat or is documented outside chat |

Suggested safeguards: exact command allowlist; configurable per-user and channel-wide cooldowns; bounded effect queue; no raw chat text rendered as HTML; no viewer control of theme, placement, or persistent settings; little or no routine bot chatter. The current `HeyGuys` example response should be replaced by the approved command behavior once implementation is authorized.

## Themes, art, and visual behavior

- Treat a theme as a replaceable art/configuration pack, not a separate overlay implementation. Use stable effect IDs so a command can map to a Halloween ghost now and a different seasonal character later.
- Ship Halloween and neutral/general themes together. Future themes may include winter and spring; theme selection remains manual in the first version.
- Use a consistent safe-area layout with restrained scale, opacity, and animation duration. Effects should not cover the center of the stream or stack without a limit.
- Provide an overlay hide/reset control and a reduced-motion or static mode. Avoid rapid flashing and audio by default.
- Proposed custom-asset support for the first version: local transparent PNG/WebP files referenced by a theme manifest, with recommended size/placement guidance. Consider animated formats only after checking their performance and OBS behavior. Do not accept SVG, remote URLs, or executable content in the first version.
- Missing or invalid custom art should fall back to the theme’s included default asset rather than breaking the overlay.

## Security, privacy, and reliability requirements

- Keep access tokens in local secret configuration, never in browser JavaScript, an OBS URL, source control, screenshots, or routine logs. Provide a sanitized `.env.example` if setup still uses environment variables.
- Request only the Twitch scopes needed for the approved operations. The exact token model and scopes must be verified against the selected Twitch chatbot model before implementation; Twitch distinguishes installed and cloud chatbots, and the sample code’s authorization approach should not be assumed to be the final distribution model.
- Bind the local web server to loopback (`127.0.0.1`) by default. Do not make the control/configuration endpoints reachable from the LAN or internet in the prototype.
- Treat chat as untrusted input. Parse a fixed command grammar; do not inject raw message text or usernames as HTML. Serve only known static assets and validate theme configuration.
- Handle EventSub welcome, notification, keepalive, reconnect, revocation, and unexpected disconnect states. Deduplicate message IDs because EventSub can redeliver notifications.
- If the overlay bridge disconnects, retry with backoff and restore persistent state. If Twitch disconnects, display a clear local status and avoid claiming that commands are active.
- Limit logs to operational status and concise errors; avoid retaining full chat history unless a future feature requires it and is explicitly approved.

## Implementation phases (proposed)

1. **Approve product decisions.** Confirm local single-channel prototype, command permissions, custom-asset format, theme controls, and whether bot chat replies are needed. Update this file with the decisions.
2. **Harden the Twitch foundation.** Make config validation explicit; verify the chosen Twitch auth model/scopes; handle EventSub reconnection/revocation and duplicate delivery; remove the sample-only reply behavior as appropriate.
3. **Define command and state contracts.** Specify command grammar, role checks, tunable cooldown defaults, theme IDs, effect IDs, expiration, and configuration schema.
4. **Build the local overlay bridge.** Add loopback HTTP serving, overlay event transport, initial state sync, connection status, and clean shutdown behavior.
5. **Create the overlay and theme packs.** Implement the transparent, low-distraction layout; build the generic theme and Halloween defaults; document the custom-art folder and manifest.
6. **Prepare OBS setup and streamer guidance.** Document startup, adding the Browser Source URL, canvas sizing, theme choice, local art replacement, troubleshooting, and safe shutdown.
7. **Review the prototype against acceptance criteria.** Only after coding is approved, verify Twitch connection and recovery, command permission/cooldown behavior, OBS rendering, refresh/state recovery, custom-art fallback, and absence of secrets from the browser/logs. No implementation checks or test files were run in this planning pass.

## Proposed acceptance criteria

- A streamer can start the local service, connect the approved Twitch account/channel, and load the overlay in OBS using a documented browser-source URL.
- Valid viewer commands produce only the approved short visual effects; invalid commands and commands over the limit do not create effects.
- Only the approved privileged roles can change persistent settings.
- The default Halloween theme is usable, the neutral theme is selectable, and a missing custom image falls back cleanly.
- The source has a transparent background, stays inside the configured safe area, and offers a hide/reset and reduced-motion option.
- Refreshing the OBS browser source restores current settings without replaying old effects.
- EventSub reconnection and duplicate notifications do not cause a flood of effects or leave the bot silently disconnected.
- Twitch credentials are never sent to the browser source or written to routine logs.

## Decisions awaiting approval

| Decision | Recommendation | Status |
| --- | --- | --- |
| First delivery | Local, single-channel prototype; revisit packaged/hosted onboarding later | Proposed |
| Viewer controls | Viewers may trigger a few allowlisted effects; broadcaster/moderators control theme and persistent settings | Proposed |
| Default themes | Ship Halloween plus a neutral/general fallback | Proposed |
| Theme switching | Manual selection; no automatic date/holiday switch in v1 | Proposed |
| Custom art | Local transparent PNG/WebP referenced by a manifest; no upload UI or remote images in v1 | Proposed |
| Overlay transport | Local HTTP page plus a local WebSocket/SSE event channel | Proposed |
| Chat responses | No response for every effect; decide whether a quiet `!help` response is needed | Proposed |
| Cooldowns and queue limits | Make configurable; choose concrete defaults before coding | Proposed |

## Risks and adjustments to consider

- **Small-streamer setup friction:** requiring Node.js and manual token setup is acceptable for a prototype but may be too technical for the intended audience. A later packaged app or OAuth setup flow may be needed before calling it generally ready for streamers.
- **Twitch auth model:** the current example uses a user token for EventSub and chat send. Twitch documents different authorization requirements for installed versus cloud chatbots. Decide which model applies before designing a multi-user setup; request only necessary scopes.
- **Overlay clutter:** even charming effects become noisy if every chatter can trigger them repeatedly. Start with few effects, sensible tunable cooldowns, a queue cap, short durations, and a streamer kill switch.
- **Theme consistency:** custom art can have very different dimensions and animation behavior. Define recommended canvas size, transparent margins, max file size, and fallback behavior before opening up customization.
- **OBS/browser lifecycle:** the page may reload or be unloaded when hidden. State snapshots and reconnect behavior need to be designed into the bridge so the overlay recovers cleanly.
- **Project history:** the workspace has no usable Git metadata. Before multiple code milestones or external sharing, establish version control and ensure `.env` is excluded; keep this plan as the human-readable decision record as well.

## References

- Twitch [Example Chatbot guide](https://dev.twitch.tv/docs/chat/chatbot-guide/) — current starter example and EventSub WebSocket pattern.
- Twitch [Authenticating and Setting up EventSub](https://dev.twitch.tv/docs/chat/authenticating/) — installed/cloud chatbot models and chat authorization.
- Twitch [Sending and Receiving Chat Messages](https://dev.twitch.tv/docs/chat/send-receive-messages/) — chat message and EventSub payload behavior.
- Twitch [Handling WebSocket Events](https://dev.twitch.tv/docs/eventsub/handling-websocket-events/) — welcome, keepalive, reconnect, and lost-connection handling.
- Twitch [Access Token Scopes](https://dev.twitch.tv/docs/authentication/scopes/) — scope reference; confirm current endpoint requirements when the auth model is selected.
- OBS [Browser Source](https://obsproject.com/kb/browser-source) — URL, viewport size, transparent background, and browser-source behavior.

---

## Decision log

No product decisions have been approved yet. Add dated entries here as they are approved or changed.

