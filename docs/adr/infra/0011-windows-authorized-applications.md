# infra/0011 — Windows Guests: Authorized Applications, Profiles and Updates

**Status:** Proposed
**Date:** 2026-10-15 (proposed)
**Scope:** What software a Windows guest of the platform carries beyond the image (`tpl-ws2025-std-gui-gold`, the Win11 Pro workstation template when it exists), who decides it, how it is installed, updated and removed, and what a developer profile adds.
**Related:** `infra/0010-provisioning-ansible-only` (every install is a role), `security/0001` (secrets in Vault; keys per person), `identity/0004` (accounts), the Windows walk of 2026-10-14/15 (template-first, min footprint, CIS L1, egress through the platform proxy only).

---

## Context

The golden template carries the platform baseline and nothing else. The owner's first sessions (2026-10-15) asked for the applications a working host needs — a browser, a PDF reader, an editor, Git — and for developer tooling per profile. Three facts shape the answer:

- **A Windows guest has no internet of its own.** Egress is the platform proxy or nothing (default-deny outbound, `roles/windows_firewall`). Every installer, extension and update therefore comes through that proxy, and the proxy today is a package cache (apt-cacher-ng: HTTPS tunnels only).
- **Nothing is clicked.** An application that exists on a host exists because a role declares it, with a pinned version (`infra/0010`; "never latest").
- **Footprint and no-freeze are the contract.** WSL, Docker Desktop and Hyper-V-based tooling add a second kernel and reserved memory to a guest that is already NUMA-pinned; they are the opposite of the template's purpose.

## Decision

1. **The authorized set is data, in the inventory, applied by one role** (`roles/windows_apps`, `playbooks/windows-guests.yml`). A host carries the **base set** plus the sets of the **profiles** its inventory entry declares. Anything else is unauthorized and is reported by the role's audit (installed but undeclared), never silently removed on a first run.
2. **Base set** (every Windows guest and workstation): Google Chrome (Enterprise MSI), Adobe Acrobat Reader, Visual Studio Code (system installer), Git for Windows (VS Code as editor, OpenSSH of Windows as its SSH), 7-Zip. Microsoft Edge stays (in-box) with its first-run and sync surfaces off by policy. **Not** Ansible: a control node needs a POSIX host — playbooks run from the Linux controller or a Linux dev guest (profile `dev-linux-remote` below), never from Windows.
3. **Browser and VS Code extensions are part of the set**, pinned by id (Chrome/Edge: `ExtensionInstallForcelist` policy; VS Code: `code --install-extension <id>` per declared id).
4. **Profiles** (an inventory label per host, additive):
   - `dev-c`: VS Code C/C++ + CMake tools, Build Tools for Visual Studio (MSVC, Windows SDK), the C# Dev Kit + .NET SDK (the "same as Visual Studio" ask: VS Code + Build Tools cover C, C++, VC++ and C#; the Visual Studio IDE itself is licensed per developer and installed by the same role when the owner approves its license).
   - `dev-ts`: Node.js LTS, VS Code TypeScript/ESLint/Prettier extensions.
   - `dev-go`: Go toolchain, the Go extension and its tools.
   - `dev-git`: GitLab Workflow and GitHub Pull Requests extensions, GitHub CLI, `glab`.
   - `dev-linux-remote` (the default for any development): VS Code Remote-SSH to a Linux dev guest of the platform where **dev containers** actually run (Docker there, not on Windows). **No WSL, no Docker Desktop on Windows guests** — the dev container experience is the same from VS Code, the kernel is Linux where it belongs, and the Windows guest stays small and pinned.
5. **Per-person Git identity** (`gitconfig`, SSH signing key, GPG) is applied on the **workstation**, where a person signs in as themselves (`identity/0004`): from the people file and the person's Vault area (`roles/identity_keys` already escrows an ed25519 + GPG pair per person). Server guests (shared `<org>` / `<org>-user` sessions) carry no personal Git identity.
6. **Install and update policy:** installers are fetched through the platform proxy, verified by pinned version and checksum, installed with each product's silent switch (MSI/EXE) by `win_package`; a version bump in the data is the update, applied by the same role; a reboot happens only when an installer reports it is required (`win_reboot`), serialized like the patch wave. Microsoft Store and auto-updaters are off by policy (updates are declared, not pulled).
7. **Source of installers:** vendor URLs for now, each pinned; a platform-hosted mirror (the object store) as soon as the set is stable, so a build never depends on a vendor's CDN. A **real forward proxy** (squid) as a platform service replaces the package cache for Windows egress — the package cache tunnels HTTPS only, plain-HTTP vendor downloads fail through it.

## Consequences

- One PR per application or profile change; the role's audit lists what a host carries beyond its declaration.
- The golden template stays application-free: applications are per host/profile, so the image is reusable for app, database and workstation roles alike.
- A developer never installs tooling by hand; a missing tool is a profile change.

## Pending decisions (owner)

1. Approve the base set (§2) and the profile contents (§4) — or amend the lists.
2. Visual Studio IDE: VS Code + Build Tools (free) by default; the IDE only with a license you provide.
3. The forward proxy as a platform service (§7) — recommended; without it only HTTPS downloads work.
4. Chrome vs Edge as the default browser (both present; policy picks one).
