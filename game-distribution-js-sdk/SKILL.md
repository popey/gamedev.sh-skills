---
name: game-distribution-js-sdk
description: Use this skill to integrate the GameDistribution JavaScript SDK into a JavaScript game.
---

# GameDistribution JavaScript SDK

## Overview

GameDistribution exposes a single browser SDK (`window.gdsdk`) configured via a global `window.GD_OPTIONS` object. The SDK loads, fires lifecycle events through a single `onEvent` callback, and exposes promise-returning methods to preload and show ads. The native SDK supports:

- Initialization with a `gameId` and a single `onEvent` callback
- Interstitial ads (`gdsdk.showAd()`)
- Rewarded ads (`gdsdk.preloadAd('rewarded')` + `gdsdk.showAd('rewarded')`)
- Display banners (`gdsdk.showAd('display', { containerId })`)
- Lifecycle pause/resume signals via `SDK_GAME_PAUSE` / `SDK_GAME_START` events

GameDistribution does not expose player/auth, platform storage (use `localStorage`), payments, leaderboards, achievements, social/share, remote config, or server-time APIs — those sections are intentionally omitted below. External-link navigation should also be avoided on this platform.

## Installation

Append the SDK script to the page after defining `window.GD_OPTIONS`. The script URL is fixed:

```js
const SDK_URL = 'https://html5.api.gamedistribution.com/main.min.js'

function loadScript(src) {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script')
        script.src = src
        script.onload = resolve
        script.onerror = reject
        document.head.appendChild(script)
    })
}
```

The SDK exposes itself as `window.gdsdk` once the `SDK_READY` event fires.

## Initialization

GameDistribution is configured by assigning `window.GD_OPTIONS` BEFORE the SDK script is appended. The only required field is `gameId` — your GameDistribution game GUID. All lifecycle and ad events arrive through the single `onEvent(event)` callback.

```js
const options = {
    gameId: 'YOUR_GAME_ID',
}

let platformSdk = null
let isInitialized = false
let currentAdvertisementIsRewarded = false

const listeners = {
    interstitialStateChanged: [],
    rewardedStateChanged: [],
    bannerStateChanged: [],
    pauseStateChanged: [],
    audioStateChanged: [],
}

function emit(name, payload) {
    listeners[name].forEach((cb) => cb(payload))
}

function initialize() {
    return new Promise((resolve, reject) => {
        if (isInitialized) {
            resolve()
            return
        }

        if (typeof options.gameId !== 'string') {
            reject(new Error('Game params are not found'))
            return
        }

        window.GD_OPTIONS = {
            gameId: options.gameId,
            onEvent(event) {
                switch (event.name) {
                    case 'SDK_READY':
                        platformSdk = window.gdsdk
                        isInitialized = true

                        // Optional: show an interstitial immediately once the SDK is ready
                        showInterstitial()
                        resolve()
                        break
                    case 'SDK_GAME_START':
                        // Ad finished — game should resume
                        if (currentAdvertisementIsRewarded) {
                            emit('rewardedStateChanged', 'closed')
                            // Preload the next rewarded ad right away
                            platformSdk.preloadAd('rewarded')
                        } else {
                            emit('interstitialStateChanged', 'closed')
                        }
                        emit('pauseStateChanged', false)
                        emit('audioStateChanged', true)
                        break
                    case 'SDK_GAME_PAUSE':
                        // Ad started — game should pause
                        if (currentAdvertisementIsRewarded) {
                            emit('rewardedStateChanged', 'opened')
                        } else {
                            emit('interstitialStateChanged', 'opened')
                        }
                        emit('pauseStateChanged', true)
                        emit('audioStateChanged', false)
                        break
                    case 'SDK_REWARDED_WATCH_COMPLETE':
                        // Reward must be granted here
                        emit('rewardedStateChanged', 'rewarded')
                        break
                    case 'SDK_GDPR_TRACKING':
                    case 'SDK_GDPR_TARGETING':
                    default:
                        break
                }
            },
        }

        loadScript(SDK_URL).catch(reject)
    })
}
```

Key event names emitted by the SDK through `onEvent`:

- `SDK_READY` — SDK is ready, `window.gdsdk` is available.
- `SDK_GAME_PAUSE` — an ad is starting; pause gameplay and mute audio.
- `SDK_GAME_START` — an ad has ended (or none was shown); resume gameplay and unmute audio.
- `SDK_REWARDED_WATCH_COMPLETE` — the user completed a rewarded ad; grant the reward.
- `SDK_GDPR_TRACKING`, `SDK_GDPR_TARGETING` — GDPR consent signals (no action is usually required in game code).

## Advertisement

GameDistribution supports interstitial, rewarded, and display banner ads. All ad methods on `gdsdk` return promises and reject when an ad cannot be served.

You must remember which ad type is currently in flight (interstitial vs rewarded) so the shared `SDK_GAME_PAUSE` / `SDK_GAME_START` events can be routed to the right state.

### Interstitial

```js
function showInterstitial() {
    currentAdvertisementIsRewarded = false

    if (!platformSdk) {
        emit('interstitialStateChanged', 'failed')
        return
    }

    platformSdk
        .showAd()
        .catch(() => {
            emit('interstitialStateChanged', 'failed')
        })
}
```

The lifecycle is:

1. `SDK_GAME_PAUSE` fires → state `opened`, pause game.
2. `SDK_GAME_START` fires → state `closed`, resume game.
3. If `showAd()` rejects → state `failed`.

GameDistribution does not enforce a minimum delay between interstitials at the SDK level — call `showAd()` whenever your game logic requires.

### Rewarded

Rewarded ads must be preloaded before they can be shown. After every rewarded ad completes, preload the next one.

```js
function preloadRewarded() {
    if (platformSdk) {
        platformSdk.preloadAd('rewarded')
    }
}

function showRewarded() {
    currentAdvertisementIsRewarded = true

    if (!platformSdk) {
        emit('rewardedStateChanged', 'failed')
        return
    }

    platformSdk
        .showAd('rewarded')
        .catch(() => {
            emit('rewardedStateChanged', 'failed')
        })
}
```

The lifecycle is:

1. `SDK_GAME_PAUSE` fires → state `opened`, pause game.
2. `SDK_REWARDED_WATCH_COMPLETE` fires → state `rewarded`, grant the reward.
3. `SDK_GAME_START` fires → state `closed`, resume game, then `preloadAd('rewarded')` for the next round.
4. If `showAd('rewarded')` rejects → state `failed`.

### Banner

Display banners are rendered into a DOM container whose id is passed to `gdsdk.showAd('display', { containerId })`. The example below uses the fixed id `'banner-container'`.

```js
const BANNER_CONTAINER_ID = 'banner-container'

function createBannerContainer(position) {
    const container = document.createElement('div')
    container.id = BANNER_CONTAINER_ID
    container.style.position = 'fixed'
    container.style.left = '0'
    container.style.right = '0'
    container.style.zIndex = '9999'

    if (position === 'top') {
        container.style.top = '0'
    } else {
        container.style.bottom = '0'
    }

    document.body.appendChild(container)
    return container
}

function showBanner(position) {
    let container = document.getElementById(BANNER_CONTAINER_ID)
    if (!container) {
        container = createBannerContainer(position)
    }

    container.style.display = 'block'

    platformSdk.showAd('display', { containerId: BANNER_CONTAINER_ID })
        .then(() => {
            emit('bannerStateChanged', 'shown')
        })
        .catch(() => {
            emit('bannerStateChanged', 'failed')
            container.style.display = 'none'
        })
}

function hideBanner() {
    const container = document.getElementById(BANNER_CONTAINER_ID)
    if (container) {
        container.style.display = 'none'
    }

    emit('bannerStateChanged', 'hidden')
}
```

`position` accepts `'top'` or `'bottom'`. Hiding a banner only toggles container visibility — it does not destroy the ad slot.

## Lifecycle / Audio / Pause

GameDistribution drives game pause/resume through the same `SDK_GAME_PAUSE` and `SDK_GAME_START` events that wrap every ad. There is no separate visibility or audio API — you derive both from these events:

- On `SDK_GAME_PAUSE`: pause gameplay, mute or duck audio. The SDK will not start an ad if the game is not paused.
- On `SDK_GAME_START`: resume gameplay, restore audio. This event also fires once after `SDK_READY` if no ad was served.

Combine this with the standard browser visibility events for tab-switch handling, since GameDistribution does not surface those itself:

```js
document.addEventListener('visibilitychange', () => {
    const isHidden = document.visibilityState === 'hidden'
    emit('pauseStateChanged', isHidden)
    emit('audioStateChanged', !isHidden)
})

window.addEventListener('blur', () => {
    emit('pauseStateChanged', true)
    emit('audioStateChanged', false)
})

window.addEventListener('focus', () => {
    emit('pauseStateChanged', false)
    emit('audioStateChanged', true)
})
```

## Storage

GameDistribution does not provide a platform storage API. Use `window.localStorage` directly for persisting player progress.

## Unsupported features

The following are NOT provided by the GameDistribution native SDK and are intentionally omitted:

- Player authentication / profile (no `playerId`, `playerName`, `playerPhotos`)
- In-app purchases / payments
- Leaderboards
- Achievements
- Social: invite friends, join community, share, create post, add-to-home-screen, add-to-favorites, rate
- Remote config
- Server time (use a public NTP-style endpoint such as `worldtimeapi.org` if needed)
- External link navigation (treat external links as not allowed on this platform)

## GameDistribution-specific notes

- `window.GD_OPTIONS` MUST be assigned before the SDK script is appended to the DOM, otherwise `gameId` and `onEvent` will be missed.
- The `onEvent` callback receives a single object with at minimum a `name` field — switch on `event.name`.
- `SDK_GAME_PAUSE` / `SDK_GAME_START` are reused across both interstitial and rewarded ads; track which ad type you requested last (e.g. a `currentAdvertisementIsRewarded` flag) to route state correctly.
- `showAd(...)` returns a Promise that rejects when the ad cannot be filled — always attach a `.catch()` to fall back gracefully.
- After every rewarded ad completes, call `gdsdk.preloadAd('rewarded')` again so the next request resolves quickly.
- The banner container id passed to `showAd('display', ...)` must match an element already attached to the DOM.
