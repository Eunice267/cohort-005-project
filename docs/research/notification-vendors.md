# Which notification vendors fit Cadence?

Research for issue #2 (parent map #1). Researched 2026-11-14 against vendor pricing pages, official docs and the npm registry. Prices and limits change, so re-check the linked pages before you decide.

**Scope:** Cadence is a React Router 8 / React 19 SSR app on a Node server. It already sends transactional email through Postmark (`app/lib/mailer.server.ts`, with a `.outbox/` fallback in dev). We want an in-app feed/inbox and browser web push, and email as a possible channel. **This doc does not pick a vendor.** That happens in the vendor grilling ticket.

## Comparison

| | Knock | Novu (Cloud) | Courier | MagicBell | SuprSend |
|---|---|---|---|---|---|
| **Free tier** | Developer plan, $0. 10,000 messages/mo. Unlimited workflows, channels and team members. Knock branding, 30-day logs. Not hard-capped. [1] | Free plan, $0. 10,000 workflow runs/mo. **Max 5 workflows**, 3 team members, Dev and Prod environments, 24h activity retention. [2] | Free up to 10,000 sends/mo. Test environments count toward the cap. Past 10k you move to pay-as-you-go at $0.005 per send, charged from the first send. [3] | Builder plan, $0. **1,000 deliveries/mo**, 1 project. [4] | Free plan, $0. 10,000 notifications/mo, unlimited channels. Feed carries SuprSend branding, 30-day logs. [5] |
| **First paid step** | Starter $250/mo [1] | Pro from $30/mo [2] | PAYG $0.005/send [3] | Startup $249/mo [4] | Essentials $100/mo [5] |
| **React inbox/feed** | `@knocklabs/react`: `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton`, toasts, preference center [6] | `@novu/react`: `<Inbox>` with `Bell`, `Notifications` and `Preferences` sub-components [7][8] | `@trycourier/courier-react`: `CourierInbox`, `CourierInboxPopupMenu`, `CourierToast`, `CourierPreferences` [9] | `@magicbell/react`: `Inbox`, `FloatingInbox`, `WebPushButton`, `UserPreferences`, toasts [10] | `@suprsend/react` with drop-in inbox and headless mode. Old `@suprsend/react-inbox` has reached end of support. [11] |
| **Headless option** | Yes, React hooks [6] | Yes, `@novu/js` client (notifications and preferences APIs) [12] | Yes, `useCourier()` hook [9] | Not documented for React. `magicbell-js` client exists. [10][13] | Yes, headless mode [11] |
| **React 19 peer dep (npm)** | `^17 \|\| ^18 \|\| ^19` (v0.15.1). `next` and `@tanstack/react-router` are optional peers. [14] | `^18 \|\| ^19` (v3.19.2) [14] | `>=18` (v9.3.5) [14] | `@magicbell/react` 1.14.0 declares no React peer [14] | `>=16.8` (v1.5.0) [14] |
| **SSR notes** | No SSR guidance in the React overview. Treat as client-only. [6] | None documented. The quickstart uses React Router's `useNavigate` through the `routerPush` prop. [7] | "Inbox and Toast support Next.js but render only on the client". Needs `'use client'` in Next. [9] | Not documented | Not documented |
| **Browser web push** | **No first-party web push.** Goes through **FCM**: Firebase project, `firebase-messaging-sw.js`, Firebase VAPID key, then the token is saved with `knock.user.setChannelData`. [15][16] | Via providers: FCM, OneSignal, Pushpad, Pusher Beams, Push Webhook (no native VAPID channel documented) [17] | Via providers: FCM (covers web), Pusher Beams (`web` mode), OneSignal, and others [18] | **Native Web Push channel.** You supply your own VAPID key pair. The service worker `importScripts` MagicBell's `sw.js`. `WebPushButton` handles subscribing. [10][19] | **Native web push.** Root `serviceworker.js` imports the SuprSend SW script. `vapidKey` goes on `SuprSendProvider`. `webpush.registerPush()` requests permission, registers and stores the token. [20] |
| **Email / Postmark** | Postmark is a supported provider (API key, from address, `MessageStream` override, sandbox toggle) [21] | Postmark is a supported provider, plus a demo provider for testing [22] | Postmark is a supported provider (Server API token, `MessageStream` override) [23] | **No Postmark integration listed.** Supports SES, Mailgun, SendGrid, SMTP and Ping Email. Postmark's SMTP relay might work but is unverified. [13] | Postmark is a supported vendor (Server Token, plus a webhook for delivery status) [24] |
| **Identifying users from SSR** | User id plus public key. In **enhanced security mode** the server signs an **RS256 JWT** (`sub` = user id) with `signUserToken` from `@knocklabs/node` and passes it as the `userToken` prop. Recommended for production. [25] | `subscriberId` plus **`subscriberHash` = HMAC-SHA256(subscriberId, NOVU_SECRET_KEY)** computed on the server. **Mandatory in production.** [26] | The backend mints a **JWT** with the Courier API key (scopes such as `inbox:read:messages`), then the client calls `auth.signIn({ userId, jwt })` [9] | The backend signs a **HS256 User JWT** with the project Secret Key (`user_external_id` or `user_email`, plus `api_key`) and passes it as the `token` prop on `ContextProvider` [27] | The backend signs an **ES256 JWT** (`entity_id` = user id) with the signing key, then the client calls `identify(id, token)`. Enhanced security is advised for production workspaces. [28] |
| **Per-recipient preferences** | Yes. PreferenceSet by category, channel type, channel and workflow. Hosted or custom preference center, with React components. Per-tenant preferences are not on Developer. [1][29][6] | Yes. Global and per-workflow, per channel, through the server and client SDKs [30] | Yes. Subscription topics (opted in, opted out or required) with per-topic channel routing, a hosted page and `CourierPreferences` [31][9] | Yes. `UserPreferences` component toggles channels per category. [10] | Yes. A preference centre exists (`$hosted_preference_url` in templates). Not deeply verified. [24] |
| **Node trigger SDK** | `@knocklabs/node` (v1.41.1, active) [14] | `@novu/api` (v3.19.1). `@novu/node` is **deprecated**. [14] | `@trycourier/courier` (v10.0.0) [14] | `magicbell-js` (browser and Node client) [13][14] | `@suprsend/node-sdk` (v1.15.1) [14] |
| **Local / test mode** | Separate Development and Production environments. Enhanced security can stay off in dev. [25] | Dev and Prod environments. Local Studio (`npx novu dev`) still needs a Cloud key and a tunnel. **Self-hostable** Community Edition (Docker), with some cloud-only features missing. [2][32][33] | Test environment with its own keys (counts toward quota) [3][23] | Push test toolkit. No separate dev environment confirmed on the free plan (1 project). [4][13] | Not verified (the docs pages fetched did not cover staging workspaces) |

### Also considered

- **Pingram (formerly NotificationAPI).** `notificationapi.com` now redirects to `pingram.io`. `@notificationapi/react` declares a React 19 peer. The pricing page now leads with email, SMS and voice (free: 3,000 emails/mo) and gives no in-app or web-push limits, and the web-push docs URL returned 404. The product looks like it is mid-pivot, so it is not shortlisted. [34][14]

## Shortlist and trade-offs

These are listed in no order of preference.

1. **Knock**
   - *For:* A generous dev tier (10k messages, unlimited workflows). The most mature React feed and preferences components, with React 19 declared. First-class Postmark provider, so the existing sender can stay. RS256 token signing comes with a Node helper. Separate dev and prod environments.
   - *Against:* No native web push, so Cadence would add Firebase (FCM SDK, `firebase-messaging-sw.js`, a Firebase project). The paid step is steep ($250/mo) once past 10k.

2. **Novu**
   - *For:* Open source and self-hostable, which reduces lock-in. The cheapest paid step ($30/mo). Postmark is supported. The `<Inbox>` already speaks React Router through `routerPush`. Simple HMAC user hash.
   - *Against:* The free tier caps you at **5 workflows** and keeps 24h of activity. Web push only goes through third parties (FCM, OneSignal or Pushpad). The Node SDK moved from `@novu/node` to `@novu/api`, so older examples are stale.

3. **SuprSend**
   - *For:* **Native web push** (VAPID plus a service worker they provide), a drop-in and headless React inbox, Postmark as a vendor, and 10k/mo free.
   - *Against:* Smaller vendor. The inbox package was recently replaced (`react-inbox` reached end of support), which is a churn signal. The free feed is branded. Staging and test story not verified.

4. **MagicBell** (worth a look mainly for push)
   - *For:* The strongest **native web push** story (`WebPushButton`, hosted `sw.js`, your own VAPID keys). Simple HS256 user JWT.
   - *Against:* Only **1,000 deliveries/mo** free. **No Postmark integration** (email would have to stay on Cadence's own mailer, or go through generic SMTP). The jump to paid is $249/mo.

5. **Courier**
   - *For:* Pay-as-you-go above 10k sends with no big plan jump. A full React kit with a hook. Postmark is supported. Rich topic-based preferences.
   - *Against:* Web push only through FCM or Pusher Beams. Test-environment sends count toward the free cap. Once past 10k, every send is billed, including the first 10k.

### Cross-cutting notes for the decision

- **Web push splits the field.** MagicBell and SuprSend handle VAPID and the service worker natively. Knock, Novu and Courier need FCM or another push provider. Every option needs a service worker served from Cadence's origin over HTTPS, so the React Router app has to serve a root-scoped SW file.
- **SSR.** None of the inbox SDKs document server rendering. Plan to render the bell and feed client-only (for example a `ClientOnly` wrapper or a lazy import), with the signed user token or HMAC created in a loader on the server.
- **Postmark can stay either way.** Cadence can keep sending refund email through `mailer.server.ts` and use the vendor only for in-app and push. Or, with Knock, Novu, Courier or SuprSend, the vendor can route email through Cadence's own Postmark server token.

## Sources

1. Knock pricing: https://knock.app/pricing
2. Novu pricing: https://novu.co/pricing
3. Courier pricing: https://www.courier.com/pricing
4. MagicBell pricing: https://www.magicbell.com/pricing
5. SuprSend pricing: https://www.suprsend.com/pricing
6. Knock React overview: https://docs.knock.app/in-app-ui/react/overview
7. Novu React quickstart: https://docs.novu.co/platform/quickstart/react
8. Novu React Inbox examples: https://docs.novu.co/.mintlify/skills/inbox-integration/references/react-inbox-examples.md
9. Courier React SDK: https://www.courier.com/docs/sdk-libraries/courier-react-web
10. MagicBell React library: https://www.magicbell.com/docs/libraries/magicbell-react
11. SuprSend React inbox: https://docs.suprsend.com/docs/inbox-react
12. Novu headless (`@novu/js`): https://docs.novu.co/.mintlify/skills/inbox-integration/references/headless-inbox-examples.md
13. MagicBell docs index (channels, email providers, libraries): https://www.magicbell.com/docs
14. npm registry manifests, fetched 2026-11-14: `https://registry.npmjs.org/<pkg>/latest` for `@knocklabs/react`, `@knocklabs/node`, `@novu/react`, `@novu/api`, `@novu/node`, `@trycourier/courier-react`, `@trycourier/courier`, `@magicbell/react`, `magicbell-js`, `@suprsend/react`, `@suprsend/react-inbox`, `@suprsend/node-sdk`, `@notificationapi/react`
15. Knock push providers: https://docs.knock.app/integrations/push/overview
16. Knock web push with FCM: https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
17. Novu push providers: https://docs.novu.co/platform/integrations/push
18. Courier push providers: https://www.courier.com/docs/integrations/push/overview, https://www.courier.com/docs/external-integrations/push/pusher-beams
19. MagicBell web push channel: https://www.magicbell.com/docs/channels/web_push
20. SuprSend web push: https://docs.suprsend.com/docs/webpush
21. Knock and Postmark: https://docs.knock.app/integrations/email/postmark
22. Novu email providers: https://docs.novu.co/platform/integrations/email
23. Courier and Postmark: https://www.courier.com/docs/external-integrations/email/postmark
24. SuprSend and Postmark: https://docs.suprsend.com/docs/postmark
25. Knock client authentication: https://docs.knock.app/in-app-ui/security-and-authentication
26. Novu Inbox HMAC: https://docs.novu.co/.mintlify/skills/inbox-integration/references/security.md
27. MagicBell user JWT: https://www.magicbell.com/docs/api/authentication/user
28. SuprSend client authentication: https://docs.suprsend.com/docs/client-authentication
29. Knock preferences: https://docs.knock.app/preferences/overview
30. Novu subscriber preferences: https://docs.novu.co/.mintlify/skills/manage-preferences/references/subscriber-preferences-examples.md
31. Courier preferences: https://www.courier.com/docs/platform/preferences/preferences-overview
32. Novu Local Studio: https://docs.novu.co/.mintlify/skills/framework-integration/references/studio-and-cli.md
33. Novu self-hosting: https://docs.novu.co/community/self-hosting-novu/overview
34. Pingram pricing (formerly NotificationAPI): https://www.pingram.io/pricing
