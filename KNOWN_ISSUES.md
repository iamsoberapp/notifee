# Known Android notification issues

This fork patches a set of bugs in `@notifee/react-native`'s Android wrapper
that cause notification events to be silently dropped, particularly during
backgrounding/foregrounding transitions and React Native bridge cold-starts.
Some bugs also live inside the precompiled `app.notifee.core` AAR
(`android/libs/app/notifee/core/202108261754/`); those are documented here for
reference but cannot be patched without rebuilding the upstream library.

The user-visible symptoms these fixes target:

1. **"Sometimes Android pushes don't arrive."** Notification appears in the
   tray but the JS event handler (`onForegroundEvent` / `onBackgroundEvent`)
   never fires, so any in-app routing or state update tied to the event is
   skipped.
2. **"Backgrounded → flurry on foreground."** No notifications surface while
   the app is backgrounded; multiple events fire all at once when the user
   foregrounds the app.

## Issues fixed in this fork

### 1. `clearRunningHeadlessTasks` corrupts its bookkeeping (`NotifeeReactUtils.java`)

```java
for (int i = 0; i < headlessTasks.size(); i++) {
  GenericCallback callback = headlessTasks.valueAt(i);
  callback.call();
  headlessTasks.remove(i);   // BUG: SparseArray.remove(int) removes by KEY, not index
}
```

Two stacked problems:

- `SparseArray.remove(int)` removes by **key**, but `i` here is a loop index.
  The keys are React Native task IDs from `taskContext.startTask` (arbitrary
  ints), so most calls do nothing while occasional accidental key collisions
  remove the wrong entry.
- Even if the call were `removeAt(i)`, mutating the array while incrementing
  `i` would skip every other element.

Net effect: when the catalyst instance is destroyed (JS reload, process
recovery, the OS killing the bridge), running headless-task bookkeeping is
not actually cleared. Stale callbacks linger and the `headlessTasksListener`
can stay attached to a dead `HeadlessJsTaskContext`, leaving the wrapper in an
inconsistent state on the next bridge init.

**Fix:** iterate from the end and use `removeAt(i)`, holding the
`headlessTasks` monitor for the whole pass.

### 2. `startHeadlessTask` silently drops events when the bridge isn't ready (`NotifeeReactUtils.java`)

`taskContext.startTask(taskConfig)` throws `IllegalStateException` if the
React catalyst isn't actually active — common during cold-start races and
after process recovery from WorkManager triggers. The call is invoked from a
greenrobot EventBus subscriber, which catches and swallows subscriber
exceptions by default, so the event vanishes with no log and no retry.

`addTaskEventListener` is also called *before* `startTask`, so a synchronous
failure leaves the listener attached to a context with zero registered tasks.

**Fix:** wrap `startTask` in try/catch, log on failure, run the completion
callback so callers don't deadlock, and detach the listener if no other tasks
are running.

### 3. Hardcoded 100 ms post-init delay races the bridge ready state (`NotifeeReactUtils.java`)

```java
new Handler(Looper.getMainLooper()).postDelayed(callback::call, 100);
```

After cold-starting the React context for a background notification event,
the wrapper waits exactly 100 ms before calling `taskContext.startTask`. On
slower devices or with heavier JS bundles, the catalyst instance isn't yet
active; `AppRegistry.registerHeadlessTask` (called at JS module construction
time) may not have run; the registered handler is missing; the task starts
and immediately resolves as a no-op. Combined with bug #2, the event is
dropped silently.

**Fix:** replace the magic delay with a bounded poll on
`reactContext.hasActiveCatalystInstance()` (50 ms × 50 attempts ≈ 2.5 s
ceiling), then proceed.

### 4. Foreground events are silently dropped when the bridge is inactive (`NotifeeEventSubscriber.java` + `NotifeeReactUtils.sendEvent`)

```java
if (isAppInForeground()) {
  sendEvent(...)            // returns silently if reactContext is null/inactive
} else {
  startHeadlessTask(...)
}
```

`ProcessLifecycleOwner` reports `RESUMED` while the JS bridge can be torn
down or mid-init — for example during JS reload, after process recovery, or
during the brief window where Notifee Core's `NotificationReceiverActivity`
has dispatched a press event but the user's launcher Activity hasn't yet
brought the bridge up. `sendEvent` returns silently in that state, the event
is lost, and there is no fallback path.

**Fix:** make `sendEvent` return a boolean indicating delivery; when it
fails, fall through to the headless-task path (which handles bridge
initialization). This preserves the original semantics for the happy path
and only alters behaviour where the event would have been dropped.

## Issues observed in `app.notifee.core` (not patched here)

These live in the precompiled AAR (`core-202108261754.aar`) and require
rebuilding the upstream library to fix. They are documented so consumers can
work around them at the call site.

### Press events on Android 12+ are dispatched before the host Activity is ready

`NotificationReceiverActivity.onCreate` posts `NotificationEvent(TYPE_PRESS)`
on the EventBus (synchronous on `ThreadMode.MAIN`), then calls `finish()` and
launches the host Activity via Intent. The press event flows through the
wrapper while `getReactContext()` is typically still null — issues #2, #3,
and #4 above mitigate this race but cannot eliminate it entirely.

### `Notifee.getInstance()` race creates an instance with no event subscriber

```java
public static Notifee getInstance() {
  if (!c) {
    Logger.w("API", "getInstance() accessed before event listener is initialized");
    b = new Notifee();
  }
  return b;
}
```

`c` is non-volatile. If anything (e.g., a `WorkManager` worker that survived
a process restart) calls `getInstance` before `InitProvider.onCreate` has run
`Notifee.initialize(new NotifeeEventSubscriber())`, the resulting `Notifee`
has no listener registered against `EventSubscriber`. Subsequent
`displayNotification` calls then post `NotificationEvent(TYPE_DELIVERED)` to
an empty subscriber set — the event is lost. The notification still appears
in the system tray, so the user perceives "the push arrived but my
background handler didn't run."

### WorkManager-triggered notifications fire `DELIVERED` before the wrapper subscriber is registered

`Worker.startWork → c.a → c.b` posts `NotificationEvent(TYPE_DELIVERED)` from
a background `ExecutorService`. WorkManager can wake the process cold; if
`NotifeeInitProvider.onCreate` hasn't yet called `Notifee.initialize`, the
listener isn't registered when the event posts and the event is lost.

### Wake-locks acquired and never released

```java
if (notificationAndroidModelA.getLightUpScreen().booleanValue()) {
  PowerManager powerManager = (PowerManager) e.a.getSystemService("power");
  if (!powerManager.isInteractive()) {
    powerManager.newWakeLock(805306394, "Notifee:lock").acquire();
    powerManager.newWakeLock(1, "Notifee:cpuLock").acquire();
  }
}
```

Both wake-locks (a full screen-bright lock and a partial CPU lock) are
acquired with no timeout and never released. Every notification that uses
`lightUpScreen: true` permanently retains them, eventually triggering
Android's wake-lock watchdog and pushing the app into restricted/Doze
buckets — which itself causes FCM to defer messages.

**Workaround:** do not pass `lightUpScreen: true` in `displayNotification`
calls until the AAR is rebuilt.

### `ReceiverService` is a plain `Service` (Android <31 press path)

For `targetSdkVersion < 31`, notification press routes through a non-
foreground `Service`. The system temp-allowlists the start, but the OS can
kill the process at any moment while the wrapper waits for the bridge,
losing the press event on low-RAM devices.

## Symptom → cause map

| Symptom | Likely cause(s) |
| --- | --- |
| Notification appears in tray but `onForegroundEvent` / `onBackgroundEvent` never runs | #2, #4 in this fork; AAR `Notifee.getInstance()` race; AAR `WorkManager` race |
| Tap notification, app opens, in-app routing skipped | #2, #3, #4 in this fork; AAR `NotificationReceiverActivity` race |
| Backgrounded → flurry on foregrounding | #1, #2, #4 in this fork; FCM holding messages because the app entered restricted bucket (often itself caused by the AAR wake-lock leak) |
| Pushes appear delayed by minutes/hours | Upstream of Notifee — FCM priority, manufacturer power management (Xiaomi/Huawei/OnePlus/Samsung), Doze, App Standby. Audit `priority: high` on the FCM payload first. |
