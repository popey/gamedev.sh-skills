---
name: lagged-js-sdk
description: Use this skill to integrate the Lagged JavaScript SDK into a JavaScript game.
---

# Lagged JavaScript SDK

Lagged hosts HTML5 games and exposes a single global `window.LaggedAPI` after its loader script finishes. Initialization, player lookup, ads, leaderboards and achievements all hang off `LaggedAPI`.

## Overview

The supported surface area is:

- Initialization handshake (`LaggedAPI.init(devId, publisherId)`)
- Player profile lookup (`LaggedAPI.User.get`)
- Interstitial ads (`LaggedAPI.APIAds.show`)
- Rewarded ads (`LaggedAPI.GEvents.reward`)
- Native leaderboard score submission (`LaggedAPI.Scores.save`)
- Achievement unlocks (`LaggedAPI.Achievements.save`)

Banners, in-app purchases, social features (share / invite / community / rate / add-to-home-screen / add-to-favorites), platform-internal storage, remote config and a platform server-time API are not exposed by Lagged and are intentionally omitted here. Use `window.localStorage` for persistence and your own backend for server time.

## Installation

The Lagged SDK is loaded from a single script URL. Inject it dynamically and wait for `window.LaggedAPI` to appear before using it.

```javascript
const SDK_URL = 'https://lagged.com/api/rev-share/lagged.js'

function loadScript(src) {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script')
        script.src = src
        script.async = true
        script.onload = () => resolve()
        script.onerror = () => reject(new Error('Failed to load ' + src))
        document.head.appendChild(script)
    })
}

function waitFor(globalName, timeoutMs = 10000) {
    return new Promise((resolve, reject) => {
        const started = Date.now()
        const tick = () => {
            if (typeof window[globalName] !== 'undefined') {
                resolve(window[globalName])
                return
            }
            if (Date.now() - started > timeoutMs) {
                reject(new Error(globalName + ' is not available'))
                return
            }
            setTimeout(tick, 50)
        }
        tick()
    })
}
```

## Initialization

`LaggedAPI.init` requires both a developer id and a publisher id, both supplied by Lagged when the game is registered. Right after `init`, query the current player so you can read their id, name and avatar.

```javascript
async function initLagged(devId, publisherId) {
    if (!devId || !publisherId) {
        throw new Error('devId and publisherId are required')
    }

    await loadScript(SDK_URL)
    const sdk = await waitFor('LaggedAPI')

    sdk.init(devId, publisherId)

    const player = await new Promise((resolve) => {
        sdk.User.get((response) => {
            const user = (response && response.user) ? response.user : {}
            resolve(user)
        })
    })

    const profile = {
        isAuthorized: false,
        id: null,
        name: null,
        avatar: null,
    }

    // The SDK returns id > 0 for authorized players; guests come back with id 0
    if (player.id > 0) {
        profile.isAuthorized = true
        profile.id = player.id
        profile.name = player.name
        profile.avatar = player.avatar
    }

    return { sdk, profile }
}
```

## Player

Player data is read once during initialization via `LaggedAPI.User.get(callback)`. The callback receives an object shaped like `{ user: { id, name, avatar } }`. There is no separate authorization call — the player is either signed in to lagged.com (id > 0) or a guest (id = 0).

```javascript
function getPlayer(sdk) {
    return new Promise((resolve) => {
        sdk.User.get((response) => {
            resolve((response && response.user) ? response.user : { id: 0 })
        })
    })
}
```

If you need a guest fallback (random id + display name), generate it locally and persist it in `localStorage` — the Lagged SDK does not provide one.

## Storage

Lagged does not expose a platform-internal key/value store. Use the standard `window.localStorage` API for persistence; serialize objects to JSON yourself.

```javascript
function storageSet(key, value) {
    const payload = (typeof value === 'object' && value !== null)
        ? JSON.stringify(value)
        : String(value)
    window.localStorage.setItem(key, payload)
}

function storageGet(key, tryParseJson = true) {
    const raw = window.localStorage.getItem(key)
    if (!tryParseJson || typeof raw !== 'string') {
        return raw
    }
    try {
        return JSON.parse(raw)
    } catch (e) {
        return raw
    }
}

function storageRemove(key) {
    window.localStorage.removeItem(key)
}
```

## Advertisement

### Interstitial

`LaggedAPI.APIAds.show(onClosed)` displays an interstitial. The callback fires once the ad is closed (no success flag is provided — treat the callback as the close signal).

```javascript
function showInterstitial(sdk, { onOpened, onClosed } = {}) {
    onOpened?.()
    sdk.APIAds.show(() => {
        onClosed?.()
    })
}
```

Pause your simulation and mute audio in `onOpened`, resume in `onClosed`.

### Rewarded

`LaggedAPI.GEvents.reward(canShowReward, rewardSuccess)` runs a two-phase rewarded ad:

1. `canShowReward(success, showAdFn)` — `success` reports whether an ad is available. If `true`, you must call `showAdFn()` to actually display the ad. If `false`, surface a fallback to the player.
2. `rewardSuccess(success)` — fires after the ad finishes. `true` means the player watched to completion and earned the reward; `false` means the ad failed or was dismissed early.

```javascript
function showRewarded(sdk, { onOpened, onRewarded, onClosed, onFailed } = {}) {
    onOpened?.()

    const canShowReward = (success, showAdFn) => {
        if (success) {
            showAdFn()
        } else {
            onFailed?.('unavailable')
        }
    }

    const rewardSuccess = (success) => {
        if (success) {
            onRewarded?.()
            onClosed?.()
        } else {
            onFailed?.('not_completed')
        }
    }

    sdk.GEvents.reward(canShowReward, rewardSuccess)
}
```

## Leaderboards

Lagged leaderboards are native — scores are submitted server-side and viewed on lagged.com. Each board is identified by a string id you configure on the Lagged dashboard.

```javascript
function submitScore(sdk, boardId, score) {
    return new Promise((resolve, reject) => {
        const params = {
            score,
            board: boardId,
        }
        sdk.Scores.save(params, (response) => {
            if (response && response.success) {
                resolve(response)
            } else {
                reject((response && response.errormsg) || 'leaderboard_failed')
            }
        })
    })
}
```

There is no API to fetch entries or open a native popup — both are handled on the Lagged site.

## Achievements

`LaggedAPI.Achievements.save(achievements, callback)` unlocks one or many achievements in a single call. Pass either a single achievement key or an array of keys; the callback receives `{ success, errormsg }`.

```javascript
function unlockAchievement(sdk, achievement) {
    if (!achievement) {
        return Promise.reject(new Error('achievement is required'))
    }

    return new Promise((resolve, reject) => {
        const payload = Array.isArray(achievement) ? achievement : [achievement]
        sdk.Achievements.save(payload, (response) => {
            if (response && response.success) {
                resolve(response)
            } else {
                reject((response && response.errormsg) || 'achievement_failed')
            }
        })
    })
}
```

There is no list/get API — display state is owned by Lagged.

## Lifecycle

Lagged does not provide game lifecycle messages (ready / level start / level complete / gameplay start / gameplay stop) or a pause/visibility callback. Drive pause and audio state from standard browser events:

```javascript
function wireLifecycle({ onPause, onResume }) {
    document.addEventListener('visibilitychange', () => {
        if (document.visibilityState === 'hidden') {
            onPause?.()
        } else {
            onResume?.()
        }
    })

    window.addEventListener('blur', () => onPause?.())
    window.addEventListener('focus', () => onResume?.())
}
```

Combine this with the interstitial/rewarded `onOpened`/`onClosed` hooks above so the game pauses while ads are on screen.
