---
status: shipped
collections: [artists, chat-messages, chat-rooms, commissions, customers, notification-settings, orders, queue, studios, tips]
features: [studio-settings, studios]
---


# Transactional Emails & Discord Webhook Notifications

> **Status**: Shipped

How 2monke dispatches tenant-branded transactional emails (via Brevo SMTP and Upstash QStash) and studio Discord webhook alerts, and how artists configure notification channels in `/admin/notification-settings`.

## Per-Tenant Notification Settings (`notification-settings`)

Each studio has a dedicated document in `src/collections/NotificationSettings.ts` (managed via `src/features/studio-settings/actions/notification-settings.action.ts` and `src/features/studios/actions/get-studio.action.ts` under **My Studio → Notifications**):

- **Browser Push Toggle (`browserNotifications`)**: Renders `src/components/admin/BrowserNotificationsField.tsx` so the artist can enable web push notifications on their current device.
- **Discord Commissions Webhook (`discordWebhookUrl`)**: Discord channel webhook URL receiving embeds for new commission requests, milestone payments, and status updates.
- **Discord Comments Webhook (`discordCommentsWebhookUrl`)**: Dedicated webhook URL for client chat/discussion messages (`chat-messages` / `chat-rooms`). If left empty, comment alerts automatically fall back to `discordWebhookUrl`.
- **Event Checkboxes (all default to `true`)**:
  - `notifyOnSubmission` — **Notify on New Commission Request** (`src/collections/hooks/notifyDiscordOnSubmission.ts`, `type: 'default'`).
  - `notifyOnPayment` — **Notify on Payment Received** (`src/collections/hooks/notifyDiscordOnPaymentSuccess.ts`, `type: 'payment'`).
  - `notifyOnComment` — **Notify on Client Comment** (`src/collections/hooks/notifyOnChatMessage.ts`, `type: 'comments'`).

In `sendDiscordNotification` (`src/lib/infra/discord.ts`), if the corresponding checkbox is `false` or no webhook URL is configured, the notification silently no-ops without failing the underlying write.

## Transactional Email Pipeline (`src/lib/email/index.ts`)

All transactional emails are rendered from MJML templates with per-studio branding resolved by `resolveTenantEmailBranding` (`src/lib/email/tenant-branding.ts`: studio name, logo, primary color, reply-to address, and studio base URL) and sent through Brevo SMTP (or mocked when `EMAIL_MOCK_MODE` is enabled). When `QSTASH_TOKEN` and `QSTASH_CURRENT_SIGNING_KEY` are configured, background hooks publish jobs to `/api/webhooks/email` via Upstash QStash (`src/lib/infra/upstash.ts`).

### Shipped Email Templates
1. **Commission Request Received** (`sendCommissionReceivedEmail`): Sent to the customer (`customers`) immediately after submitting a commission (`commissions`), linking to `/dashboard/commission/[id]`.
2. **Commission Quote Ready / Approved** (`sendApprovalEmail`): Sent when the artist approves a commission and sets the quote (`Subject: Your Commission Quote for <Name> is Ready!`).
3. **Slot Waiting for Payment** (`sendSlotWaitingForPaymentEmail`): Reminder linking to `/dashboard/commission/[id]` to pay or `/dashboard/commission/[id]/close` to release the slot.
4. **Commission Declined** (`sendRejectionEmail`): Sent when an artist declines a request (`Subject: Update regarding <Name>`).
5. **Commission Payment Received** (`sendPaymentConfirmationEmail`): Sent after Stripe confirms commission payment (`src/collections/hooks/notifyCustomerOnPaymentSuccess.ts`).
6. **Queue Progress Update** (`sendProgressUpdateEmail`): Sent when the artist notifies the customer of milestone progress on `/queue/[queueId]` (`queue`).
7. **Commission Finished & Review Invite** (`sendCommissionFinishedReviewEmail`): Sent when deliverables are finalized, linking to `/dashboard/commission/[id]` and `/review/commission/[reviewToken]`.
8. **Shop Purchase Confirmation** (`sendPurchaseConfirmationEmail`): Sent upon `orders` fulfillment, linking to `/dashboard/library`.
9. **Shop Order Review Invite** (`sendOrderReviewInviteEmail`): Invites the buyer to review purchased items.
10. **Tip Received** (`sendTipReceivedEmail`): Sent to supporters after a completed tip (`tips`, `Subject: Thank you for your tip!`).
