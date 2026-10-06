<div align="left">

# BootstrapMate

## From unboxed to ready, on its own.

Zero-touch provisioning for **Mac and Windows**. BootstrapMate reads one manifest, downloads what a new machine needs, verifies it, and installs it, during Setup Assistant on the Mac and the Enrollment Status Page on Windows, then again after the first login.

**Built for MDM, not instead of it.** Your MDM delivers BootstrapMate; BootstrapMate lays down the rest of your management stack.

[**Mac**](https://github.com/bootstrapmate/bootstrapmate-macintosh) · [**Windows**](https://github.com/bootstrapmate/bootstrapmate-windows) · [**Mac releases**](https://github.com/bootstrapmate/bootstrapmate-macintosh/releases/latest) · [**Windows releases**](https://github.com/bootstrapmate/bootstrapmate-windows/releases/latest)

</div>

## What it does

The first minutes of a new machine decide whether it arrives managed or arrives half-built. BootstrapMate owns those minutes: a small, native, one-shot tool that your MDM installs first, which then fetches your agents, your software deployment tool, and your scripts, in order, with a progress window for the person at the desk.

- **Cross-platform** — native tools for macOS (Swift, universal arm64 + x86_64) and Windows (C#/.NET, x64 + ARM64)
- **One manifest** — a JSON or YAML file you host anywhere, listing what to install and in which stage
- **Two phases** — a root stage during Setup Assistant / ESP, and a user stage after the first login
- **Preflight decides** — a script you write chooses skip, baseline or full provisioning on every run
- **Self-healing** — a baseline run brings an already-provisioned machine's tooling back to the published versions, without reprovisioning it
- **Signature-gated** — every installer is checked against Apple Developer ID or Authenticode before it runs as root or SYSTEM, optionally pinned to your Team ID or publisher
- **Visible** — a progress dialog during provisioning (swiftDialog on the Mac, csharpDialog on Windows), and silence during skip and baseline runs
- **Reportable** — every run leaves a local record and can POST a vendor-neutral JSON summary to ReportMate, MunkiReport, or any collector

## How a run works

**Preflight → Setup Assistant → Userland**

```
MDM installs BootstrapMate (pkg / MSI)
  │
  ▼
fetch manifest ──► preflight script
                     │
       ┌─────────────┼──────────────────┐
     exit 0        exit 2          any other positive
      Skip        Baseline            Provision
       │             │                    │
     done     setupassistant      setupassistant ──► wait for login ──► userland
              items, silent       with progress dialog
```

| Preflight exit | Mode | What runs |
|---|---|---|
| `0` | Skip | Nothing. The machine is already where it should be. |
| `2` | Baseline | Root-stage items only, no dialog, no user stage, no reboot. Anything already current is skipped. |
| any other positive | Provision | The full bootstrap, root stage then user stage. |

A manifest is three lists, `preflight`, `setupassistant` and `userland`, each item naming a URL, a destination and a type. On the Mac:

```yaml
preflight:
  - name: Preflight Check
    type: rootscript
    url: https://example.com/bootstrap/preflight.sh
    file: /Library/Application Support/BootstrapMate/preflight.sh
    hash: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
setupassistant:
  - name: Munki
    type: package
    url: https://example.com/bootstrap/munkitools.pkg
    file: /Library/Application Support/BootstrapMate/munkitools.pkg
    hash: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
    packageid: com.googlecode.munki.core
    version: 6.0.0
userland: []
```

## Mac and Windows, side by side

| | Mac | Windows |
|---|---|---|
| Runs during | Setup Assistant (Automated Device Enrollment) | OOBE Enrollment Status Page |
| Delivered as | Installer package | MSI (or `.intunewin`) |
| Launched by | One-shot LaunchDaemon that removes itself after the run | The MSI at install time, plus a daily Self-Heal scheduled task |
| Item types | `package`, `rootscript`, `userscript` | MSI, EXE, PowerShell, Chocolatey `.nupkg`, sbin-installer `.pkg` |
| Signature check | `pkgutil --check-signature`, optional Team ID pin | `WinVerifyTrust`, optional publisher pin |
| Configured with | Configuration profile (`com.github.bootstrapmate`) | Intune CSP / Group Policy (bundled ADMX) or registry |
| Status for MDM | `/Library/Managed Bootstrap/last-run.json`, `managedbootstrapinstall --last-run` | `HKLM\SOFTWARE\BootstrapMate\LastRunVersion` for Intune detection |
| Pairs with | Munki | Cimian |

Both install the command-line tool `managedbootstrapinstall` beside a **Managed Bootstrap Install** app for settings, runs and logs.

## Safe to run again

BootstrapMate is designed to be run on a schedule against machines that are already in use:

- **Hash-checked downloads** — a file already on disk with the manifest's SHA-256 is never fetched again
- **Receipt-aware** — a package already installed at the manifest's version or newer is skipped before download
- **Install ledger** — a baseline run skips any payload whose hash it has already installed
- **Throttled baselines** — a minimum interval between baseline runs, with a bounded retry after a failure and an immediate run when BootstrapMate itself updates
- **One run at a time** — a second instance exits at once; a run cut off by a restart is marked `interrupted` and retried
- **Dry run** — `--dry-run` downloads and verifies every item without installing anything

## Repositories

| Repo | Stack | License | |
|---|---|---|---|
| [bootstrapmate-macintosh](https://github.com/bootstrapmate/bootstrapmate-macintosh) | Swift · SwiftUI | MIT | macOS provisioning during Setup Assistant and first login |
| [bootstrapmate-windows](https://github.com/bootstrapmate/bootstrapmate-windows) | C# · .NET · WPF · WiX | MIT | Windows provisioning during OOBE/ESP and first login |

## Releases and signing

Both repositories build **unsigned** artifacts on every push and publish them as GitHub release assets on a version tag: a `.pkg` for the Mac, and x64 and ARM64 MSIs and zips for Windows. Versions are date stamps, `YYYY.MM.DD.HHMM`. Signing, notarization and deployment happen downstream, in your own pipeline, against a pinned release. No signing identity or fleet configuration lives in these repositories.

## Part of a bigger toolkit

BootstrapMate is the first step of a managed fleet: it installs the tools that take over from there, such as [Munki](https://github.com/munki/munki) on the Mac and [Cimian](https://github.com/windowsadmins/cimian) on Windows, and it reports each run to [ReportMate](https://github.com/reportmate), so "did this machine provision cleanly?" becomes a dashboard query.

## Open source

BootstrapMate is open source under the **MIT** license. Issues and pull requests are welcome on either repository.
