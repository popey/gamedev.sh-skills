---
name: y8-js-sdk
description: Use this skill to integrate the Y8 JavaScript SDK into a JavaScript game.
---

# Y8 JavaScript SDK

Y8 hosts HTML5 games and exposes a global `window.ID` after its loader script finishes. The Y8 SDK wraps the player session, achievement / leaderboard `GameAPI`, and a generic `api()` HTTP helper. Advertising is delegated to Google AdSense via `adsbygoogle.push` (the SDK does not ship an ads module of its own).

## Overview

The supported surface area is:

- Initialization handshake (`ID.init({ appId })`) and `id.init` event subscription
- Player authorization (`ID.getLoginStatus`, `ID.login`)
- Platform-internal cloud storage built on `ID.api('user_data/...', 'POST', ...)`
- Interstitial and rewarded ads driven by Google AdSense `adsbygoogle.push`
- Native achievements: unlock (`ID.GameAPI.Achievements.save`), list (`listCustom`), and native popup (`list`)
- In-game leaderboards: submit (`ID.GameAPI.Leaderboards.save`) and fetch entries (`listCustom`)

Banners, in-app purchases, social actions (share / invite / community / rate / add-to-home-screen / add-to-favorites), remote config and a server-time API are not exposed by Y8 and are intentionally omitted here.

## Installation

The Y8 SDK is loaded from a single script URL. Inject it dynamically and wait for `window.ID` to appear before using it.

```javascript
const SDK_URL = 'https://cdn.y8.com/api/sdk.js'

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

Y8's bootstrap is event-driven: subscribe to `id.init` first, then call `ID.init({ appId })`. The `appId` (a.k.a. Game API Key) is issued by Y8 when you register the game. Once `id.init` fires, fetch the current login status to populate the player profile and load the Google AdSense ad-break runtime in parallel.

```javascript
async function initY8({ gameId, adSenseId, channelId }) {
    if (!gameId) {
        throw new Error('gameId (Y8 appId) is required')
    }

    await loadScript(SDK_URL)
    const sdk = await waitFor('ID')

    return new Promise((resolve, reject) => {
        sdk.Event.subscribe('id.init', () => {
            // Bootstrap Google AdSense alongside the Y8 session
            loadAdsByGoogle({ adSenseId, channelId })
                .then((showAd) => {
                    sdk.getLoginStatus((data) => {
                        const profile = readPlayerInfo(data)
                        resolve({ sdk, showAd, profile })
                    })
                })
                .catch(reject)
        })

        sdk.init({
            appId: gameId,
        })
    })
}
```

`Y8` recommends a 60-second initial delay before showing the first interstitial — track this in your game logic if you gate ads on session age.

## Player / Authorization

`ID.getLoginStatus(callback)` returns the current session without prompting. `ID.login(callback)` opens Y8's login dialog. Both deliver the same payload shape — a `status` of `'ok'` means the player is signed in and `authResponse.details` holds the profile.

```javascript
function readPlayerInfo(data) {
    const profile = {
        isAuthorized: false,
        id: null,
        name: null,
        locale: null,
        photos: [],
    }

    if (!data || data.status !== 'ok') {
        return profile
    }

    const details = data.authResponse.details
    const {
        pid,
        locale,
        nickname,
        first_name: firstName,
        last_name: lastName,
        avatars,
    } = details

    profile.isAuthorized = true
    profile.id = pid || null
    profile.locale = locale || null
    profile.name = [firstName, lastName].filter((x) => !!x).join(' ') || nickname || null

    const {
        thumb_url: photoSmall,
        medium_url: photoMedium,
        large_url: photoLarge,
    } = avatars || {}

    if (photoSmall) profile.photos.push(photoSmall)
    if (photoMedium) profile.photos.push(photoMedium)
    if (photoLarge) profile.photos.push(photoLarge)

    return profile
}

function authorizePlayer(sdk) {
    return new Promise((resolve, reject) => {
        sdk.login((response) => {
            const profile = readPlayerInfo(response)
            if (response && response.status === 'ok') {
                resolve(profile)
            } else {
                reject(new Error('Y8 login was cancelled or failed'))
            }
        })
    })
}
```

## Storage

Y8 provides cloud-backed user data through the generic `ID.api()` helper. A practical pattern is to store all game data as a single JSON blob under one server key (`'userData'`) — read it once, mutate the object locally, and write it back. Storage is only available while the player is authorized.

```javascript
const USERDATA_KEY = 'userData'
const NOT_FOUND_ERROR = 'Key not found'

let cachedUserData = null

function loadUserData(sdk) {
    return new Promise((resolve, reject) => {
        if (cachedUserData) {
            resolve(cachedUserData)
            return
        }

        sdk.api('user_data/retrieve', 'POST', { key: USERDATA_KEY }, (response) => {
            if (response.error && response.error !== NOT_FOUND_ERROR) {
                reject(response)
                return
            }

            let userData = {}
            try {
                if (response.jsondata) {
                    userData = JSON.parse(response.jsondata)
                }
            } catch (e) {
                // Leave userData as an empty object on parse failure
            }

            cachedUserData = userData
            resolve(userData)
        })
    })
}

function saveUserData(sdk, userData) {
    return new Promise((resolve, reject) => {
        const payload = { key: USERDATA_KEY, value: JSON.stringify(userData) }
        sdk.api('user_data/submit', 'POST', payload, (response) => {
            if (response.status === 'ok') {
                cachedUserData = userData
                resolve()
            } else {
                reject(response)
            }
        })
    })
}

function getStorageValue(sdk, key) {
    return loadUserData(sdk).then((userData) => userData[key] ?? null)
}

function setStorageValue(sdk, key, value) {
    return loadUserData(sdk).then((userData) => {
        const next = { ...userData, [key]: value }
        return saveUserData(sdk, next)
    })
}

function deleteStorageValue(sdk, key) {
    return loadUserData(sdk).then((userData) => {
        const next = { ...userData }
        delete next[key]
        return saveUserData(sdk, next)
    })
}
```

Invalidate `cachedUserData` (set it back to `null`) after a successful `ID.login` call so a freshly authorized player sees their own cloud state instead of the previous session's data.

## Advertisement

Y8 monetizes through Google AdSense for Games. The `adsbygoogle.push` runtime exposes an `adBreak` API; you call it with a `type` (`'start'` for interstitials, `'reward'` for rewarded) and a set of lifecycle callbacks. Use `data-ad-channel` if Y8 gave you a channel id, otherwise fall back to your own AdSense client id.

```javascript
const ADS_ID = '6129580795478709'

function loadAdsByGoogle({ adSenseId, channelId }) {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script')
        script.src = 'https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js'
        script.setAttribute('crossorigin', 'anonymous')
        script.setAttribute('data-ad-frequency-hint', '180s')

        // Y8 channel id takes precedence; falls back to host id
        script.setAttribute('data-ad-client', channelId ? `ca-pub-${ADS_ID}` : adSenseId)
        if (channelId) {
            script.setAttribute('data-ad-channel', channelId)
        } else {
            script.setAttribute('data-ad-host', `ca-host-pub-${ADS_ID}`)
        }

        script.addEventListener('load', () => {
            window.adsbygoogle = window.adsbygoogle || []
            window.adsbygoogle.push({
                preloadAdBreaks: 'on',
                sound: 'on',
                onReady: () => {},
            })
            resolve((adOptions) => window.adsbygoogle.push(adOptions))
        })

        script.addEventListener('error', () => reject(new Error('adsbygoogle failed to load')))
        document.head.appendChild(script)
    })
}
```

### Interstitial

```javascript
function showInterstitial(showAd, { onOpened, onClosed, onFailed } = {}) {
    if (!showAd) {
        onFailed?.('ads_not_ready')
        return
    }

    showAd({
        type: 'start',
        name: 'start-game',
        beforeAd: () => {
            onOpened?.()
        },
        afterAd: () => {
            onClosed?.()
        },
        adBreakDone: (placementInfo) => {
            if (placementInfo.breakStatus !== 'viewed') {
                onFailed?.(placementInfo.breakStatus)
            }
        },
    })
}
```

### Rewarded

Rewarded breaks are gated by `beforeReward(showAdFn)` — call `showAdFn(0)` to display the ad. `adViewed` fires when the player completed the ad and earned the reward; `adDismissed` means they bailed out early.

```javascript
function showRewarded(showAd, { onOpened, onRewarded, onClosed, onFailed } = {}) {
    if (!showAd) {
        onFailed?.('ads_not_ready')
        return
    }

    showAd({
        type: 'reward',
        name: 'rewarded Ad',
        beforeAd: () => {
            onOpened?.()
        },
        afterAd: () => {
            onClosed?.()
        },
        beforeReward: (showAdFn) => {
            showAdFn(0)
        },
        adDismissed: () => {
            onFailed?.('dismissed')
        },
        adViewed: () => {
            onRewarded?.()
        },
        adBreakDone: (placementInfo) => {
            if (placementInfo.breakStatus === 'frequencyCapped' || placementInfo.breakStatus === 'other') {
                onFailed?.(placementInfo.breakStatus)
            }
        },
    })
}
```

## Achievements

Y8 native achievements live under `ID.GameAPI.Achievements`. The player must be authorized for `save` to succeed; both `achievement` (display name) and `achievementkey` (stable id) are required by the SDK.

### Unlock

```javascript
function unlockAchievement(sdk, options) {
    if (!options || !options.achievement || !options.achievementkey) {
        return Promise.reject(new Error('achievement and achievementkey are required'))
    }

    return new Promise((resolve) => {
        sdk.GameAPI.Achievements.save(options, (data) => {
            resolve(data)
        })
    })
}
```

### List achievements (custom query)

`listCustom` returns achievements with a nested `player` object. Flatten it for easier consumption.

```javascript
function getAchievementsList(sdk, options = {}) {
    return new Promise((resolve, reject) => {
        sdk.GameAPI.Achievements.listCustom(options, (data) => {
            if (!data.success) {
                reject(new Error(data.errorcode))
                return
            }

            const list = data.achievements.map(({ player, ...achievement }) => ({
                ...achievement,
                playerid: player.playerid,
                playername: player.playername,
                lastupdated: player.lastupdated,
                date: player.date,
                rdate: player.rdate,
            }))
            resolve(list)
        })
    })
}
```

### Native popup

`ID.GameAPI.Achievements.list(options)` opens Y8's built-in achievements UI. It returns nothing — treat the call itself as the action.

```javascript
function showAchievementsPopup(sdk, options = {}) {
    sdk.GameAPI.Achievements.list(options)
}
```

## Leaderboards

Y8 leaderboards are in-game (the host site does not render a popup). Each board is identified by a string `table` id you configure in the Y8 dashboard. The player must be authorized to submit or read entries.

### Submit a score

```javascript
function submitScore(sdk, tableId, score) {
    return new Promise((resolve, reject) => {
        const options = {
            table: tableId,
            points: score,
        }

        sdk.GameAPI.Leaderboards.save(options, ({ success, errormessage }) => {
            if (success) {
                resolve()
            } else {
                reject(new Error(errormessage || 'leaderboard_save_failed'))
            }
        })
    })
}
```

### Fetch entries

```javascript
function getLeaderboardEntries(sdk, tableId) {
    return new Promise((resolve, reject) => {
        const options = {
            table: tableId,
            mode: 'alltime',
        }

        sdk.GameAPI.Leaderboards.listCustom(options, ({ scores, success, errormessage }) => {
            if (!success) {
                reject(new Error(errormessage || 'leaderboard_list_failed'))
                return
            }

            const entries = scores.map((entry) => ({
                id: entry.playerid,
                name: entry.playername,
                score: entry.points,
                rank: entry.rank,
                photo: null,
            }))
            resolve(entries)
        })
    })
}
```

Other valid `mode` values include `'daily'`, `'weekly'`, and `'monthly'` if you need scoped boards — `'alltime'` is the default used in the example above.