Symptom: Switching from a workspace on one Mac DEV instance to a workspace on another makes the terminal take a long time to become usable.
User impact: Multi-Mac terminal navigation feels stalled even though all three Mac instances are visible.
Source: User report during physical-iPhone dogfood.
Target surface: iOS physical device and macOS IROH hosts.
Build/version/tag: iOS phand1 at cmux commit 99830e9c79, compatible Mac tags phand1, phand2, phand3.
Repro workload: Open All Computers, enter a workspace on one Mac, leave it, then enter a workspace on another Mac.
Expected bad behavior: The destination terminal remains loading long enough to feel stuck before its first usable frame.

Workload:
1. Start with all three tagged Mac apps running and visible on the phone.
2. Open a terminal workspace on Mac A.
3. Return to All Computers and open a terminal workspace on Mac B.
4. Observe the interval from selecting the workspace to the first usable terminal frame.
Stop condition: The destination terminal accepts input and renders output.

Evidence:
- All three IROH peer sessions remained established during the slow switches.
- iOS acknowledged the destination render-grid subscription within tens of milliseconds.
- The destination Mac then rejected `mobile.terminal.replay` with `VIEWPORT_PENDING expected=unavailable` until a later viewport report was accepted.
- On phand3, replay rejection lasted from 21:05:23.308 to 21:05:42.740, about 19.4 seconds, before replay succeeded at 21:05:42.773.
- phand2 showed the same rejection for at least 16.9 seconds.

Root cause:
- iOS erased `viewportReportGenerationsBySurfaceID` whenever focus adopted another already-connected peer.
- Each Mac connection retained its generation-carrying detach tombstone.
- Returning to that Mac therefore sent a generationless or reused low-generation viewport with the replay. The Mac correctly rejected it as stale.
- The sequence owner is the exact Mac app instance plus terminal surface, not the currently focused client and not a globally scoped surface id.

Fix:
- Scope viewport generations by `(MacPairingKey, surfaceID)`.
- Preserve those monotonic fences across warm focus changes and reconnects, clearing them only at the account boundary.
- Carry the prepared report's owner through its asynchronous acknowledgement checks so a focus change cannot publish it through another Mac.
