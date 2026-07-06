---
layout: post
title: "CA-02 — The 3 Essential Conditional Access Policies Every Tenant Should Have"
date: 2026-06-20
categories: [identity, azure, security]
tags: [Conditional Access, Entra ID, Zero Trust, MFA]
description: "MFA, blocking legacy auth, and requiring compliant devices — the three Conditional Access policies that form the baseline every Microsoft 365 or Entra ID tenant should have before anything more advanced."
---

In the last post we covered what Conditional Access actually is and how the If/Then logic works under the hood. This time we're getting practical — three policies that form the baseline every tenant should have in place before you even think about anything more advanced.

If you only ever build three Conditional Access policies, build these.

## 1. Require MFA for All Users

This is the one everyone knows about, but it's worth being precise about *why* it matters.

**The logic:**
> **IF** any user signs in → **THEN** require multi-factor authentication

A stolen password on its own should never be enough to get into your tenant. Phishing kits, credential stuffing, and password reuse from unrelated breaches are all extremely common — MFA is the first wall that stops a stolen password from becoming a compromised account.

A few things worth getting right when you build this:

- Apply it to **all users**, not just admins. Attackers don't only target privileged accounts — a regular user mailbox is still a foothold.
- Exclude your **break-glass accounts** from this policy. If Conditional Access ever misfires, you still need a way in.
- Push everyone toward **Microsoft Authenticator or a phishing-resistant method** (FIDO2, passkeys) rather than SMS, which is the weakest MFA option available.

## 2. Block Legacy Authentication

**The logic:**
> **IF** a sign-in uses legacy authentication protocols → **THEN** block it

Legacy auth (POP, IMAP, SMTP AUTH, older Office desktop clients) doesn't support modern authentication — which means it can't enforce MFA at all. It's the single most common path attackers use to bypass MFA entirely, because the protocol simply doesn't know how to ask for it.

If you haven't checked this yet: most tenants have far more legacy auth traffic than they expect, usually from old scripts, scanners, or a handful of devices nobody's touched in years.

Before you flip this policy to **On**, run it in **Report-only** mode first (same advice as last post) and check the sign-in logs for anything still relying on legacy protocols. Better to find that out before you block it than after someone calls asking why their app stopped working.

## 3. Require Compliant Devices

**The logic:**
> **IF** a user signs in → **THEN** require the device to be marked compliant in Intune

This is the layer that stops a stolen token or password from being useful on a random, unmanaged laptop. Even if an attacker has valid credentials and gets past MFA, they still need to be doing it from a device your organisation actually trusts and manages.

This one has a real prerequisite, though: you need Intune (or another compliance-capable MDM) actually enrolling and evaluating devices first. If your device fleet isn't enrolled yet, this is the policy that tells you exactly how big that gap is.

## Quick Reference

| Policy | Applies To | Grant Control |
|---|---|---|
| Require MFA | All users (exclude break-glass) | Require multi-factor authentication |
| Block legacy auth | All users, legacy auth clients | Block access |
| Require compliant device | All users | Require device marked as compliant |

## What Comes Next

These three policies are your baseline — not your finish line. Once they're in place and stable, the next posts go further:

- **Post 3** — Geo Lock policies, blocking sign-ins by location
- **Post 4** — Advanced risk-based policies using Entra ID Protection

Got these three running already, or hit a snag rolling them out? Drop a comment below.
