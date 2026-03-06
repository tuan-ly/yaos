# Zero Config Onboarding

Self-hosted software usually dies at the onboarding step. Forcing a user to open a terminal, run OpenSSL to generate a 32-byte cryptographic secret, and paste it into a .env file guarantees a 90% abandonment rate.

YAOS implements a consumer-grade, zero-terminal claim flow, while gracefully handling the realities of infrastructure paywalls.

## The Framework Migration: Killing the CLI

The first version of YAOS was built on PartyKit. PartyKit provided an incredible early abstraction - it wrapped Cloudflare's complex Durable Objects behind a simple "Room" API and made real-time multiplayer trivially easy to bootstrap.

However, the deployment worked exclusively through their proprietary CLI. The problem is that users must login through partykit-cli to deploy, meaning we couldn't utilize Cloudflare's "One-Click Deployment" button. This violated our core onboarding goal: Zero-terminal, consumer-grade self-hosting.

To unlock the deploy button, we stripped out the PartyKit framework and ported the entire transport layer to native Cloudflare Workers using y-partyserver, handle WebSocket transport and Durable Object coordination. We define the entire infrastructure (Workers, Durable Objects, and Storage) in a standard `wrangler.toml` file, eliminating the CLI entirely and allowing users to deploy straight from their browser.

# The Single-Use Claim Architecture

When deployed, the YAOS server boots into an "Unclaimed" state.
- The user visits the Worker URL in their browser and is greeted by a lightweight, dependency-free HTML setup page.
- The browser utilizes crypto.getRandomValues() to generate a high-entropy token locally.
- The user clicks "Claim". The token is sent to the server.
- The server hashes the token (SHA-256) and stores only the hash inside a singleton Config Durable Object via an ACID transaction.
- The setup route permanently locks itself.

For subsequent authentication, the plugin utilizes modern HTTP headers—specifically Authorization: Bearer \<token>—rather than leaking the shared token into query strings, browser history, or proxies.

# The URI Protocol Handshake

To completely eliminate the copy-paste step, the setup page generates a custom deep-link: `obsidian://yaos?action=setup&host=...&token=....`

When clicked, the OS routes this directly to the Obsidian plugin, which intercepts the URI, configures its internal settings, and immediately boots the sync engine.

# Graceful Degradation and the Credit Card Wall

Because YAOS utilizes native `wrangler.toml` bindings, Cloudflare can automatically provision Durable Objects and R2 buckets upon deployment. 

However, we made the intentional product decision **not** to force the R2 bucket binding in the default deployment template. Cloudflare enforces a strict requirement: users must have a primary payment method (credit card) on file to provision an R2 bucket. If YAOS required this binding by default, the "Deploy to Cloudflare" button would hit a billing wall, and users without a configured payment profile would abandon the setup.

We solved this via Capability Negotiation:
- The default YAOS deployment provisions only the text-sync CRDT engine (Worker + Durable Object). It requires no credit card.
- When the Obsidian plugin connects, it performs a capability probe (`GET /api/capabilities`).
- If the server lacks the `YAOS_BUCKET` binding, it returns `{ attachments: false, snapshots: false }`.
- The plugin reads this and gracefully disables the attachment and snapshot UI. It continues to sync markdown text flawlessly.

![Deploy-button resilience without mandatory R2](./diagrams/deploy-button-resilience-without-mandatory-r2.webp)


Power users who want attachment sync can easily add the R2 binding later via the Cloudflare dashboard **one-step (Just add an R2 binding to the Worker)**. The server will dynamically detect the new binding, update its capabilities, and the plugin will unlock the UI without a single line of code changing.
