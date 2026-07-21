---
layout: post
title: "OA02 - How Attackers Abuse OAuth in Microsoft Entra Tenants — Techniques, Real-World Campaigns, and Defences"
date: 2026-06-12
categories: [Security, Microsoft 365]
tags: [OAuth, Entra ID, Microsoft 365, Security]
---

If you manage a Microsoft Azure environment, you've likely come across the term "OAuth attack" in threat reports or security advisories. It's worth understanding what's actually behind it — OAuth covers a wide range of flows and attack techniques, and the specifics matter when it comes to configuring the right controls in your tenant.

This post covers the OAuth attack surface in Microsoft Entra: how the most active techniques work, what real-world campaigns have demonstrated, and what defenders can do about it.

---

## Why OAuth Is Worth Attacking

OAuth 2.0 is the protocol that underpins how applications access Microsoft 365 data on behalf of users. It's deeply embedded in how Entra ID, Microsoft Graph, and third-party integrations work. When you grant an app access to your mailbox or OneDrive, Microsoft issues an **access token** — a signed credential the app uses to call Microsoft Graph API on your behalf. Tokens are scoped (they define what the app can access), and issued after a user or admin grants consent.

Here's the key point from an attacker's perspective: **once a valid token exists, it works independently of your password**. Resetting a password does not revoke an existing OAuth token — the token stays valid until it expires or is explicitly revoked by an admin. MFA does not change this either, because MFA protects the sign-in step, not the token that was already issued.

That separation between authentication and token issuance is exactly what makes OAuth worth targeting.

### Access Tokens vs Refresh Tokens — Why Persistence Matters

Two token types are relevant to understanding attacker persistence:

**Access tokens** are short-lived — typically 60 to 90 minutes in Entra ID. They are used directly to make Graph API calls. Once expired, they cannot be reused.

**Refresh tokens** are long-lived. By default, a refresh token remains valid for up to 90 days of inactivity — but each time it is used to obtain a new access token, the clock resets. An attacker actively using a refresh token can keep it alive indefinitely in practice.

There is a second factor that contributes to persistence: the **app client secret**. This is the credential the attacker's registered application uses to authenticate to Entra ID when exchanging a refresh token for a new access token. App client secrets can be configured for up to 24 months by default, and longer via PowerShell. If the attacker registered the malicious app themselves, they control this value entirely.

The combination of a rolling refresh token and a long-lived app client secret means attacker access can persist effectively indefinitely — until an admin explicitly revokes the token or disables the app, regardless of what the affected user does with their own account.

---

## Attack 1: Illicit Consent Grant

### How It Works

The illicit consent grant attack has been documented since at least 2019 and remains one of the most effective ways to gain persistent access to a Microsoft Entra tenant — particularly where user consent is not restricted.

Here is the attack chain:

1. **The attacker registers an application** in Entra ID or any external Azure tenant. App registration is a legitimate, open action — anyone with a Microsoft account can do it.
2. **The app is configured to request delegated permissions** — such as `Mail.Read`, `Files.ReadWrite`, or `Contacts.Read` — scopes that access user data via Microsoft Graph.
3. **The attacker crafts a phishing URL** pointing to Microsoft's own OAuth consent endpoint, pre-populated with their app's client ID and requested scopes. The URL is a legitimate Microsoft domain.
4. **The victim clicks the link**, signs in with their real credentials (completing MFA if required), and is presented with a consent prompt asking them to approve the app's permissions.
5. **The victim approves** — because the app is named something convincing like "SharePoint Document Sync".
6. **Microsoft issues an OAuth token** to the attacker's app. The attacker now has persistent, delegated access to that user's data.

The attacker never touched the user's password. The victim completed MFA themselves during the sign-in flow. The attacker simply receives the token once consent is granted.

### What Makes It Dangerous

- **No credentials are stolen.** Detection based on password spray, brute force, or impossible travel will not fire.
- **MFA is not relevant here.** The victim completes MFA themselves as part of the normal sign-in. The attacker never authenticates — they receive a token after the victim does.
- **Access survives password resets.** The OAuth grant exists independently of the user's credentials.
- **Persistence is long-term.** The combination of a rolling refresh token and a long-lived app client secret means access continues until explicitly revoked.
- **It is quiet.** The sign-in looks completely normal in audit logs. There is no failed authentication noise.

### Real-World Reference

This technique has been used across multiple documented campaigns targeting Microsoft 365 tenants. Threat actors have registered fake applications impersonating well-known services — including SharePoint, Adobe, DocuSign — to trick users into granting consent. In early 2025, Proofpoint reported a campaign that attempted to compromise accounts across more than 900 Microsoft 365 environments, with a confirmed success rate exceeding 50%.

Microsoft and CISA have also documented illicit consent grant as a technique used by nation-state actors to establish persistent access to cloud environments, particularly in campaigns targeting government and critical infrastructure organisations.

---

## Attack 2: Device Code Phishing

### How It Works

Device code phishing abuses a different OAuth flow entirely — the **device authorization grant** (RFC 8628). This flow was designed for input-constrained devices such as IoT devices that cannot easily open a browser.

The legitimate flow works as follows:
1. The device requests a short-lived code from the identity provider.
2. The user goes to a separate device, visits a verification URL, enters the code, and approves access.
3. The device polls the identity provider and receives a token once the user approves.

Attackers weaponise this flow:

1. **The attacker initiates a device code request** against Microsoft's authentication endpoint using their own registered application.
2. **A device code and a verification URL are generated** — both on legitimate Microsoft domains (e.g. `microsoft.com/devicelogin`).
3. **The attacker sends the code to the victim** via phishing email, Teams message, or other social engineering — framed as a document share, a security re-verification prompt, or an MFA setup step.
4. **The victim visits the legitimate Microsoft URL**, enters the code, and approves — believing they are completing a normal authentication step.
5. **The attacker's application receives a valid access token and refresh token** for the victim's account.

Like the illicit consent grant attack, the victim completes any MFA challenge themselves as part of the normal sign-in. The attacker never interacts with the authentication step — they wait for the token to be issued once the victim approves.

### What Makes It Dangerous

- **The URLs involved are real Microsoft domains.** There is nothing obviously fake for the victim to identify.
- **MFA provides no protection.** The victim's MFA approval is part of completing the sign-in that issues the token. MFA verifies the person signing in — in this attack, that person is the legitimate user. The attacker receives the resulting token without ever authenticating themselves.
- **The attacker receives both an access token and a refresh token upfront.** Combined with a long-lived app client secret, this creates the same effectively indefinite persistence described earlier.
- **Readily available tooling lowers the bar.** Phishing kits such as SquarePhish and Graphish automate the device code phishing flow end to end, making this technique accessible to lower-skill threat actors.

### Real-World Reference

Microsoft publicly disclosed in February 2025 that a Russia-aligned threat actor (tracked as Storm-2372) had been running device code phishing campaigns since mid-2024, targeting government agencies, NGOs, defence contractors, and energy sector organisations across Europe, North America, and the Middle East. By late 2025, multiple independent threat clusters — both financially motivated and state-aligned — had adopted the technique at scale across hundreds of Microsoft 365 organisations.

---

## Other OAuth Abuse Techniques to Be Aware Of

Illicit consent grant and device code phishing are the most actively exploited techniques right now, but they are not the only OAuth abuse vectors relevant to Microsoft 365 environments.

**OAuth token theft** — Rather than tricking a user into issuing a token, an attacker can steal an existing token directly from browser storage, memory dumps, proxy logs, or compromised endpoints. A stolen token is just as valid as one obtained through consent. This is particularly relevant in post-exploitation scenarios where an attacker has already gained a foothold on an endpoint.

**App impersonation and typosquatting** — Attackers register applications with names, icons, and descriptions that closely mimic legitimate Microsoft services or well-known third-party apps. The goal is to make the consent prompt appear trustworthy enough for a user or admin to approve. This technique is what underpins most illicit consent grant campaigns.

**Over-permissioned app exploitation** — Not strictly an attack technique on its own, but legitimately consented applications with excessive Microsoft Graph permissions represent a persistent risk. If a third-party app with `Mail.ReadWrite.All` or `Files.ReadWrite.All` is compromised at the vendor level, the impact extends directly into your tenant. Regular OAuth app permission audits are a key control here.

---

## Side-by-Side Comparison

| | Illicit Consent Grant | Device Code Phishing |
|---|---|---|
| **OAuth flow abused** | Authorization code grant | Device authorization grant |
| **What the victim does** | Approves a consent prompt after clicking a phishing link | Enters a device code on a legitimate Microsoft page |
| **MFA effective?** | No — victim completes MFA themselves; attacker never authenticates | No — victim completes MFA themselves; attacker never authenticates |
| **Credential theft required?** | No | No |
| **Access persists after password reset?** | Yes | Yes |
| **What the attacker gets** | Delegated OAuth token for Graph API | Access token + refresh token |
| **Persistence mechanism** | Rolling refresh token + long-lived app client secret | Rolling refresh token + long-lived app client secret |
| **Detection difficulty** | High — looks like a normal consent event | High — sign-in occurs on legitimate Microsoft URL |
| **Primary defence** | Disable user consent; require admin approval | Block device code flow via Conditional Access |

---

## What Defenders Should Focus On

**For illicit consent grant:**
- Disable user consent for applications — require admin approval for all app consent requests.
- Enable the admin consent request workflow so users can submit requests through a controlled process rather than being blocked outright.
- Regularly audit OAuth app permissions in your tenant using the Entra admin portal or Microsoft Graph API.
- Review and restrict existing consent grants, particularly apps with broad Graph API permissions such as `Mail.ReadWrite` or `Files.ReadWrite.All`.

**For device code phishing:**
- Create a Conditional Access policy targeting the **Authentication Flows** condition and set **Device Code Flow** to Block. Microsoft introduced this condition specifically in response to this attack pattern.
- Before enforcing the block, audit your environment for legitimate device code dependencies such as legacy IoT integrations or service accounts that rely on this flow.
- Train users to never enter a device code they did not personally initiate from a known device.

**Across all OAuth abuse techniques:**
- Monitor Entra ID audit logs for new app registrations, consent grants, and unusual token issuance events.
- Use Microsoft Defender for Cloud Apps OAuth app governance to flag apps with high-risk or overly broad permission scopes.
- Include OAuth token revocation (`revokeSignInSessions`) in your incident response playbook — a password reset alone is not sufficient to terminate attacker access.
- Periodically review app client secret expiry dates for registered applications in your tenant. Long-lived secrets on apps with broad permissions represent a persistent risk if the app is ever compromised or misused.

---

## Closing Thought

The common thread across all of these techniques is this: **OAuth tokens are the target, not passwords.** MFA is highly effective against credential-based attacks — but it was never designed to protect the token issuance process. These attacks operate after authentication completes, in a layer where MFA has already done its job and has nothing further to verify.

As organisations continue to harden against credential theft and phishing-resistant authentication becomes more widespread, expect attackers to keep targeting the consent and token layer. The controls exist in Entra ID to significantly reduce this attack surface. The question is whether they are configured.

---

*Related reading:*
- *Microsoft: Detect and Remediate Illicit Consent Grants in Office 365*
- *Microsoft Security Blog: How to Protect Your Organisation Against Device Code Phishing*
- *CISA Advisory: OAuth 2.0 Abuse in Cloud Environments*
- *Proofpoint Threat Research: Microsoft OAuth App Impersonation Campaign (2025)*
- *Microsoft Threat Intelligence: Storm-2372 Device Code Phishing Campaign (February 2025)*
