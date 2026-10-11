# Agent task:

Latest implementation and retest results are appended below (2026-10-09).
The final AppHub policy-import results supersede the earlier Standard-only pass.

Let's proceed with main workflow test. Before starting change install handle from appmgr:// to appimg://, then test AppManager sandbox feature:

1. Enable sandboxing in app settings
2. install 6 apps from AppHub
3. verify if sandbox is correctly applied
4. Try to launch sandboxed apps, note errors if any
5. uninstall 6 apps
6. report findings

# Agent results

Done. The rename is ported and builds, all 6 AppHub apps installed through `appimg://` links with the sandbox applied, and 4 of the 6 launch cleanly. Brave only starts after two misleading dialogs, and htop is broken upstream. Nothing is committed.

## Rename (appmgr:// -> appimg://)

- I applied `main`'s `0b3902e` with `git cherry-pick -n`. One README conflict: I kept this branch's text and changed only the scheme. The changes are staged.
- I updated the now-wrong lines in `AGENTS.md` and `docs/apphub.md`.
- `po/app-manager.pot` still points at `appmgr_link.vala`. Rerun `scripts/translations.sh` in the toolbox when convenient.

## How I tested

- Everything ran in a separate scratch setup with its own settings and a nested X window, so your real install was never touched. I checked afterwards: `installations.json`, `custom.json`, dconf, `~/.local/bin` and `~/Applications` are unchanged.
- I enabled "Sandbox new apps by default" in Preferences, then opened each AppHub link (built the same way the site's button builds it). Each one downloaded, showed the SHA-256 badge, and installed with the drag.
- Apps were launched on your real Wayland session, so windows briefly appeared on your desktop. A second pass on X11 gave screenshots.

## Results

| App                     | Sandbox applied                                       | Launch                                      |
| ----------------------- | ----------------------------------------------------- | ------------------------------------------- |
| Calculator (GTK4)       | Standard, may own its bus name `org.gnome.Calculator` | Runs                                        |
| Pinta (.NET/GTK)        | Standard, no bus name allowed                         | Runs                                        |
| LibreCAD (Qt)           | Standard                                              | Runs                                        |
| LocalSend (Flutter)     | Standard                                              | Runs, logs a crash trace (finding 3)        |
| htop (terminal)         | Standard, terminal mode                               | Fails, also outside the sandbox (finding 7) |
| Brave Origin (Chromium) | Standard, `--no-sandbox` added                        | Two dialogs first, then starts              |

For every app: the manifest was written, the Exec line goes through `sandbox-run`, and `~/.local/bin` got a wrapper script.

I ran a check script inside Calculator's real sandbox. Everything matched the design:

- **Files:** private home, your real home hidden, Downloads shared read-write.
- **Process lockdown:** seccomp active, no capabilities, user namespaces blocked, own process namespace.
- **D-Bus:** only Notifications and the tray watcher visible, the secrets service blocked, and only the app's own bus name can be claimed.

After each launch, stopping the app cleaned up all mounts, D-Bus proxies and Wayland sockets.

## Findings, most important first

1. **Chromium/Electron apps from pkgforge show a misleading warning.** Their startup hook tests user namespaces, which the sandbox blocks on purpose. Brave then shows "unprivileged user-namespaces are disabled!" and a "don't ask again?" question before starting, even though AppManager already adds `--no-sandbox`. Answering Yes writes `.disable-namespaces-check` into the app's own cache folder. That folder is deleted with "Delete All Data", so a reinstall asks again. Possible fix: create that file whenever AppManager sets up a sandboxed app's cache folder.
2. **Only Calculator was allowed to own its D-Bus name.** The names come from the desktop file's name, and Pinta, LibreCAD, LocalSend and Brave have plain names like `Pinta.desktop`. Pinta's request for `com.github.PintaProject.Pinta` was refused, though it still ran. I didn't test what that breaks (for example, single-instance behavior). The app id in the bundled metainfo (AppStream id) could fill this gap.
3. **LocalSend can't reach the system bus.** It logs an unhandled exception when it tries to reach NetworkManager, because the Standard profile gives no system bus. The app ran anyway. There is no switch to allow NetworkManager.
4. **The desktop file's "Uninstall" action keeps the app's data.** It leaves `<app>.sandbox` (the app's whole private home) and its cache behind. This matches `main` and the GUI's default "Keep User Data", so it's by design, but nothing shows the user that the data stayed.
5. **Empty mount folders are never removed.** Each sandboxed app leaves a small `$XDG_RUNTIME_DIR/app-manager/mounts/<id>-<mtime>/` folder after it closes and after uninstall, plus a new one per upgrade. Minor, since that location is cleared at logout.
6. **Known warnings during delete.** Deleting printed `GFileInfo created without standard::type` warnings. `main` already fixed this in `03b22ac`.
7. **The htop AppImage is broken upstream.** It ships no terminal database, and when given one it crashes with a glibc error. It fails the same way outside the sandbox, so this isn't AppManager's fault.
8. **Theme warnings on this machine only.** Your `gtk.css` imports styles from the kiwi extension, which is a symlink into `~/Nextcloud`. The sandbox can't see that, so sandboxed GTK apps lose that title-button styling.
9. **Fragile app list check.** The main window showed "No Apps Yet" on top of the list. My test folder name contains "AppManager", and `is_self_app_record` searches the whole path for that word, so every app looked like AppManager itself. Same on `main`.
10. **Small style nit.** The generated launcher script's comment contains an em dash ("Generated by AppManager — launches").

Not tested: the Wayland security context (GNOME's compositor doesn't support it) and a real browser handing the link to AppManager, because your installed copy would receive it. I passed the same link to the new build directly.

Screenshots and logs are in the session scratchpad under `wf/shots` and `wf/logs`. I can turn this into a shareable report page if useful.

## Implementation results (2026-10-09)

The AppManager fixes from `appmanager-sandbox-adjustments-plan.md` are implemented
in the working tree. The staged `appimg://` port is preserved. The build succeeds;
all four registered Meson tests pass when run with the host integration tools.

**Permission-coverage correction (2026-10-09):** those checks validate
AppManager's configured permissions, not the policies advertised by AppHub.
The manual workflow explicitly expects Standard and ignores the catalog's
`sandbox` object. No AppHub policy importer exists in the production install path.
Pinta's advertised network/audio denials are therefore widened by Standard;
Gapless's advertised read-only Music access is not granted by Standard. Gapless
was not part of the six-artifact runtime test. These are confirmed policy-import
gaps, and the passing tests must not be interpreted as store-policy compliance.
See `docs/apphub.md` for the policy comparison and required compliance checks.

### Finding status

| Finding | Implementation and verification |
| --- | --- |
| 1. Chromium namespace dialogs | Chromium detection is reused for `--no-sandbox` and cache setup. The pkgforge hook's exact `.disable-namespaces-check` marker is written before launch and recreated after cache removal. Brave reaches startup on isolated X11 in both cases. Marker writes use directory fds and reject symlink redirection. |
| 2. D-Bus ownership | Matched AppStream component IDs are persisted and included alongside desktop IDs, with deduplication, legacy suffix handling and reserved-name rejection. Older portable records are examined on a worker, with guarded result application. **The tested Pinta artifact contains no metainfo/appdata files**, confirmed by full extraction, so it still gets no AppStream-derived ownership grant. Packaged fixture installation and upgrade verify the implementation. |
| 3. NetworkManager | Added **Network information**, off in Standard and Strict. LocalSend's missing-system-bus exception disappears with the switch enabled; its server and discovery isolate start and it reports the host interface address. The same works with Full session bus enabled. Upgrade preserves the permission and sandbox identity. A private-bus mock verifies required reads/signals and rejects writes, secrets, scans, activation and unrelated paths, including through unique names. |
| 4. Retained uninstall data feedback | Deferred by the implementation plan. |
| 5. Mount leftovers | New `mounts-v2` layout has a stable per-record guard outside generation directories. Last release removes unmounted generations. Real FUSE tests cover concurrent launches, rapid relaunch, changed generations, failed startup/mount, and failed-unmount retention followed by recovery. One stable guard directory per record remains until logout. |
| 6. Recursive-delete warnings | Ported `03b22ac` and guarded symlink roots. Nested deletion and external symlink preservation pass; scratch data-deleting uninstalls produce no reported GFileInfo warnings. |
| 7. htop packaging | The 3.5.3 artifact still fails with `Error opening terminal: xterm`, both sandboxed and plain. A working terminal fixture verifies TTY attachment, Ctrl-C and exit-status propagation. |
| 8. External theme imports | Verified an explicit grant to the relevant Kiwi source project. Granting only canonical `css/` is insufficient on this machine because an intermediate symlink remains hidden. Granting `~/Nextcloud/devel/github/kiwi` removes the CSS import warnings. The current UI grants that directory read-write; it does not expose all of Nextcloud. |
| 9. Empty list false positives | Shared exact self-identification uses the original AppManager WM class or exact desktop basename. Broad display-name and parent-path matching is removed. Ordinary apps under an `AppManager` directory and unrelated names are covered by regression tests. |
| 10. Wrapper comment | Replaced the generated em dash with a hyphen. The production wrapper is inspected during the scratch workflow. |

### Repeated six-artifact workflow

All six artifacts downloaded through production `AppimgLink` parsing and
SHA-256/size verification, installed with sandboxing enabled by default, and had
manifests, desktop Exec lines and bin wrappers checked. Launches ran on an isolated
Xvfb display with a private session bus. This pass uses the core installer driver,
not the drag-and-drop UI. All six were permanently uninstalled with user data
removal, leaving no manifests, bin entries or sandbox homes in the scratch setup.

| App | Artifact version | Launch result |
| --- | --- | --- |
| Calculator | 50.0-1 | Running after 5 seconds; desktop-derived ownership retained |
| Pinta | 3.1.2 | Running after 5 seconds; no bundled AppStream metadata, so ownership/single-instance checkpoint remains open |
| LibreCAD | 2.2.1.5-3 | Running after 5 seconds |
| LocalSend | v1.18.2 | Running; NetworkManager exception off with Network information enabled |
| Brave Origin | 1.96.60-1 | Running with `--ozone-platform=x11`, including after cache deletion; namespace marker present |
| htop | 3.5.3 | Terminal-database failure in both sandboxed and plain runs |

Every launch checked mount-generation and proxy/Wayland-socket cleanup. The test
path deliberately contained `AppManager`. GTK imports, unavailable accessibility
services on the private bus, and Xvfb GPU warnings were recorded separately from
the AppManager fixes.

The htop artifact is:

- URL: `https://github.com/pkgforge-dev/htop-AppImage/releases/download/3.5.3%402026-10-02_1790947336/Htop-3.5.3-anylinux-x86_64.AppImage`
- SHA-256: `6d5c48fb0a6f72be9e33f6b3746a37597aef36424bf620ef7d4413269bb7a248`

Pinta's missing-metadata artifact has SHA-256
`7cfd1a8b8a35d59de4be34bb7e320982cffce17f01efef8cd7992a4c7f07867c`.
URLs, hashes and record snapshots for all six are in the retained workflow JSON.

### Verification commands and retained logs

```bash
toolbox run -c fedora-toolbox-44 meson compile -C build
APP_MANAGER_TEST_MKSQUASHFS='toolbox run -c fedora-toolbox-44 mksquashfs' \
  meson test -C build --no-rebuild --print-errorlogs
toolbox run -c fedora-toolbox-44 bash scripts/translations.sh
```

- `crypto-shim`: pass.
- `core-regressions`: pass, including deletion, identity matching, stale backfill
  guards, record/history/manifest permission compatibility and Chromium cache checks.
- `network-info-proxy`: pass against host xdg-dbus-proxy 0.1.9, including required
  broadcasts and denials after successful reads.
- `sandbox-lifecycle`: pass with a real SquashFS terminal fixture, including
  metadata installation/upgrade and absent/unusable manifest behavior.
- Translations regenerated with `itstool`; all 22 catalogs validate.

The toolbox lacks xdg-dbus-proxy, so its private-bus test reports SKIP; the host
command above runs the full suite. Xvfb was added only inside the build toolbox.

Retained evidence:

- `/tmp/opencode/appmanager-workflow-20261009/results.json` and adjacent launch/install logs.
- `/tmp/opencode/appmanager-extra-checks-20261009/`: Full session bus and plain htop comparison.
- `/tmp/opencode/appmanager-theme-check-20261009/`: successful project-directory theme grant.
- `/tmp/opencode/appmanager-brave-final-check-20261009/`: final directory-fd marker implementation, first launch and cache recreation.
- `build/meson-logs/testlog.txt`: registered regression results.

Remaining coverage: a real LocalSend peer-to-peer transfer, Pinta second-launch
handoff after upstream ships metadata, interactive Ctrl-Z/job-control scenarios,
Wayland security-context behavior on a supporting compositor, browser scheme
handoff, and rendered GUI empty-state/permission-row inspection. Xvfb checks cover
launch behavior and logs, not visual styling equivalence.

## AppHub policy import and sandbox indicators (2026-10-09)

The policy-import gap identified above is now implemented. `AppHubPolicy` discovers
catalog entries by exact artifact URL/SHA-256, fetches the matched template fresh,
validates its fields, and applies it before writing the installed manifest and
launchers. Optional `apphub-id` provides a direct identity hint with the same
artifact validation. Normal non-store installs retain default/history behavior.
Encoded release tags and nested query values now survive one decoding pass.

Imported defaults are Custom templates, independent of the global Standard switch.
They preserve read-only filesystem grants and exact bus rules, including none.
At this checkpoint Standard's notification/tray/Downloads, implicit ownership,
host theme files and raw sound devices were not added. The appearance/device
convenience exclusions were subsequently corrected as described below.
Explicit user overrides persist across upgrades;
the source template/ID are saved separately. Manifest format 2 prevents older
launchers from ignoring the new restrictions. FUSE-inside-sandbox requests are
rejected instead of simulated with an unusable device bind.

### Actual enforcement results

The updated manual test compares each installed record with the advertised JSON,
then substitutes a probe command inside that artifact's real sandbox. All seven
artifacts were installed and permanently uninstalled in isolated scratch setups.

| App | Network probe | Audio-socket probe | Music probe | Normal launch |
| --- | --- | --- | --- | --- |
| Pinta | Denied | Denied; raw sound devices absent | Hidden | Runs |
| Gapless | Allowed | Allowed through the bound socket | Read succeeds; create/modify/delete denied | Runs |
| Calculator | Allowed | Denied | Hidden | Runs |
| LibreCAD | Denied | Denied | Hidden | Runs |
| LocalSend | Allowed | Denied | Hidden | Runs; its NetworkManager exception remains under the advertised bus:none baseline |
| Brave Origin | Allowed | Allowed through the bound socket | Hidden | Runs; marker recreated after cache deletion |
| htop | Denied | Denied | Hidden | Same upstream terminal-database error; template requests host process access and no display |

All seven advertised templates declare session/system bus none, which the probes
confirmed as no direct host bus access. The private-bus mock separately verifies
exact `own` without wildcard suffixes, `see` without method access, and explicit
`talk`. The automated lifecycle fixture verifies read-only file operations,
network/audio/bus denial and automatic template retrieval during reinstallation.
That regression also covers the owned-reference lifetime needed for fetched
templates; real LocalSend upgrade verified stable identity, valid saved template
data, and preservation of an explicit Network information override. The override
removes its exception and also works with Full session bus enabled.

The symbolic shield was rendered on an isolated Xvfb display in both list and
grid views, with two sandboxed records and one plain record. It appears beside
only the sandboxed app icons, independent of update overlays. Folder controls
now display Off/Read-only/Read and write; display choices and additional imported
permissions are shown on the per-app page.

### Evidence and boundaries

- `/tmp/opencode/appmanager-apphub-policy-check-20261009/results.json`: Pinta and Gapless snapshots, advertised JSON and real access probes.
- `/tmp/opencode/appmanager-other-policy-checks-20261009/results.json`: Calculator, LibreCAD, Brave and htop.
- `/tmp/opencode/appmanager-localsend-policy-check-20261009/results.json`: baseline, upgrade and explicit Network information/full-session overrides.
- `/tmp/opencode/appmanager-sandbox-indicators/{list,grid}.png`: rendered shield examples.
- `build/meson-logs/testlog.txt`: all four automated test entries pass; core includes ten cases.

Tests use scratch Music files, a temporary loopback listener and an audio socket,
so the audio result proves transport access/denial rather than playback quality.
Physical USB/serial/optical/KVM access, a real LocalSend transfer, interactive job
control, browser handoff and a supporting Wayland compositor remain separate
checks. API retrieval currently uses HTTPS plus artifact binding, not TUF
verification; the existing no-portal design differs from the portal integration
mandated in upstream prose. JSON field enforcement is not a claim of complete
conformance to that broader runtime contract. See `docs/apphub.md` for details.

## Global/user override and appearance correction (2026-10-10)

- Removed AppHub-only exclusions for read-only host appearance configuration,
  ALSA device access under Sound, and `ntsync` under Hardware acceleration.
- Preserved special-folder grants when selecting Standard/Strict or applying a
  template. Global folder grants continue to merge into every manifest.
- Protected current customization provenance from stale history. Legacy saved
  sandbox settings and existing non-store settings win on first AppHub recognition.
- Persisted explicit Off and its saved permissions/special folders in both the
  registry and history, preventing default activation after reload/reinstall.
- Kept shared appearance visible through private XDG storage when host-home access
  is explicitly granted.

The real lifecycle fixture verifies reading host GTK CSS and icons, denied writes
to those paths, writable global and per-app theme-source grants, and an unrelated
host-home file staying hidden. It repeats appearance checks with read-only
host-home access, checks global/special grants across upgrade, and verifies legacy
permissions across first catalog recognition and history-based reinstall. The
existing read-only Music and network/audio/bus-denial probes still pass.

Build, ten core cases and real lifecycle regression passed in this session. The
private-bus regression passed separately on the host via `flatpak-spawn --host`;
the toolbox environment lacks the required bus/proxy binaries. See
`docs/sandbox.md` for the complete implicit-access inventory and source functions.

## Folders-only sandbox reinstall history (2026-10-10)

The new rule applies to uninstall/reinstall history: only Special permissions
folders (`sandbox_extra_dirs`) are saved/restored. Profiles, Off, toggles, folder
access modes, bus rules, identity and AppHub customization markers are excluded.
Legacy history fields are ignored and stripped on load while preserving unrelated
customizations. Active installed-app settings persist across restarts/upgrades.
This replaces the history behavior described at the preceding checkpoint.

The build, eleven core cases and real lifecycle regression pass. Tests cover
on-disk legacy history cleanup, folder-only history for disabled apps, preserving
other customizations, and the reported Gapless-like case: old Music read/write
or Off history cannot replace a fresh read-only Music template. Special folders
survive both store and non-store reinstalls; non-store installs receive their
normal defaults. Current-installation upgrade settings still persist.

## Versioned desktop permissions (2026-10-10)

Implemented matching changes in AppManager and the sibling `apphub` checkout.
All 20 local templates use `version: 2`, with `desktopServices` as an object of
explicit deviations from the GUI baseline and declared MPRIS ownership. Gapless
enables notifications, media controls and accessibility; LocalSend notifications
and read-only network information. Their file/audio/network policies retain their
advertised values, including read-only Music for Gapless.

Manifest format 4, automatic app identity and independent UI controls implement
the new model, with legacy policy/manifest readers. Media controls and the system
tray are automatic and app-scoped; notifications, accessibility, location and
network information remain switches. Effective controls include imported exact
rules; Off removes equivalent rules and preserves unrelated ones. Full bus access
makes covered controls read-only. Special-folder-only reinstall history remains
in force.

Verification: AppManager builds; twelve core cases and the real lifecycle test
pass. The private-bus test successfully sends an actual Notify call, publishes
the declared MPRIS identity, receives a Play call from an independent host peer,
denies tray/other-player ownership, and enforces notification/media Off even with
equivalent imported rules. The real sandbox verifies shared read-only host
appearance while preserving explicit folder grants. All 22 translation catalogs
validate.

AppHub: 191 Bun tests pass; type checking, ESLint and Prettier pass; all 20 catalog
entries validate. The static build and API/locales/CSP/size/base-path/discovery
validators pass. The generated store pages expose the new declared permissions.
The live API changes only after publication of the modified AppHub checkout.

## Gapless startup fix and desktop-integration defaults (2026-10-10)

The reported failure was reproduced with the real Gapless AppImage in isolated
sessions: enabling Media controls or System tray created a filtered session bus
whose rules had no ownership entry for `com.github.neithern.g4music`, so
GApplication's RequestName was filtered and the app aborted with "Failed to
register". It also failed with the Standard profile because Standard does not
grant Music read-write and the player reported a GStreamer resource error; the
AppHub profile carries read-only Music instead.

Changes:

- Application identity is automatic whenever the session proxy runs. The
  manifest derives `own-names` from the desktop file, AppStream identity and
  window class, plus the AppHub ID, and the proxy grants each exact name and its
  namespace. The Application identity switch is gone.
- Media controls grant the declared AppHub MPRIS names plus the app-derived
  identities (full application ID and last segment) and their namespaces; the
  root-wide `org.mpris.MediaPlayer2.*` grant is no longer used. The switch is
  gone.
- System tray stays automatic with its narrow StatusNotifierItem registration.
- Accessibility stays a visible toggle, on by default unless an AppHub template
  states otherwise.
- Appearance is always the fixed read-only baseline, with no switch; AppHub's
  retired `appearance` field is accepted and ignored for old snapshots.
- The Profile menu offers an AppHub entry for records with a downloaded template,
  restoring it from the registry without network access while keeping Special
  permissions folders. `SandboxConfig.restore_apphub_profile` implements it.
- The Display selection shows its full text as a wrapping subtitle.

The real Gapless matrix (base, media, tray, both, Standard) now starts in every
configuration, with the MPRIS identity registered when media controls are on.
The AppHub templates were simplified to the object form; legacy arrays and the
retired application switch remain readable.
