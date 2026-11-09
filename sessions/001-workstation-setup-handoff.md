# Handoff 001 — Workstation setup
Date: 2026-11-09

## Done
All acceptance criteria are met.

| Tool | Version | Installed via | Notes |
|---|---|---|---|
| Homebrew | 7.0.12 | (already present) | |
| SDKMAN | script 5.23.0 | (already present) | |
| JDK | Temurin **25.0.4** LTS | SDKMAN | Was installed; now the SDKMAN default (was 17.0.20). Older JDKs left installed. |
| Gradle | **9.9.0** | SDKMAN | Newly installed and set as default (was 8.14). Runs on JDK 25. |
| OrbStack | 2.2.5 | (already present) | `memory_mib: 8192` (already set, confirmed) |
| Docker Engine | client 29.8.2 / server 29.8.2 | OrbStack | `docker run --rm hello-world` → "Hello from Docker!" |
| Docker Compose | v5.5.0 | OrbStack | |
| Node | **v26.13.0** (LTS "lithium") | nvm | Newly installed; nvm default is now `lts/*` |
| nvm | 0.40.3 | Homebrew (already present) | Brief allowed fnm *or* nvm; nvm already existed, so kept it |
| pnpm | **12.18.0** | Homebrew | New. See "Decisions" for why not Corepack |
| Git | 2.55.0 | (already present) | `user.name` = Azad Asanali, `user.email` = azneotech@gmail.com, `init.defaultBranch` = **master** (newly set) |
| kubectl | v1.37.0 | (already present) | Client only, no cluster |
| Helm | **v4.3.1** | Homebrew | New |
| minikube | v1.39.0 | (already present) | Not started (out of scope) |
| grpcurl | **1.9.4** | Homebrew | New |
| jq | 1.7.1 (Apple build) | macOS | |
| Claude Code | 2.1.319 | (already present) | |

Versions were verified from a **clean login shell**, the same as a brand-new terminal tab.

## Decisions made (and ADRs written)
No ADRs. The decisions were small enough that they're only recorded here:
- **pnpm via Homebrew, not Corepack.** Node 25+ no longer bundles Corepack, and Corepack is being deprecated. The old Corepack from Node 22 couldn't run pnpm 12, which now ships as a native binary. I undid the Corepack shim I had created (`corepack disable pnpm` in Node 22.14.0). pnpm from Homebrew works with any Node version.
- **nvm kept instead of installing fnm.** It was already installed, and the brief allowed either one.
- **Gradle 9.9.0.** This is the newest stable on SDKMAN today. Each service repo will pin its own version with the Gradle Wrapper later, so this global copy is mainly for running `gradle init`/`wrapper`.

## Files created or modified
In the repo:
- `docs/sessions/001-workstation-setup-handoff.md` (this file, created)

Outside the repo (global tool config, as approved in the plan):
- `~/.sdkman/candidates/java/current` → 25.0.4-tem; `~/.sdkman/candidates/gradle/current` → 9.9.0 (new install)
- `~/.nvm/alias/default` → `lts/*`; `~/.nvm/versions/node/v26.13.0/` (new)
- `~/.gitconfig` → added `init.defaultBranch = master`
- Homebrew: installed `pnpm`, `helm`, `grpcurl`
- Side effect: `~/.cache/node/corepack/v1/pnpm/12.18.0/` was downloaded by the failed Corepack attempt. It's harmless and safe to delete.

## Known issues / blockers
1. **Already-open terminals still show Node 22.14.0.** They started before the change. Open a new tab, or run `nvm use default`.
2. **Homebrew also has `node` 26.8.1 installed.** nvm's Node comes first in PATH, so it wins today. Having two Node installs can cause confusion later. If nothing else of yours needs it, consider `brew uninstall node`. Check `brew uses --installed node` first. I didn't remove it.
3. **Git email is `azneotech@gmail.com`.** Make sure that's the address you want on commits for this project, especially if it'll be on a public GitHub profile.

## Manual steps for Azad
1. Open a new terminal and check: `java -version`, `node -v`, `pnpm -v`.
2. Apply the IntelliJ IDEA settings below.
3. Optional: `brew uninstall node` (see known issue 2).

### IntelliJ IDEA recommendations (Java 25 + Gradle, 16 GB Mac)
- **Version:** latest stable IntelliJ IDEA (2026.x), through JetBrains Toolbox, which you already have. Java 25 needs 2025.2 or newer.
- **IDE heap:** *Help → Change Memory Settings* → **3072 MB**. That's enough for several Gradle projects, and with OrbStack capped at 8 GB it leaves about 5 GB for macOS, the browser and the IDE together. If you run many containers and things start swapping, drop it to 2048 MB.
- **Project SDK:** *File → Project Structure → SDKs* → add `~/.sdkman/candidates/java/25.0.4-tem`. Set language level **25**.
- **Gradle:** *Settings → Build, Execution, Deployment → Build Tools → Gradle*:
  - *Build and run using* **Gradle**, *Run tests using* **Gradle**. The IDE then behaves the same as CI and JaCoCo.
  - *Distribution:* **Wrapper** (each repo pins its own Gradle version)
  - *Gradle JVM:* **Project SDK (25)**
- **Gradle daemon memory:** later, in each repo's `gradle.properties`: `org.gradle.jvmargs=-Xmx1g` and `org.gradle.caching=true`. This keeps the daemon from taking too much memory.
- **Plugins:** turn off bundled plugins you won't use (Android, Kotlin Multiplatform, etc.) so the IDE uses less memory. Keep Docker, Kubernetes, Protocol Buffers, Database Tools and HTTP Client.
- **Line endings / encoding:** LF, UTF-8 (the defaults on macOS).

## What I learned (for interview prep): the concepts
- **SDKMAN vs Homebrew for JDKs.** Homebrew installs one "current" version of each package system-wide. SDKMAN is built for JVM tools: you can keep many JDK and Gradle versions side by side and switch per shell (`sdk use`) or per project (`.sdkmanrc`). Different projects need different JDKs, so SDKMAN fits Java better.
- **What OrbStack replaces.** Docker Desktop. Containers need a Linux kernel, so on a Mac a small Linux VM runs underneath. OrbStack provides that VM plus the `docker` CLI and engine. It starts faster and uses less memory and battery on Apple Silicon. The `memory_mib: 8192` cap keeps that VM from taking more than 8 GB of your 16 GB, so macOS and the IDE always have room.
- **What `docker compose` is.** A tool that reads a YAML file describing several containers (Postgres, Kafka, Grafana…) with their ports, networks, volumes and memory limits, and starts or stops them together with one command. It's our local "mini platform" before Kubernetes.
- **Why kubectl, helm and minikube are separate tools.** They do different jobs. **minikube** creates a local Kubernetes *cluster* (a single-node VM or container). **kubectl** is the *client* that talks to any cluster's API: get pods, apply manifests, read logs. **Helm** is a *package manager* that turns templated charts plus values into Kubernetes manifests and installs them as versioned releases. You could swap minikube for a cloud cluster and keep the other two the same.
- **What pnpm adds over npm.** All packages are stored once in a global content-addressed store and hard-linked into projects, so installs are faster and use far less disk. Its `node_modules` is strict, so code can't import packages it didn't declare ("phantom dependencies"). It also has first-class workspace (monorepo) support, which will help with the React shell and its remotes.

### Status after Azad's manual check
- Done: new-terminal versions, IDE heap, Project SDK, plugins.
- Done: IntelliJ Gradle defaults for new projects (Distribution = Wrapper). In this IntelliJ version, *Build and run using*, *Run tests using* and *Gradle JVM* only appear per project. **Check them once when the first Gradle repo is opened (session 003):** they should be Gradle / Gradle / Project SDK 25.
- **Open:** line endings. Optional on macOS, where the default is already LF. The setting is at *Settings → Editor → Code Style → General → Line separator*. Better: put an `.editorconfig` in each repo so the rule lives in Git, not in one IDE.

## Suggested next step
Phase 1 step 2: the `ticketing-docs` repo with an ADR template and ADR-001..003. Before that, open a new terminal and do the IntelliJ settings above.
