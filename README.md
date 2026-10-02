# AI for SW Engineer

A [KiroCrew](https://github.com/kirodotdev/KiroCrew) app registry: KiroCrew apps for software engineering teams.

## Use it

In KiroCrew, open **Discover → Add source** (External Registries → Add Registry) and enter:

| Field | Value |
|---|---|
| Repo | `https://github.com/warren830/AI-for-SW-Engineer` |
| Branch | `main` |

The apps below then appear in Discover, where you can install and update them. Grant trust to each
app individually when prompted; blanket third-party trust is not needed.

### Network requirement: KiroCrew must reach GitHub directly

KiroCrew clones this registry and each app repository itself. For browsing, app details and installs it
clones **anonymously, on purpose**:
- your shell's proxy variables (`HTTPS_PROXY`, …) are stripped;
- your global git config is ignored;
- SSH keys and agents are hidden.

So a VPN or proxy in **system-proxy mode is not enough**: Discover shows `0 apps`, or the app page
hangs on "Loading app details…". Then:

1. **Recommended:** switch your VPN/proxy client to **TUN / enhanced / global mode**, so all traffic is
   routed, then refresh the source in Discover. In Clash, this is the TUN (virtual network adapter) switch.
2. **If that is not possible:** download the app's source zip from its GitHub Releases page in the browser,
   unzip it, and use **Install from Path** with that directory. Updates then mean downloading the next
   release again.

Private or company-internal Git hosts (for example `gitlab.aws.dev`, which allows only Midway-signed SSH)
cannot be used as a source for these apps, because KiroCrew never uses your credentials for these clones.
Clone such repositories yourself and use **Install from Path**.

## Apps

| App | What it does | Repository | Branch |
|---|---|---|---|
| AI-DLC Studio (`aidlc-studio`) | One Action Center for every AI-DLC Workflows approval gate, question and recovery decision across your repositories | [warren830/kirocrew-app-aidlc](https://github.com/warren830/kirocrew-app-aidlc) | `release` |
| ADLC 控制台 (`workshop-customizer`) | The ADLC platform on Amazon Bedrock AgentCore: build, evaluate and canary-release agents with evidence-gated promotion, measure a skill's lift on the real agent, and turn a customer brief into a class-ready Workshop scenario pack (Autopilot) | [warren830/kirocrew-app-adlc](https://github.com/warren830/kirocrew-app-adlc) (`app/`) | `release` |

## Add an app

Append an entry to `app-registry.json`:

```json
{ "name": "<app.json name>", "gitUrl": "https://github.com/<owner>/<repo>", "repo": "https://github.com/<owner>/<repo>", "branch": "release" }
```

- `name` must equal the `name` in the app's `app.json`.
- The app repository must be public, and everything `app.json` references must be committed, including
  the built UI, because KiroCrew installs by plain clone with no build step.
- Display copy, icons and screenshots come from the app's own `app.json`, so do not repeat them here.
- Point `branch` at a release branch that only moves when a version is tagged, not at `main`.
- An app that lives in a subdirectory of its repository adds `"subdirectory": "path/to/app"`.
