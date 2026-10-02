# AI for SW Engineer

A [KiroCrew](https://github.com/kirodotdev/KiroCrew) app registry: KiroCrew apps for software engineering teams.

## Use it

In KiroCrew, open **Discover → Add source** (External Registries → Add Registry) and enter:

| Field | Value |
|---|---|
| Repo | `https://github.com/warren830/AI-for-SW-Engineer` |
| Branch | `main` |

The apps below then appear in Discover, where you can install and update them. KiroCrew clones this
repository and each app's repository itself, so the gateway needs direct HTTPS access to GitHub.
Grant trust to each app individually when prompted; blanket third-party trust is not needed.

## Apps

| App | What it does | Repository | Branch |
|---|---|---|---|
| AI-DLC Studio (`aidlc-studio`) | One Action Center for every AI-DLC Workflows approval gate, question and recovery decision across your repositories | [warren830/kirocrew-app-aidlc](https://github.com/warren830/kirocrew-app-aidlc) | `release` |

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
