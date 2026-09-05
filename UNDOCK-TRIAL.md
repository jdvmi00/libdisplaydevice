# Undock recovery trial

Base: Sunshine v2026.516.143833 (14ffa6fdaa53f7b51512be2b3d24f3939695403c),
using libdisplaydevice fe7e6a81f65deae91594702e1a185f47229745b9.

## Reproduction

1. Dock a Windows laptop with its lid closed and an external monitor active.
2. Start a Sunshine stream with `ensure_only_display` targeting a virtual display.
3. Unplug the dock during streaming, then disconnect the last streaming client.
4. Open the lid.

The original library cannot restore the absent external monitor. Its error guard
activates all available outputs, including the virtual display. The original
snapshot is retained, so subsequent disconnects repeat the failure.

## Change

After an initial topology switch fails, enumerate available devices. Only if an
original device has disappeared, try the surviving original outputs. If none
survive, use a positively identified internal panel (Windows internal, LVDS,
embedded DisplayPort or embedded UDI connector). Unknown connectors are not
assumed to be laptop panels.

Preserve unrelated currently active outputs. Activate the replacement alongside
the current output and read it back before removing any outputs activated by the
stream (modified topology minus initial topology). Read back the final topology
before completing restoration and clearing persistence. A missing replacement,
failed write, failed readback or failed persistence clear retains pending
recovery. Mode/HDR/primary restoration must still succeed before this fallback;
this does not discard unresolved changes to those settings.

The normal restore path and Sunshine's device-change retry mechanism are unchanged.
Device enumeration JSON adds `is_internal`; older JSON remains readable.

## Validation

The platform-independent suite runs locally. The `Undock trial` Windows workflow
runs the complete library suite, including the undock regressions, then builds
and tests the matching Sunshine source with this library patch. No releases or
upstream pull requests are published by this workflow.

Live validation must verify normal disconnect, closed-lid undock followed by
opening the lid, redocking, and a subsequent connect/disconnect cycle. The old
helper must remain disabled. Keep the original Sunshine executable and config
backup for rollback; do not replace the installed executable before tests pass.

## Scope

This is a conservative laptop recovery path. If every original output is absent
and no internal panel is exposed, recovery remains pending. It does not guess
which newly attached external or virtual display should replace the old layout.
Native API readback verifies the Windows configuration; a person must confirm the
physical screen is visible. A switch can still race a concurrent hardware change;
a failed result retains the recovery record for a later retry.

## Live trial and docking persistence follow-up

The d906331 trial passed the closed-lid undock/reopen test on the laptop:
internal 1920x1200 became primary, virtual output became inactive, and Sunshine
removed the recovery record. A subsequent undocked stream also restored correctly.
The SSH helper stayed disabled throughout.

Redocking exposed another issue. Native QDC_DATABASE_CURRENT readback showed
virtual-only as the saved docked topology. After explicitly saving Dell-only,
starting a Sunshine stream changed that saved topology to virtual-only again.
Normal disconnect restored both active and saved Dell-only layouts, but undocking
mid-stream left the absent dock's saved layout at virtual-only.

The follow-up adds an opt-out of CCD database writes to WinDisplayDevice, defaulting
to the existing behavior. The accompanying Sunshine patch opts out for streaming.
Temporary topology uses SDC_USE_SUPPLIED_DISPLAY_CONFIG plus SDC_ALLOW_CHANGES,
not SDC_TOPOLOGY_SUPPLIED (which the live test showed changes the remembered
layout). Mode, primary, and rollback CCD calls likewise omit SDC_SAVE_TO_DATABASE.
HDR behavior is unchanged; HDR changes are disabled in this trial configuration.
Windows selects modes for temporary topology; actual virtual and physical modes
must be verified on hardware, alongside saved-layout readback while streaming.

The follow-up is not yet hardware validated. The first trial remains installed,
with the Dell-only docked layout restored, until the new build passes its checks.
