---
title: Getting Started
parent: Pro
nav_order: 10
---

# Getting Started

Ductwork Pro is a commercial product with a paid license. This guide walks you through account setup, checkout, and installation.

## 1. Create an account

Sign up at [getductwork.io](https://getductwork.io).

During registration, you can optionally specify an organization name. If you skip this, a "Personal" organization is created automatically.

Once your account is active, you can invite team members to your organization. Any member can provision authentication tokens.

## 2. Complete checkout

Choose your billing cycle:

- **Monthly** — pay as you go
- **Yearly** — get two months free

See [Pricing](https://www.getductwork.io/#pricing) for current rates. Payment is processed securely through Stripe.

## 3. Provision a token

After checkout, you'll land on your dashboard where you can create authentication tokens for Bundler. Each token consists of a **key** (prefixed with `dw_`) and a **secret**.

> ⚠️ **Important:** The secret is displayed only once. Copy it immediately and store it in a password manager.

### Token strategies

Create as many tokens as your workflow requires:

| Strategy | Use case |
|----------|----------|
| Per-developer | Revoke access when someone leaves the team |
| Per-environment | Separate tokens for production, CI, and local development, etc. |
| Shared | Single token for small teams or simple setups |

## 4. Configure Bundler

You have two main options for authenticating with the Ductwork Pro gem server.

### Option A: Global Bundler config (recommended)

Configure credentials once for all projects on your machine:

```shell
bundle config set --global https://gems.getductwork.io <TOKEN_KEY>:<TOKEN_SECRET>
```

### Option B: Per-project Gemfile

Reference credentials directly in your Gemfile. Avoid committing secrets in plaintext—use environment variables instead:

```ruby
source "https://#{ENV['DUCTWORK_KEY']}:#{ENV['DUCTWORK_SECRET']}@gems.getductwork.io" do
  gem "ductwork-pro"
end
```

## 5. Install

Add Ductwork Pro to your Gemfile if you haven't already:

```ruby
source "https://gems.getductwork.io" do
  gem "ductwork-pro"
end
```

Then install:

```shell
bundle install
```

You're all set!
