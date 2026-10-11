# AppManager sandbox adjustments and implementation plan

Date: 2026-10-09

Implementation status (2026-10-09): the in-scope AppManager code changes and
registered regressions are implemented. See the appended results in
`appmanager-sandbox-tests.md` for the six-artifact retest, Pinta's missing upstream
metadata, and remaining interactive/environment-dependent coverage.

AppHub follow-up (2026-10-09): policy import and list/grid sandbox indicators are
now implemented. The updated seven-artifact workflow compares advertised policies
with installed state and actual in-sandbox operations. See the final policy-import
section of `appmanager-sandbox-tests.md` and `docs/apphub.md` for the current result.

## Basis and scope

This plan is based on `AGENTS.md`, the Findings section of
`appmanager-sandbox-tests.md`, `docs/sandbox.md`, and inspection of the relevant
implementation.

The implementation scope covers findings 1-3 and 5-10. Finding 4, concerning
feedback about data retained after uninstall, is deferred.

Address the reproducible AppManager bugs first, then application compatibility,
followed by lifecycle cleanup and a repeat of the six-app workflow. Finding
numbers below refer to the original test report; implementation steps are
numbered independently.

## Findings and proposed treatment

| Finding | Proposed adjustment | Priority |
| --- | --- | --- |
| 1. pkgforge Chromium namespace dialogs | Set the packaging hook's suppression marker during sandbox launch, after confirming its exact location and behavior | High |
| 2. Missing D-Bus ownership permissions | Extract and persist the matching AppStream ID; use it alongside the desktop ID | High |
| 3. LocalSend NetworkManager exception | Add an explicit, narrowly filtered network-information permission | Medium |
| 5. Empty mount directories accumulate | Implement cleanup with stable locking and concurrency checks | Low, but requires careful implementation |
| 6. Recursive-delete warnings | Port the small upstream fix from `03b22ac` | High |
| 7. Broken htop AppImage | Record as an upstream packaging failure; use a working terminal app for sandbox regression testing | External |
| 8. Theme imports point outside the sandbox | Document the limitation and verify an explicit, narrowly scoped folder grant | Configuration/documentation |
| 9. Incorrect "No Apps Yet" state | Replace broad name/path matching with exact AppManager identity checks | High |
| 10. Em dash in generated wrapper comment | Replace it with the repository's conventional hyphen | Low |

## Implementation constraints

- Preserve the existing staged `appmgr://` to `appimg://` port when editing
  overlapping files.
- Keep `sandbox-run` dispatch before GApplication construction.
- Preserve the launcher contract: an absent manifest allows plain execution;
  a present but unusable manifest causes exit status 3.
- Write the sandbox manifest before publishing desktop entries that invoke it.
- Keep `sandbox_app_id` stable across upgrades and separate from the application's
  own D-Bus names.
- Continue deriving rewritten launch arguments from pristine original arguments.
- Preserve the in-flight and registry-update ordering documented in `AGENTS.md`.
- Follow the existing Vala threading and GTK persistence patterns.
- Run meaningful checks appropriate to each change, then complete the final
  workflow verification.
- Leave implementation changes in the working tree for review.

## 1. Establish the baseline and isolated test environment

1. Review the existing staged scheme-renaming changes before editing overlapping
   files. At the planning baseline, `src/application.vala`, `meson.build`, and
   `po/POTFILES` are among the staged files that later steps may touch.
2. Build the current branch and record baseline results.
3. Prepare a short-path scratch environment with separate settings, registry,
   application files, launchers, and cache. Short paths avoid Unix socket-length
   failures.
4. Copy installation inputs into the scratch area because installation deletes
   or moves its input. Keep inputs outside the scratch applications directory.
5. For GUI and installer testing, follow the requirements in `AGENTS.md`:
   - Set `DBUS_SESSION_BUS_ADDRESS=disabled:` to avoid forwarding invocations to
     the installed AppManager.
   - Compile a scratch GSettings schema and set `GSETTINGS_SCHEMA_DIR`.
   - Put the bundled helper directory on `PATH`.
   - Unset inherited `APPDIR` and set `APPIMAGE` deliberately.
   - Use isolated XDG data, configuration, launcher, and cache locations, with
     a scratch applications-directory setting.
6. Test actual sandbox launches separately with the intended display and bus
   environment. `sandbox-run` dispatches before GApplication and does not need
   the GUI forwarding workaround.
7. Record the baseline logs and identify the exact AppImage artifacts used.

**Checkpoint:** baseline results correspond to the development build and scratch
registry, and the environment can reproduce the relevant findings.

## 2. Port the recursive-delete fix (finding 6)

**File:** `src/utils/file_utils.vala`

**Function:** `remove_dir_recursive()`

1. Apply the exact upstream adjustment from `03b22ac`:
   - Request both `standard::name` and `standard::type`.
   - Enumerate with `FileQueryInfoFlags.NOFOLLOW_SYMLINKS`.
2. Verify deletion of nested directories and files.
3. Verify that a symlink inside the deleted tree is removed without traversing
   or deleting its target outside the tree.
4. Repeat a data-deleting uninstall in the scratch environment and inspect the
   log for the reported GFileInfo warnings.

This fixes the underlying directory-detection problem: the current code reads
an attribute that was never requested.

**Checkpoint:** nested cleanup succeeds, external symlink targets survive, and
the warnings disappear.

## 3. Correct AppManager self-identification (finding 9)

**Files:** `src/windows/main_window.vala`, and
`src/core/installation_record.vala` if a shared helper is introduced.

**Functions:** `is_self_app_record()`, `has_only_self_records()`

The current implementation accepts a display-name prefix and searches the whole
desktop-file path for `appmanager`, including parent directories.

1. Make the primary check the exact original identity already used elsewhere:
   `original_startup_wm_class == APPLICATION_ID`.
2. If legacy records require a fallback, use an exact desktop-file basename
   match to the application ID.
3. Remove broad display-name-prefix and parent-directory-substring matching.
4. If a shared helper is introduced on `InstallationRecord`, use it consistently
   in the relevant self-record checks.
5. Verify these cases:
   - Ordinary apps installed under a directory containing `AppManager`.
   - An unrelated app whose name begins with `AppManager`.
   - AppManager itself after a custom display-name change.
   - Empty, self-only, and mixed registries.

**Checkpoint:** "No Apps Yet" appears only for an actually empty or self-only
registry.

## 4. Add AppStream-based D-Bus ownership (finding 2)

**Files:**

- `src/core/app_image_assets.vala`
- `src/core/installation_record.vala`
- `src/core/installer.vala`
- `src/core/sandbox_identity.vala`
- `src/core/sandbox_config.vala`
- `src/application.vala` for existing-record backfill

### 4.1 Extract the correct component identity

1. Add an AppStream-ID extraction helper using the existing metainfo extraction
   machinery and reuse extracted assets where possible.
2. Parse the component ID and its desktop launchable using an XML parser, such
   as GLib's markup parser.
3. Match the component to the pristine bundled desktop entry.
4. Require a meaningful match before deriving ownership permission.

The current metainfo selector gives every file a minimum score of 1. That loose
fallback is unsuitable for deciding which bus names an app may own, particularly
in multi-component AppImages.

### 4.2 Persist identity metadata

1. Add an optional `appstream_id` field to `InstallationRecord`.
2. Read and write it in the installation-registry JSON with compatible handling
   of older records where the field is absent.
3. Extract it during every fresh install and upgrade, even when sandboxing is
   currently off.
4. Treat it as bundled application metadata rather than a user customization.
5. Assign it before the install-time manifest is written.

### 4.3 Compose ownership permissions

1. Retain the existing desktop-ID derivation and side-by-side suffix handling.
2. Add the validated AppStream ID as another candidate.
3. Handle legacy AppStream IDs ending in `.desktop`.
4. Deduplicate candidates.
5. Apply `is_ownable_bus_name()` and the reserved-name exclusions to every
   candidate.
6. Keep `sandbox_app_id` separate and stable: it remains AppManager's
   Wayland/cache identity.

### 4.4 Upgrade existing installations

1. Backfill metadata for older installed records.
2. Run extraction on a worker thread rather than in `sandbox-run` or
   synchronously during window construction.
3. Track whether metadata was already examined, including apps without usable
   metainfo, to avoid repeated extraction on every startup.
4. Before applying the result, confirm that the record still refers to the same
   installation.
5. Persist the metadata and regenerate the manifest using the existing registry
   update and manifest synchronization patterns.

**Checkpoint:** Pinta can own `com.github.PintaProject.Pinta`; Calculator retains
its existing permission; unrelated and reserved names remain unavailable. Verify
second-launch behavior as well as initial startup, including side-by-side copies.

## 5. Suppress misleading pkgforge startup dialogs (finding 1)

**File:** `src/core/sandbox_bwrap.vala`

**Functions:** `compose()`, `needs_no_sandbox()`, `add_cache_dir()`

1. Inspect the affected Brave packaging hook to confirm:
   - The exact marker path.
   - Whether it is directly under `XDG_CACHE_HOME` or a package-specific
     subdirectory.
   - What the marker suppresses.
2. Determine Chromium compatibility once per launch and reuse the result for
   cache setup and `--no-sandbox`.
3. For an affected sandboxed app, create the marker in its own effective cache
   location before `AppRun` executes.
4. Make this idempotent and recreate the marker after cache deletion.
5. Preserve the existing namespace and seccomp restrictions.
6. Treat marker-write failure as a compatibility failure with a useful
   diagnostic.

**Checkpoint:** Brave starts without the two dialogs on first launch, after an
upgrade, and after "Delete All Data" followed by reinstall.

## 6. Add filtered NetworkManager information access (finding 3)

**Recommended policy:** a per-app **Network information** switch, off by default
in Standard and Strict. Enabling it makes the profile Custom.

**Files:**

- `src/core/installation_record.vala`
- `src/core/sandbox_config.vala`
- `src/core/sandbox_manifest.vala`
- `src/core/sandbox_dbus_proxy.vala`
- `src/core/sandbox_bwrap.vala`
- `src/core/sandbox_launcher.vala`
- `src/windows/details_window.vala`

1. Capture LocalSend's actual NetworkManager calls and subscriptions.
2. Add a field such as `sandbox_allow_network_info` throughout:
   - Record serialization/deserialization.
   - Customization history.
   - Upgrade copying.
   - Preset application and profile matching.
   - Manifest serialization/deserialization.
3. Add the Services-row switch and connect it to the existing permission
   persistence path, including row synchronization and toggle handlers.
4. Extend proxy rules with only the calls and signals required for network
   information:
   - Explicitly evaluate property reads, device enumeration, and change
     notifications.
   - Exclude connection modification, settings secrets, activation, and property
     writes.
5. Validate the effective restrictions using the installed `xdg-dbus-proxy`
   version before describing the permission as read-only.
6. Make filtered system-bus access independent of full-session-bus access.
   Currently `full_session_bus` suppresses the entire proxy, and `add_buses()`
   returns before binding a system proxy. The new switch should work with either
   a filtered or full session bus. Adjust proxy setup and accessibility-bus
   handling consistently with that separation.
7. Read missing permission keys as false, preserving older records and
   manifests.
8. Test against a private bus/mock NetworkManager:
   - Required reads reach the service.
   - Disallowed calls never reach it.
9. Retest LocalSend with the switch off and on, including discovery, transfer,
   and application logs.
10. Verify persistence through restart, upgrade, and history-based reinstall,
    and verify that selecting Standard or Strict resets the switch to off.

**Checkpoint:** enabling the permission removes the NetworkManager exception
and preserves LocalSend discovery/transfer behavior.

**Policy consequence:** with the recommended opt-in default, LocalSend may still
log the exception under untouched Standard settings. Making the permission
Standard-on would need a deliberate policy decision.

## 7. Implement race-safe mount-directory cleanup (finding 5)

**File:** `src/core/sandbox_mount.vala`

**Functions:** `acquire()`, `release()`, `unmount()`

A simple recursive delete after unmount would be unsafe: another launcher may
already have opened the current generation's lock file.

1. Move coordination to a stable, reusable per-record guard outside the
   generation directory.
2. Acquire that guard before creating or opening generation state.
3. Check lock-acquisition results and handle failure explicitly.
4. Keep holder locks for live-launch references.
5. On final release:
   - Obtain exclusive holder access.
   - Unmount.
   - Confirm the mount is gone.
   - Remove the empty mount point and generation bookkeeping.
   - Remove the generation directory while holding the stable guard.
6. Make `unmount()` report success so failed unmounts cannot trigger directory
   deletion.
7. Clean partial acquisition failures using the same locking discipline.
8. Add bounded stale-generation cleanup using the same checks. Avoid traversing
   payload contents during cleanup.
9. If the lock protocol changes incompatibly, use a versioned runtime layout
   so an already-running old supervisor cannot share the new coordination
   scheme. Legacy session leftovers can expire at logout.
10. Verify normal exit, simultaneous launches, rapid relaunch, failed startup,
    failed unmount, and generation changes after upgrade.

**Checkpoint:** normal exits remove generation directories; two simultaneous
launches share the mount until the last exits; failed unmounts preserve the
mount; rapid relaunches do not split coordination across lock-file inodes.

## 8. Correct the generated wrapper comment (finding 10)

**File:** `src/core/installer.vala`

**Function:** `create_bin_wrapper()`

1. Replace the em dash in the generated comment with a hyphen.
2. Inspect a regenerated wrapper.
3. Verify argument forwarding through the wrapper as part of launch testing.

## 9. Address upstream and environment-specific findings (findings 7 and 8)

### 9.1 htop

1. Retain the sandboxed/unsandboxed comparison as evidence.
2. Identify the tested artifact by URL, version, and checksum.
3. Record the terminal-database and glibc failures as upstream packaging issues.
4. Use a known-working terminal AppImage to verify terminal attachment, Ctrl-C,
   job control, and exit-status propagation.

**Checkpoint:** the report distinguishes the broken artifact from verified
terminal-sandbox behavior.

### 9.2 Theme imports

1. Document that shared GTK configuration can reference assets outside the
   sandbox's visible filesystem.
2. Verify that explicitly granting only the relevant theme-asset directory
   resolves the styling issue.
3. Explain the access implications of the current folder-grant UI, which stores
   writable grants.
4. Verify that the app remains usable without the imported styling.

**Checkpoint:** the theme issue has a documented, explicit configuration remedy
without automatically exposing the larger Nextcloud directory.

## 10. Update documentation and translations

1. Update `docs/sandbox.md` for:
   - AppStream identity derivation and backfill.
   - Chromium compatibility-marker setup.
   - Network information permission and defaults.
   - Independent session/system proxy handling.
   - Mount locking and cleanup.
   - Terminal-artifact and external-theme limitations.
2. Update `appmanager-sandbox-tests.md` with a status and verification result for
   each finding addressed by this plan.
3. Update `AGENTS.md` where implementation or test instructions have changed.
4. Add any new source files to the appropriate Meson source lists, and add files
   containing translatable strings to `po/POTFILES` where needed.
5. Regenerate translations only in an environment with `itstool`, following the
   warning in `AGENTS.md` about obsolete metainfo translations.
6. Include the existing template reference correction from `appmgr_link.vala`
   to `appimg_link.vala`.
7. Register the meaningful new regression checks with Meson where applicable.

**Checkpoint:** documentation matches the implemented behavior, translation
inputs resolve, and regression checks have a documented execution path.

## 11. Run targeted regressions and repeat the workflow

### 11.1 Build and automated checks

Run after the relevant changes:

```bash
meson compile -C build
meson test -C build
```

The existing Meson test covers crypto-shim behavior. Add focused regression
coverage for meaningful failure cases introduced or fixed by this work:

- External symlink preservation during recursive deletion.
- Exact self-identification and false-positive records.
- Matched AppStream identity selection, duplicate candidates, side-by-side
  naming, and reserved-name rejection.
- Backward-compatible record and manifest permission handling.
- Network-information persistence and effective proxy restrictions.
- Concurrent mount acquisition/release and failure cleanup.

Use controlled fixtures for metadata and permission tests. Use a private bus
with a mock service to verify denied NetworkManager operations without sending
mutating test calls to the host service. Use real FUSE/bwrap integration checks
for mount lifecycle and actual sandbox restrictions.

### 11.2 Six-app install, launch, and uninstall workflow

Repeat the report's workflow with sandboxing enabled by default:

| App | Main verification |
| --- | --- |
| Calculator | Existing bus ownership and baseline isolation |
| Pinta | AppStream-derived ownership and second-launch behavior |
| LibreCAD | Qt launch and baseline isolation |
| LocalSend | Network information toggle, discovery, transfer, and logs |
| Brave Origin | Dialog-free startup and marker recreation |
| htop | Confirm upstream failure; test terminal behavior with a working substitute |

Across the workflow, verify:

1. `appimg://` installation completes using the development build.
2. Manifests exist before generated sandbox launchers become usable.
3. Desktop entries and bin wrappers invoke `sandbox-run` correctly.
4. The private home, folder grants, seccomp, and bus restrictions match the
   requested profile.
5. Permissions and stable sandbox identity persist through restart and upgrade.
6. Mounts, proxies, Wayland sockets, and eligible mount-generation directories
   are cleaned up after launch termination.
7. Scratch uninstall cleanup completes without the recursive-delete warnings.
8. An absent manifest allows the documented plain-execution fallback, and an
   unusable present manifest causes exit status 3.
9. The main window's empty-state behavior is correct in a scratch path that
   contains `AppManager`.

### 11.3 Remaining environment-dependent coverage

1. Verify Wayland security-context behavior on a supporting compositor such as
   sway/wlroots or KDE. The report's GNOME environment only covers the fallback.
2. Test real-browser `appimg://` handoff in an isolated desktop environment so
   the installed AppManager cannot receive the test invocation.
3. Record supporting-compositor and browser-handoff results separately from
   the checks available in the original test environment.

**Completion criteria:** all in-scope AppManager fixes pass their targeted
checks; the repeated workflow records results for all six artifacts; upstream
and environment-dependent outcomes are identified explicitly; changes are ready
for working-tree review.
