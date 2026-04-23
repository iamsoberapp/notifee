## iamsoberapp fork notes

This fork is consumed directly via git URL (not `npm publish`), so a few upstream publish-time steps are done manually here and committed:

1. `ios/NotifeeCore/` is vendored in this repo. Upstream this is in `.gitignore`, because `build_ios_core.sh` copies it into the RN package at publish time.
2. `dist/` is committed. Upstream builds it in the `prepare` hook, which our consumer disables via `enableScripts: false` in its `.yarnrc.yml`.

### Local dev loop

When iterating on a change, you don't need to round-trip through GitHub for each rebuild. Two options:

**Edit in `node_modules` directly** (fastest for small changes). The consumer pulls the full checkout via git URL, so `node_modules/@notifee/react-native/` has everything — `src/`, `dist/`, `ios/`, `android/`.

- **iOS native**: Xcode recompiles `ios/NotifeeCore/*` and `ios/RNNotifee/*` on every app build. Edit and rebuild. No `pod install` needed unless you touch a podspec.
- **Android native**: Gradle recompiles `android/src/**` on every app build. Edit and rebuild.
- **TypeScript**: Metro serves from `dist/`. Run `yarn build` inside the package directory, then restart Metro with `--reset-cache`.

Two gotchas:
- `yarn install` in the consumer wipes your edits. Easy to lose work if you forget and install another dep mid-flow.
- There's no `git status` inside `node_modules`, so when the change is ready you have to copy it back to a clone of this fork before committing.

**Point the consumer at a local clone** (safer for multi-file changes). In the consumer's `app/package.json`, swap the git URL for a local path — absolute or relative (resolved from `app/package.json`):

```
"@notifee/react-native": "portal:../../path-to-repo/notifee"
```

Then `yarn install` and `cd ios && pod install` in the consumer. Edits now live in your fork checkout from the start — `yarn install` doesn't touch them and `git status` shows exactly what's pending.

### Publishing a change

1. Edit files in a clone of this fork.
2. For TypeScript changes, run `yarn install && yarn build` and commit `src/` and `dist/`. For native changes (iOS `ios/NotifeeCore/*`, `ios/RNNotifee/*`. Android `android/src/**`), just edit and commit — no build step, the consuming app compiles them.
3. Push, then take note of the commit SHA.
4. In the consumer run:

   ```
   yarn add "@notifee/react-native@ssh://git@github.com:iamsoberapp/notifee#<sha>"
   ```

5. Then run `cd ios && pod install` for iOS and `cd android && ./sbin/gradlew clean && cd ../ && npx react-native run-android.` for android.

Note: pod install won't mention Notifee when you bump the consumer's SHA — the fork's package.json version is frozen at 7.8.0, so CocoaPods sees no version change even though
▎  the path-based pod picks up the new sources.

The prebuilt core aar at `android/libs/app/notifee/core/.../core-*.aar` is proprietary and unchanged from upstream — don't touch it.

<p align="center">
  <a href="https://notifee.app">
    <img width="160px" src="https://notifee.app/logo-icon.png"><br/>
  </a>
  <h2 align="center">Notifee - React Native</h2>
</p>

---

A feature rich Android & iOS notifications library for React Native.

[> Learn More](https://notifee.app/)

## Installation

```bash
yarn add @notifee/react-native
```

## Documentation

- [Overview](https://notifee.app/react-native/docs/overview)
- [Licensing](https://notifee.app/react-native/docs/license-keys)
- [Reference](https://notifee.app/react-native/reference)

### Android

The APIs for Android allow for creating rich, styled and highly interactive notifications. Below you'll find guides that cover the supported Android features.

| Topic                                                                                    |                                                                                                                                   |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [Appearance](https://notifee.app/react-native/docs/android/appearance)                   | Change the appearance of a notification; icons, colors, visibility etc.                                                           |
| [Behaviour](https://notifee.app/react-native/docs/android/behaviour)                     | Customize how a notification behaves when it is delivered to a device; sound, vibration, lights etc.                              |
| [Channels & Groups](https://notifee.app/react-native/docs/android/channels)              | Organize your notifications into channels & groups to allow users to control how notifications are handled on their device        |
| [Foreground Service](https://notifee.app/react-native/docs/android/foreground-service)   | Long running background tasks can take advantage of a Android Foreground Services to display an on-going, prominent notification. |
| [Grouping & Sorting](https://notifee.app/react-native/docs/android/grouping-and-sorting) | Group and sort related notifications in a single notification pane.                                                               |
| [Interaction](https://notifee.app/react-native/docs/android/interaction)                 | Allow users to interact with your application directly from the notification with actions.                                        |
| [Progress Indicators](https://notifee.app/react-native/docs/android/progress-indicators) | Show users a progress indicator of an on-going background task, and learn how to keep it updated.                                 |
| [Styles](https://notifee.app/react-native/docs/android/styles)                           | Style notifications to show richer content, such as expandable images/text, or message conversations.                             |
| [Timers](https://notifee.app/react-native/docs/android/timers)                           | Display counting timers on your notification, useful for on-going tasks such as a phone call, or event time remaining.            |

### iOS

Below you'll find guides that cover the supported iOS features.

| Topic                                                             |                                                                          |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------ |
| [Appearance](https://notifee.app/react-native/docs/ios/appearance)           | Change now the notification is displayed to your users.       |
| [Behaviour](https://notifee.app/react-native/docs/ios/behaviour)            | Control how notifications behave when they are displayed to a device; sound, crtitial alerts etc.  |
| [Categories](https://notifee.app/react-native/docs/ios/categories) | Create & assign categories to notifications.          |
| [Interaction](https://notifee.app/react-native/docs/ios/interaction)                 | Handle user interaction with your notifications. |                                                    |
| [Permissions](https://notifee.app/react-native/docs/ios/permissions)                 | Request permission from your application users to display notifications. |                                                    |

### Jest Testing

To run jest tests after integrating this module, you will need to mock out the native parts of Notifee or you will get an error that looks like:

```bash
 ● Test suite failed to run

    Notifee native module not found.

      59 |     this._nativeModule = NativeModules[this._moduleConfig.nativeModuleName];
      60 |     if (this._nativeModule == null) {
    > 61 |       throw new Error('Notifee native module not found.');
         |             ^
      62 |     }
      63 |
      64 |     return this._nativeModule;
```

Add this to a setup file in your project e.g. `jest.setup.js`:

If you don't already have a Jest setup file configured, please add the following to your Jest configuration file and create the new jest.setup.js file in project root:

```js
setupFiles: ['<rootDir>/jest.setup.js'],
```

You can then add the following line to that setup file to mock `notifee`:

```js
jest.mock('@notifee/react-native', () => require('@notifee/react-native/jest-mock'))
```

You will also need to add `@notifee` to `transformIgnorePatterns` in your config file (`jest.config.js`):

```bash
transformIgnorePatterns: [
    'node_modules/(?!(jest-)?react-native|@react-native|@notifee)'
]
```

### Detox Testing

To utilise Detox's functionality to mock a local notification and trigger notifee's event handlers, you will need a payload with a key `__notifee_notification`:

```js
{
  title: 'test',
  body: 'Body',
  payload: {
    __notifee_notification: {
      ios: {
        foregroundPresentationOptions: {
          banner: true,
          list: true,
        },
      },
      data: {}
    },
  },
}
```

The important part is to make sure you have a `__notifee_notification` object under `payload` with the default properties.

## License

- See [LICENSE](/LICENSE)

---

<p>
  <img align="left" width="50px" src="https://static.invertase.io/assets/invertase-logo-small.png">
  <p align="left">
    Built and maintained with 💛 by <a href="https://invertase.io">Invertase</a>.
  </p>
</p>

---
