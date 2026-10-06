# Allternit

Allternit builds AI agents that work on your computer, in the cloud, and from your phone. The same workspace runs in the desktop app, at [ai.allternit.com](https://ai.allternit.com), and as a phone app at [m.allternit.com](https://m.allternit.com).

## Products

| Product | What it is | Get it |
|---------|------------|--------|
| **Allternit Desktop** | The full workspace as a Mac and Windows app | [Download](https://github.com/Allternit/desktop/releases/latest) · `brew install --cask allternit/tap/allternit` |
| **Gizzi Code** | Workspace-aware AI assistant for the terminal | `npm install -g @allternit/gizzi-code` · `brew install allternit/tap/gizzi-code` |
| **Allternit Platform** | Agent runtime, API gateway, SDKs and services | [allternit-platform](https://github.com/Allternit/allternit-platform) |

## Repositories

- [`allternit-platform`](https://github.com/Allternit/allternit-platform) — the core monorepo. Gizzi Code, the API, SDKs and services all live here, and Gizzi Code releases are published from it.
- [`desktop`](https://github.com/Allternit/desktop) — Allternit Desktop downloads.
- [`homebrew-tap`](https://github.com/Allternit/homebrew-tap) and [`scoop-bucket`](https://github.com/Allternit/scoop-bucket) — package manager installs for macOS and Windows.
- [`allternit-tts`](https://github.com/Allternit/allternit-tts) — the Kokoro text-to-speech process used by Allternit voice, kept separate because it is GPL-licensed.
- [`gizzi-code`](https://github.com/Allternit/gizzi-code) — hosts the Linux runtime binary used by Allternit's cloud computers.

## Links

[allternit.com](https://allternit.com) · [Docs](https://docs.allternit.com) · [Services](https://services.allternit.com) · [A://Labs](https://labs.allternit.com)

Security issues: email security@allternit.com. See [SECURITY.md](https://github.com/Allternit/.github/blob/main/SECURITY.md).
