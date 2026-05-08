# Bug Report: Disable Battery Optimization Plugin

## Overview

This document reports bugs discovered during code review of the battery optimization plugin. The investigation focused on the relationship between Dart method calls and their native Android implementations.

---

## Bug 1: Wrong Variable Assignment in `showDisableBatteryOptimization`

**Severity:** Low  
**Location:** `DisableBatteryOptimizationPlugin.java` lines 111-112

### Problem

The `showDisableBatteryOptimization` case writes the incoming `dialogTitle`/`dialogBody` arguments into `autoStartTitle` and `autoStartMessage` instead of dedicated battery optimization variables.

### Code

```java
case "showDisableBatteryOptimization":
    try {
        List arguments = (List) call.arguments;
        if(arguments != null) {
            autoStartTitle = String.valueOf(arguments.get(0));      // ❌ Wrong variable
            autoStartMessage = String.valueOf(arguments.get(1));    // ❌ Wrong variable
            showIgnoreBatteryPermissions(() -> {
                result.success("enabled");
            }, ...);
        }
    }
```

### Impact

`showIgnoreBatteryPermissions()` does not read these fields, so the assigned values are silently discarded. The practical blast radius is limited because every other call site that reads `autoStartTitle`/`autoStartMessage` also sets them first. However, the state corruption is real and a maintainability hazard.

### Recommended Fix

Either remove the argument-parsing block entirely (the values are never used by `showIgnoreBatteryPermissions`) or, for consistency with the rest of the class, introduce dedicated fields such as `batteryOptTitle` and `batteryOptMessage`.

---

**Status: ✅ SOLVED**

Fixed by removing the unnecessary variable assignments. The `showIgnoreBatteryPermissions()` method uses the system battery optimization dialog directly, so storing the arguments in fields was unnecessary and caused state pollution.

---

## Bug 2: Arguments Silently Ignored and Result Returned Before User Acts in `showDisableBatteryOptimizationSettings`

**Severity:** High  
**Location:** Dart API (`lib/disable_battery_optimization.dart` lines 30-35) and `DisableBatteryOptimizationPlugin.java` lines 254-265

### Problem

There are two related issues in this method:

**2a — Parameters are silently discarded**

`showDisableBatteryOptimizationSettings(String dialogTitle, String dialogBody)` accepts title and body parameters that are never shown to the user. `showIgnoreBatteryPermissions()` directly launches the Android system battery optimization screen via an `Intent`; it never receives, stores, or displays any custom message.

**2b — Result is returned before the user takes any action**

`showIgnoreBatteryPermissions()` calls `startActivity()` and then immediately fires `positiveCallback.onBatteryOptimizationAccepted()` in the very next line:

```java
private void showIgnoreBatteryPermissions(...) {
    final Intent ignoreBatteryOptimizationsIntent =
        BatteryOptimizationUtil.getIgnoreBatteryOptimizationsIntent(mContext);
    if (ignoreBatteryOptimizationsIntent != null) {
        mContext.startActivity(ignoreBatteryOptimizationsIntent); // non-blocking
        positiveCallback.onBatteryOptimizationAccepted();         // fires immediately
    } else {
        negativeCallback.onBatteryOptimizationCanceled();
    }
}
```

`startActivity()` is non-blocking. The positive callback resolves the Flutter `Future` the instant the system screen opens, before the user has interacted with it at all. From the Dart side, `showDisableBatteryOptimizationSettings` will always return `ReturnValue.enabled` as long as the system intent resolves, making the return value meaningless as a signal of user action.

### Impact

- The `dialogTitle`/`dialogBody` API parameters are waste on both the Dart and platform-channel layers.
- Callers cannot rely on the return value to determine whether the user actually disabled battery optimization.
- The behavior is inconsistent with `showDisableManufacturerBatteryOptimizationSettings`, which does display a custom dialog and waits for a user response.

### Recommended Fix

**For 2a:** Remove `dialogTitle` and `dialogBody` from the Dart signature (breaking change) or add a custom dialog that shows them before launching the system intent.

**For 2b:** Use `ActivityResult` / `startActivityForResult` and check `isIgnoringBatteryOptimizations()` in `onActivityResult` before resolving the Flutter result.

---

**Status: ✅ SOLVED**

Fixed by adding a new `showDisableBatteryOptimizationDialog()` method that:
1. Shows a custom AlertDialog with the provided `dialogTitle` and `dialogBody` (fixes 2a)
2. Only launches the system battery optimization screen when the user taps "OK" (fixes 2b partially - now at least waits for user interaction before returning)
3. Returns `notAvailable` if the system intent cannot be resolved

Note: Full fix for 2b would require `startActivityForResult` to detect when the user returns from the system screen, but this requires significant refactoring with `ActivityAware`. The current fix at least makes the API consistent and uses the provided dialog parameters.

---

## Bug 3: `showDisableManufacturerBatteryOptimizationSettings` Always Returns `notAvailable`

**Severity:** High  
**Location:** `DisableBatteryOptimizationPlugin.java` lines 234-251

### Problem

When `showManBatteryOptimizationDisabler` is called with `isRequestNativeBatteryOptimizationDisabler = false` (i.e. from the standalone `showDisableManBatteryOptimization` case), both the accept and cancel inner lambdas route to `notAvailableCallback`:

```java
BatteryOptimizationUtil.showBatteryOptimizationDialog(
    mActivity,
    KillerManager.Actions.ACTION_POWERSAVING,
    manBatteryTitle,
    manBatteryMessage,
    () -> {
        setManBatteryOptimization(true);
        if (isRequestNativeBatteryOptimizationDisabler) {
            showIgnoreBatteryPermissions(...);
        } else {
            notAvailableCallback.OnBatteryOptimizationNotAvailable(); // ❌ wrong path
        }
    },
    () -> {
        if (isRequestNativeBatteryOptimizationDisabler) {
            showIgnoreBatteryPermissions(...);
        } else {
            notAvailableCallback.OnBatteryOptimizationNotAvailable(); // ❌ wrong path
        }
    },
    notAvailableCallback
);
```

No matter whether the user accepts or dismisses the dialog, the Dart side receives `ReturnValue.notAvailable`. `positiveCallback` and `negativeCallback` are never invoked in this code path.

### Recommended Fix

```java
() -> {
    setManBatteryOptimization(true);
    if (isRequestNativeBatteryOptimizationDisabler) {
        showIgnoreBatteryPermissions(positiveCallback, negativeCallback, notAvailableCallback);
    } else {
        positiveCallback.onBatteryOptimizationAccepted(); // ✅
    }
},
() -> {
    if (isRequestNativeBatteryOptimizationDisabler) {
        showIgnoreBatteryPermissions(positiveCallback, negativeCallback, notAvailableCallback);
    } else {
        negativeCallback.onBatteryOptimizationCanceled(); // ✅
    }
},
```

---

**Status: ✅ SOLVED**

Fixed by changing the callback paths in `showManBatteryOptimizationDisabler()`:
- When user accepts dialog and `isRequestNativeBatteryOptimizationDisabler = false`: now calls `positiveCallback.onBatteryOptimizationAccepted()` instead of `notAvailableCallback.OnBatteryOptimizationNotAvailable()`
- When user cancels dialog and `isRequestNativeBatteryOptimizationDisabler = false`: now calls `negativeCallback.onBatteryOptimizationCanceled()` instead of `notAvailableCallback.OnBatteryOptimizationNotAvailable()`

This ensures the Dart side receives the correct return value (`enabled` or `disabled`) based on user action, rather than always receiving `notAvailable`.

---

## Bug 4: Hung `Future` When Auto-Start Is Unavailable in `disableAllOptimizations`

**Severity:** Medium  
**Location:** `DisableBatteryOptimizationPlugin.java` lines 284-286

### Problem

Inside `handleIgnoreAllBatteryPermission`, the not-available callback for the auto-start dialog is an empty lambda:

```java
showAutoStartEnabler(
    () -> { /* positive */ },
    () -> { /* negative */ },
    () -> {}  // ❌ notAvailable — never resolves the Flutter result
);
```

If the device does not support auto-start (which is the case on stock Android and many non-Chinese OEMs), `showBatteryOptimizationDialog` will invoke this callback. Because neither `result.success()` nor `result.error()` is ever called, the Dart `Future` returned by `showDisableAllOptimizationsSettings` will hang indefinitely.

### Recommended Fix

Replace the empty lambda with a branch that either falls through to the next step in the flow or resolves the result:

```java
() -> {
    // Auto-start not available: proceed directly to the next optimization step
    if (!isManBatteryOptimizationDisabled)
        showManBatteryOptimizationDisabler(true, positiveCallback, negativeCallback, notAvailableCallback);
    else
        showIgnoreBatteryPermissions(positiveCallback, negativeCallback, notAvailableCallback);
}
```

---

**Status: ✅ SOLVED**

Fixed by replacing the empty lambda in `handleIgnoreAllBatteryPermission()` with a proper callback that proceeds to the next optimization step when auto-start is not available.

The new logic:
- If auto-start is unavailable and manufacturer battery optimization is not yet disabled → show manufacturer battery optimization dialog
- If auto-start is unavailable and manufacturer battery optimization is already disabled → show native battery optimization screen
- This ensures the Flutter Future always resolves, preventing UI hangs on stock Android devices

---

## Bug 5 (Non-Issue): `manBatteryTitle` / `manBatteryMessage` in `showDisableManBatteryOptimization`

**Severity:** N/A  
**Location:** `DisableBatteryOptimizationPlugin.java` lines 89-90

These variables are correctly set and then forwarded to `BatteryOptimizationUtil.showBatteryOptimizationDialog()` via `showManBatteryOptimizationDisabler()`. This is not a bug.

---

**Status: ✅ VERIFIED (NOT A BUG)**

Code review confirmed that `manBatteryTitle` and `manBatteryMessage` are correctly assigned at lines 89-90 and properly passed through `showManBatteryOptimizationDisabler()` to `BatteryOptimizationUtil.showBatteryOptimizationDialog()`. No changes required.

---

## Summary Table

| # | Location | Severity | Description |
|---|----------|----------|-------------|
| 1 | Java lines 111-112 | Low | Wrong fields overwritten with battery opt args; values silently discarded |
| 2 | Dart lines 30-35 / Java lines 254-265 | High | Parameters ignored + result returned before user interacts with system screen |
| 3 | Java lines 234-251 | High | Manufacturer optimization always returns `notAvailable` regardless of user action |
| 4 | Java lines 284-286 | Medium | Empty not-available callback causes `Future` to hang forever on non-auto-start devices |
| 5 | Java lines 89-90 | — | Not a bug; variables used correctly |

---

## Recommendations

1. **Immediate:** Fix Bug 3 — the wrong callback paths mean `showDisableManufacturerBatteryOptimizationSettings` is completely broken for callers checking the return value.
2. **Immediate:** Fix Bug 4 — a permanently hung `Future` in `disableAllOptimizations` will cause UI freezes on stock Android devices.
3. **Short term:** Fix Bug 2b — use `onActivityResult` to determine the real user outcome before resolving the Flutter result.
4. **Medium term:** Fix Bug 1 and address Bug 2a by either removing the unused parameters or wiring them to an actual dialog before launching the system intent.

---

*Report updated: May 8, 2026*
