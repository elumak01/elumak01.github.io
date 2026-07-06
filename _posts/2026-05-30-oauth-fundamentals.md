---
layout: post
title: "OAUTH-01 - OAuth 2.0 Explained — The Foundations"
date: 2026-05-30
categories: [identity, security]
tags: [OAuth, Entra ID, Microsoft 365, Zero Trust]
description: "Before you can understand how OAuth apps work — or how attackers abuse them — you need to understand what OAuth 2.0 actually is and how it operates inside a Microsoft Entra tenant."
---

If you've read anything about modern identity security, you've seen OAuth 2.0 mentioned everywhere. But most explanations either go straight to RFC-level theory or skip the basics entirely and assume you already know what an access token is.

This post is the one that fills that gap. Before we get into app governance, attack techniques, or Conditional Access policies for OAuth apps — let's make sure the foundations are solid.

## Why OAuth Exists

Before OAuth, if you wanted a third-party app to access your email or calendar, you'd hand it your username and password. The app would then authenticate as you — with full access to everything, no expiry, no way to limit what it could do, and no way to revoke it without changing your password.

That's the problem OAuth was built to solve.

> Instead of sharing your credentials, OAuth lets you issue a temporary, scoped permission to an app — like a visitor pass that only opens specific doors and expires when the visit is done.

The app never sees your password. You stay in control of what it can access and for how long.

## The Four Players in Every OAuth Flow

Every OAuth interaction involves four roles — worth knowing these by name because you'll see them referenced constantly in Entra documentation:

- **Resource Owner** — the user who owns the data and is granting access (you)
- **Client** — the app requesting access (a third-party tool, a script, a SaaS platform)
- **Authorisation Server** — the identity provider that issues tokens (Microsoft Entra ID)
- **Resource Server** — the API or service being accessed (Microsoft Graph, Exchange Online, SharePoint)

In the Microsoft 365 world: Entra ID is always the Authorisation Server. The resource server is almost always Microsoft Graph.

## The Flows That Actually Matter in Entra

OAuth 2.0 defines several "flows" — different ways an app can request a token depending on its use case. In Entra, two of them cover the vast majority of real-world scenarios:

### Authorisation Code Flow — Delegated (User Context)

This is what happens when a user signs in and an app acts *on their behalf*.

1. User clicks "Sign in with Microsoft"
2. Entra ID presents the login and consent screen
3. User authenticates and approves the requested permissions
4. Entra issues an **authorisation code** to the app
5. App exchanges that code for an **access token** and a **refresh token**
6. App uses the access token to call Microsoft Graph as that user

The key point: Entra ID handles the sign-in. The app never touches the password. This redirect to `login.microsoftonline.com` is the moment Conditional Access policies evaluate — before Entra issues anything back to the app. The app just waits.

At that exact point, CA can enforce:

- **MFA** — Entra challenges the user for a second factor before proceeding
- **Compliant device** — is the device managed and compliant in Intune? If not, block or require remediation
- **Location/IP restrictions** — is the sign-in from a trusted named location? If not, block or step up
- **Sign-in risk** — Entra ID Protection has already scored this sign-in (impossible travel, anonymous IP, leaked credentials) — CAP can block or require MFA based on that risk score
- **Session controls** — how long before Entra forces re-authentication, and whether the browser session can persist

The app has no visibility into any of this. It either receives a token or gets a failure response — CAP is entirely Entra's domain.

> This is also why Conditional Access only protects delegated flows. Client credentials flow has no interactive sign-in step — the app goes directly to Entra's token endpoint with its secret or certificate, and there's no moment for CAP to intervene. App-only access needs a different set of controls entirely, which we'll cover later in this series.

### Client Credentials Flow — App Only (No User)

This is what background services, daemons, and automation scripts use when there's no signed-in user involved.

1. App authenticates directly to Entra using a **client secret** or **certificate**
2. Entra issues an access token scoped to the app's own permissions
3. App calls Microsoft Graph as itself — not on behalf of any user

The important security note: **there is no MFA in this flow**. Authentication is purely credential-based — which is exactly why client secrets and certificates need to be treated with the same care as privileged passwords.

### The Others — Brief Mentions

- **Device Code Flow** — for devices with no browser (think CLI tools, IoT). A legitimate flow, but frequently abused in phishing — more on that in a later post.
- **Implicit Flow** — older browser-based flow. Deprecated. If you see it in use, that's need immediate attention.

## Tokens — What They Are and Why They Matter

Once a flow completes, Entra issues tokens. Three types worth understanding:

| Token | What It Is | Typical Lifetime |
|---|---|---|
| **Access Token** | Proof of authorisation — presented to the resource server on every API call | 60–90 minutes |
| **Refresh Token** | Used to get a new access token without re-authenticating | Hours to days (configurable) |
| **ID Token** | Contains identity claims about the user — used by the app for sign-in, not API calls | Short-lived, single use |

The one that matters most from a security perspective is the **refresh token**. An access token expires in under 90 minutes — a stolen one has a short window. A refresh token can persist for much longer and can be used to silently mint new access tokens without the user re-authenticating. This is why token theft attacks target refresh tokens, not just access tokens.

## Consent Grants — The Part Most Admins Underestimate

When an app requests permissions, someone has to approve them. This is the consent grant, and it comes in two forms:

- **User consent** — the individual user approves permissions scoped to their own data. They see the consent screen and click Accept.
- **Admin consent** — a Global Admin or Privileged Role Administrator approves permissions tenant-wide, on behalf of all users. Required for high-privilege scopes like `Mail.Read` for all users or anything under application permissions.

Admin consent is a significant control point. An over-permissioned app that received admin consent has broad access across your entire tenant — not just one user's data. This is why governing which apps receive admin consent, and who can grant it, is a core part of OAuth security.

## Scopes and Permissions — Delegated vs Application

Every permission an app requests is called a **scope**, and they come in two types:

- **Delegated permissions** — the app acts on behalf of a signed-in user. The effective access is the intersection of what the app is allowed and what the user is allowed. The user is present in the flow.
- **Application permissions** — the app acts as itself, with no user context. Access is entirely determined by what permissions were granted at admin consent. No user = no user-level restriction.

Application permissions are inherently broader and higher risk. An app with `Mail.Read` as an application permission can read every mailbox in your tenant — not just the mailbox of whoever signed in.

## How This Connects to What Comes Next

Understanding these foundations is what makes everything else in OAuth security make sense:

- Why **consent phishing** works — attackers register their own app and trick users into approving it through a legitimate-looking consent screen
- Why **client credential flow abuse** bypasses MFA — there's no user auth step to intercept
- Why **refresh token theft** is the real goal of most token-based attacks — access tokens expire, refresh tokens don't (easily)
- Why **admin consent governance** matters so much — one approval can expose your entire tenant

In the next post we'll get into exactly how attackers exploit these flows in Microsoft Entra environments — techniques, real-world campaigns, and how to defend against them.

## Quick Reference

| Concept | Plain English |
|---|---|
| Authorisation Server | Entra ID — the one that issues tokens |
| Resource Server | Microsoft Graph — the API being accessed |
| Access Token | Short-lived key that opens the door |
| Refresh Token | The key that makes new keys — protect this one |
| Delegated Permission | App acts as the signed-in user |
| Application Permission | App acts as itself — no user context |
| User Consent | One user approves access to their own data |
| Admin Consent | Admin approves access tenant-wide |

## Final Thoughts

OAuth 2.0 is one of those things that looks complicated from the outside but clicks into place once you understand the four players and the two main flows. Everything else — app governance, attack techniques, Conditional Access integration — builds directly on top of this.

If any part of the above wasn't clear, drop a comment below. Getting this foundation right makes the rest of the series a lot easier to follow.

---
