---
name: poki-js-sdk
description: Use this skill to integrate the Poki JavaScript SDK into a JavaScript game.
---

# Poki JavaScript SDK

## Overview

Poki is an HTML5 game distribution platform. Its JavaScript SDK (`PokiSDK`) is loaded as a single script and exposes promise-based methods for initialization, ad breaks, loading state, and gameplay lifecycle reporting.

Supported features:
- Initialization with loading-finished signal
- Interstitial ads (`commercialBreak`)
- Rewarded ads (`rewardedBreak`)
- Gameplay lifecycle (`gameplayStart`, `gameplayStop`)

## Installation

Add the official Poki SDK script to your page:

```html
<script src='https://game-cdn.poki.com/scripts/v2/poki-sdk.js'></script>
```

You can also inject it dynamically:

```js
function loadPokiSdk() {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script')
        script.src = 'https://game-cdn.poki.com/scripts/v2/poki-sdk.js'
        script.onload = resolve
        script.onerror = reject
        document.head.appendChild(script)
    })
}
```

## Initialization

Wait for `window.PokiSDK` to expose `init`, then call it. After init resolves, mark loading as finished so Poki can dismiss its loader and start the session.

```js
loadPokiSdk()
    .then(() => window.PokiSDK.init())
    .then(() => {
        // SDK is ready
        window.PokiSDK.gameLoadingFinished()
    })
    .catch(() => {
        // SDK init failed - continue without Poki integration
        window.PokiSDK && window.PokiSDK.gameLoadingFinished()
    })
```

If your loader has discrete phases, you may bracket them with `gameLoadingStart` and `gameLoadingFinished`. Call `gameLoadingFinished` when the game is ready for the player.

## Advertisement

### Interstitial

`commercialBreak` accepts an optional callback fired immediately before the ad opens. The returned promise resolves once the ad flow ends. If the SDK could not deliver an ad, the callback is not invoked, so track this with a flag.

```js
function showInterstitial() {
    let isOpened = false

    window.PokiSDK.commercialBreak(() => {
        isOpened = true
        // Pause game, mute audio
    })
        .then(() => {
            if (isOpened) {
                // Resume game, unmute audio
            } else {
                // Ad was not shown
            }
        })
        .catch(() => {
            // Ad request failed
        })
}
```

### Rewarded

`rewardedBreak` mirrors `commercialBreak` but resolves with a boolean indicating whether the user earned the reward. Grant the reward only when the resolution value is truthy and the open callback fired.

```js
function showRewarded() {
    let isOpened = false

    window.PokiSDK.rewardedBreak(() => {
        isOpened = true
        // Pause game, mute audio
    })
        .then((success) => {
            if (isOpened) {
                if (success) {
                    // Grant reward to the player
                }
                // Resume game, unmute audio
            } else {
                // Ad was not shown
            }
        })
        .catch(() => {
            // Ad request failed
        })
}
```

## Lifecycle

Tell Poki when active gameplay starts and stops. Poki uses these signals to time ad breaks and to measure engagement. Call `gameplayStart` when the player gains control (level start, level resume) and `gameplayStop` when control is taken away (level paused, completed, failed, or returning to a menu).

```js
function onLevelStart() {
    window.PokiSDK.gameplayStart()
}

function onLevelEnd() {
    window.PokiSDK.gameplayStop()
}
```

Pair every `gameplayStart` with a matching `gameplayStop`.

---

Notes on omitted features: this skill does not cover server time, ad-block detection, `happyTime`, or `gameInteractive`; keep the implementation focused on the Poki SDK features listed above.
