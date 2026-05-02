---
name: youtube-playables-js-sdk
description: Use this skill to integrate the YouTube Playables JavaScript SDK into a JavaScript game.
---

# YouTube Playables JavaScript SDK

## Overview

YouTube Playables runs HTML5 games inside the YouTube client. The JavaScript SDK is exposed as a global `window.ytgame` object created by the YouTube host once the embedding script bootstraps. Every API documented here belongs to the YouTube Playables SDK surface used by HTML5 games.

Supported features:
- Initialization and game-ready signaling
- Persistent saved data via `ytgame.game.loadData` / `ytgame.game.saveData`
- Interstitial ads via `ytgame.ads.requestInterstitialAd`
- Rewarded ads via `ytgame.ads.requestRewardedAd`
- Leaderboard score submission via `ytgame.engagement.sendScore`
- Player language via `ytgame.system.getLanguage`
- Lifecycle hooks `ytgame.system.onPause` / `ytgame.system.onResume`
- Audio enablement hook `ytgame.system.onAudioEnabledChange`

## Installation

YouTube injects `ytgame` automatically when the game is loaded inside the YouTube Playables container. There is no public CDN URL to add to your HTML; the host page provides the global. The integration pattern is therefore to wait until `window.ytgame` is defined before using the SDK.

```js
// Poll until the host injects the global, then resolve with it
function waitForYtgame() {
    return new Promise((resolve) => {
        if (window.ytgame) {
            resolve(window.ytgame)
            return
        }

        const intervalId = setInterval(() => {
            if (window.ytgame) {
                clearInterval(intervalId)
                resolve(window.ytgame)
            }
        }, 100)
    })
}
```

## Initialization

Wait for `ytgame`, read the player language, load any previously saved data, attach lifecycle and audio listeners, then call `ytgame.game.firstFrameReady()` once your first frame has been rendered. When the game becomes fully interactive (assets loaded, menu visible, input accepted), call `ytgame.game.gameReady()`.

```js
let ytgame = null
let platformLanguage = 'en'
let savedData = {}

waitForYtgame().then((sdk) => {
    ytgame = sdk

    const getLanguagePromise = ytgame.system.getLanguage().then((language) => {
        // Normalize to a 2-letter code for common localization tables
        platformLanguage = language.length > 2 ? language.slice(0, 2) : language
    })

    const getDataPromise = ytgame.game.loadData().then((data) => {
        if (typeof data === 'string' && data !== '') {
            savedData = JSON.parse(data)
        }
    })

    ytgame.system.onAudioEnabledChange((isEnabled) => {
        // Mute or unmute the game audio
    })

    ytgame.system.onPause(() => {
        // Pause the game loop
    })

    ytgame.system.onResume(() => {
        // Resume the game loop
    })

    Promise.all([getLanguagePromise, getDataPromise]).finally(() => {
        // Tell YouTube the first frame has rendered
        ytgame.game.firstFrameReady()
    })
})
```

When your game is fully interactive, signal the host:

```js
function notifyGameReady() {
    ytgame.game.gameReady()
}
```

## Storage (saved data)

Saved data is a single JSON-serializable blob stored per-player by YouTube. The JavaScript SDK exposes `loadData()` and `saveData(stringValue)`. Keep an in-memory cache and merge per-key reads and writes into the single blob to support multiple keys without overwriting unrelated values.

```js
// Read one or many keys from saved data
function getDataFromStorage(key, tryParseJson) {
    return ytgame.game.loadData().then((data) => {
        if (typeof data === 'string' && data !== '') {
            savedData = JSON.parse(data)
        }

        const readOne = (k) => {
            let value = typeof savedData[k] === 'undefined' ? null : savedData[k]

            if (typeof value === 'string' && tryParseJson) {
                try {
                    value = JSON.parse(value)
                } catch (e) {
                    // Keep value as-is on parse error
                }
            }

            return value
        }

        if (Array.isArray(key)) {
            return key.map(readOne)
        }

        return readOne(key)
    })
}

// Write one or many keys, preserving the rest of the blob
function setDataToStorage(key, value) {
    const data = savedData !== null ? { ...savedData } : {}

    if (Array.isArray(key)) {
        for (let i = 0; i < key.length; i++) {
            data[key[i]] = value[i]
        }
    } else {
        data[key] = value
    }

    return ytgame.game.saveData(JSON.stringify(data)).then(() => {
        savedData = data
    })
}

// Delete one or many keys
function deleteDataFromStorage(key) {
    const data = savedData !== null ? { ...savedData } : {}

    if (Array.isArray(key)) {
        for (let i = 0; i < key.length; i++) {
            delete data[key[i]]
        }
    } else {
        delete data[key]
    }

    return ytgame.game.saveData(JSON.stringify(data)).then(() => {
        savedData = data
    })
}
```

## Advertisement

### Interstitial

`ytgame.ads.requestInterstitialAd()` returns a promise that resolves when the ad flow ends. There is no preload step — request the ad when you want it shown. Pause the game before the request and resume after the resolution.

```js
function showInterstitial() {
    // Pause game, mute audio
    ytgame.ads.requestInterstitialAd()
        .then(() => {
            // Resume game, unmute audio
        })
        .catch(() => {
            // Ad failed to deliver - resume game and continue
        })
}
```

### Rewarded

`ytgame.ads.requestRewardedAd(placement)` accepts a placement string and resolves with a boolean indicating whether the reward was earned. Grant the reward only when the resolved value is truthy.

```js
function showRewarded(placement) {
    // Pause game, mute audio
    ytgame.ads.requestRewardedAd(placement)
        .then((isRewardEarned) => {
            if (isRewardEarned) {
                // Grant reward to the player
            }
            // Resume game, unmute audio
        })
        .catch(() => {
            // Ad failed to deliver
        })
}
```

## Leaderboards

YouTube exposes a single native leaderboard per game. Submit integer scores via `ytgame.engagement.sendScore({ value })`. Coerce strings to integers before sending.

```js
function submitScore(score) {
    const value = typeof score === 'string' ? parseInt(score, 10) : score
    return ytgame.engagement.sendScore({ value })
}
```

## Lifecycle

The SDK exposes three key lifecycle signals:

- `ytgame.game.firstFrameReady()` — call once the first frame has rendered, after init promises settle.
- `ytgame.game.gameReady()` — call when the game becomes fully interactive.
- `ytgame.system.onPause(callback)` / `ytgame.system.onResume(callback)` — host-driven pause/resume.

```js
ytgame.system.onPause(() => {
    // Stop game loop, pause audio, store transient state
})

ytgame.system.onResume(() => {
    // Restart game loop, resume audio
})
```

## Audio

YouTube can mute or unmute games globally (for example when the user toggles the YouTube player volume). Subscribe with `ytgame.system.onAudioEnabledChange` and apply the new state to your audio system.

```js
ytgame.system.onAudioEnabledChange((isEnabled) => {
    if (isEnabled) {
        // Unmute all game audio
    } else {
        // Mute all game audio
    }
})
```

## Player language

`ytgame.system.getLanguage()` returns a promise resolving to a language tag. Slice it to a 2-letter code to match common content tables.

```js
ytgame.system.getLanguage().then((language) => {
    const code = language.length > 2 ? language.slice(0, 2) : language
    // Use code to localize UI
})
```

---

Notes on omitted features: the YouTube Playables SDK surface covered here does not include player identity, server time, achievements, payments, banners, share, rate, remote config, or ad-block detection. Server time should be sourced from your own backend if you need it. Health-event reporting via `ytgame.health.*` is out of scope for this skill.
