---
name: yandex-js-sdk
description: Use this skill to integrate the Yandex Games JavaScript SDK into a JavaScript game.
---

# Yandex Games JavaScript SDK

## Overview

Yandex Games loads games inside a frame on `yandex.<tld>/games`. The host injects the `YaGames` global once the SDK script is loaded; calling `YaGames.init()` returns a `ysdk` instance that exposes every supported feature:

- Initialization, environment (language / TLD), device info
- `LoadingAPI.ready()` and `GameplayAPI.start/stop` lifecycle hooks
- Player profile and authorization (`getPlayer`, `auth.openAuthDialog`)
- Per-player key/value storage (`player.getData` / `player.setData`)
- Fullscreen interstitial, rewarded video and sticky banner ads (`adv.*`)
- In-app purchases (`getPayments` → `purchase`, `getPurchases`, `consumePurchase`, `getCatalog`)
- Leaderboards (`leaderboards.setScore`, `leaderboards.getEntries`)
- Remote config flags (`getFlags`)
- Server time (`serverTime()`)
- Review prompt (`feedback.canReview` / `feedback.requestReview`)
- Add-to-home-screen shortcut (`shortcut.canShowPrompt` / `shortcut.showPrompt`)
- Clipboard write (`clipboard.writeText`)
- Catalog / lookup of other Yandex Games (`features.GamesAPI`)
- Pause / resume notifications via the `game_api_pause` and `game_api_resume` events

The SDK does NOT expose social share, invite-friends, community-join, add-to-favorites, achievements or stat counters in the surface used here, so those are intentionally omitted.

## Installation

Load the official SDK script before bootstrapping the game.

```html
<script src='https://yandex.ru/games/sdk/v2'></script>
```

The script registers the `YaGames` global asynchronously. If you load it dynamically, poll until `window.YaGames && window.YaGames.init` is defined before calling `init()`.

## Initialization

`YaGames.init()` returns a promise that resolves with the `ysdk` object. Once the loading screen is gone, call `features.LoadingAPI.ready()` so Yandex hides its splash.

```javascript
async function initYandex() {
    const ysdk = await window.YaGames.init()

    // Subscribe to host pause / resume early so events are not missed
    ysdk.on('game_api_pause', () => {
        // Mute audio and pause gameplay
    })
    ysdk.on('game_api_resume', () => {
        // Unmute audio and resume gameplay
    })

    // Tell Yandex the loading screen is gone
    ysdk.features.LoadingAPI?.ready()

    return ysdk
}
```

Useful values from `ysdk.environment` and `ysdk.deviceInfo`:

```javascript
function readEnvironment(ysdk) {
    return {
        language: ysdk.environment.i18n.lang.toLowerCase(),
        tld: ysdk.environment.i18n.tld.toLowerCase(),
        deviceType: ysdk.deviceInfo.type, // 'desktop' | 'mobile' | 'tablet' | 'tv'
    }
}
```

## Player / Authorization

`getPlayer({ signed })` returns a player object whose `isAuthorized()` flag tells you whether the user has a Yandex account session. If not, call `auth.openAuthDialog()` to prompt sign-in and re-fetch the player. Pass `signed: true` to receive a server-verifiable signature on the player payload.

```javascript
async function getPlayer(ysdk, { useSignedData = false } = {}) {
    const player = await ysdk.getPlayer({ signed: useSignedData })

    return {
        raw: player,
        id: player.getUniqueID(),
        name: player.getName() || null,
        photos: ['small', 'medium', 'large']
            .map((size) => player.getPhoto(size))
            .filter(Boolean),
        isAuthorized: player.isAuthorized(),
        payingStatus: player.getPayingStatus(),
        signature: useSignedData ? player.signature : null,
    }
}

async function authorizePlayer(ysdk, options) {
    const current = await getPlayer(ysdk, options)
    if (current.isAuthorized) {
        return current
    }

    await ysdk.auth.openAuthDialog()
    return getPlayer(ysdk, options)
}
```

## Storage

Yandex stores per-player JSON data via `player.getData` / `player.setData`. There is no per-key API: every call writes the full document. Cache the latest snapshot client-side and merge before each save. Storage is only available for authorized players; fall back to `localStorage` for guests.

```javascript
async function loadStorage(player) {
    return player.getData()
}

async function setStorage(player, cache, key, value) {
    const data = { ...(cache || {}) }
    if (Array.isArray(key)) {
        for (let i = 0; i < key.length; i++) {
            data[key[i]] = value[i]
        }
    } else {
        data[key] = value
    }
    await player.setData(data)
    return data
}

async function deleteStorage(player, cache, key) {
    const data = { ...(cache || {}) }
    if (Array.isArray(key)) {
        for (let i = 0; i < key.length; i++) {
            delete data[key[i]]
        }
    } else {
        delete data[key]
    }
    await player.setData(data)
    return data
}

function getCached(cache, key) {
    if (Array.isArray(key)) {
        return key.map((k) => (typeof cache?.[k] === 'undefined' ? null : cache[k]))
    }
    return typeof cache?.[key] === 'undefined' ? null : cache[key]
}
```

## Advertisement

### Fullscreen interstitial

```javascript
function showInterstitial(ysdk, handlers = {}) {
    ysdk.adv.showFullscreenAdv({
        callbacks: {
            onOpen: () => handlers.onOpen?.(),
            onClose: (wasShown) => handlers.onClose?.(wasShown),
            onError: (err) => handlers.onError?.(err),
        },
    })
}
```

`onClose(wasShown)` fires both for normal close and when the SDK skips the ad. Treat `wasShown === false` as a failure.

### Rewarded video

```javascript
function showRewarded(ysdk, handlers = {}) {
    ysdk.adv.showRewardedVideo({
        callbacks: {
            onOpen: () => handlers.onOpen?.(),
            onRewarded: () => handlers.onRewarded?.(),
            onClose: () => handlers.onClose?.(),
            onError: (err) => handlers.onError?.(err),
        },
    })
}
```

Grant the reward inside `onRewarded`. `onClose` fires after the ad UI is dismissed.

### Sticky banner

The banner is placed by Yandex; the game just toggles it. Inspect the result of every call to know whether the banner is actually on screen.

```javascript
async function getBannerStatus(ysdk) {
    return ysdk.adv.getBannerAdvStatus() // { stickyAdvIsShowing: boolean, reason?: string }
}

async function showBanner(ysdk) {
    const data = await ysdk.adv.showBannerAdv()
    return data.stickyAdvIsShowing === true
}

async function hideBanner(ysdk) {
    const data = await ysdk.adv.hideBannerAdv()
    return data.stickyAdvIsShowing === false
}
```

Yandex serves ads regardless of adblock state, so an explicit adblock check is unnecessary.

## Payments / IAP

Obtain the payments object once after init. Pass `signed: true` if your backend needs to validate purchase signatures.

```javascript
async function initPayments(ysdk, { signed = true } = {}) {
    return ysdk.getPayments({ signed })
}
```

### Catalog

```javascript
async function getCatalog(payments) {
    const products = await payments.getCatalog()
    return products.map((p) => ({
        id: p.id,
        title: p.title,
        description: p.description,
        imageURI: p.imageURI,
        price: p.price,
        priceValue: p.priceValue,
        priceCurrencyCode: p.priceCurrencyCode,
        priceCurrencyImage: p.getPriceCurrencyImage?.('medium'),
    }))
}
```

### Purchase

```javascript
async function purchase(payments, productId, payload) {
    const result = await payments.purchase({ id: productId, developerPayload: payload })
    // result.purchaseData = { productID, purchaseToken, developerPayload, signature, ... }
    return { id: productId, ...result.purchaseData }
}
```

### Owned purchases

```javascript
async function getPurchases(payments) {
    const list = await payments.getPurchases()
    return list.map((purchase) => ({ id: purchase.productID, ...purchase.purchaseData }))
}
```

### Consume

After the game has granted the in-game item, consume the purchase token so the player can buy the same SKU again.

```javascript
async function consumePurchase(payments, purchaseToken) {
    await payments.consumePurchase(purchaseToken)
}
```

## Leaderboards

Leaderboards are configured server-side in the developer console; the game references each board by its string id. Setting a score requires an authorized player.

```javascript
async function setLeaderboardScore(ysdk, leaderboardId, score) {
    const lb = await ysdk.getLeaderboards()
    await lb.setLeaderboardScore(leaderboardId, score)
}

async function getLeaderboardEntries(ysdk, leaderboardId, { isPlayerAuthorized = false } = {}) {
    const lb = await ysdk.getLeaderboards()

    const options = { quantityTop: 20 }
    if (isPlayerAuthorized) {
        options.includeUser = true
        options.quantityAround = 3
    }

    const result = await lb.getLeaderboardEntries(leaderboardId, options)
    if (!result || result.entries.length === 0) {
        return []
    }

    return result.entries.map((e) => ({
        id: e.player.uniqueID,
        name: e.player.publicName,
        score: e.score,
        rank: e.rank,
        photo: e.player.getAvatarSrc('large'),
    }))
}
```

## Remote config

`ysdk.getFlags(options)` returns a flat `{ key: stringValue }` map. Pass `clientFeatures` to deliver client-conditioned variants.

```javascript
async function getRemoteConfig(ysdk, options = {}) {
    const params = { clientFeatures: [], ...options }
    return ysdk.getFlags(params)
}
```

## Server time

```javascript
function getServerTime(ysdk) {
    // Returns Unix timestamp in milliseconds
    return ysdk.serverTime()
}
```

## Social

### Review prompt

`feedback.canReview()` returns `{ value: boolean, reason?: string }`. If `value` is true, call `requestReview()` and inspect `feedbackSent` to know whether the player actually submitted feedback.

```javascript
async function requestReview(ysdk) {
    const can = await ysdk.feedback.canReview()
    if (!can.value) {
        throw new Error(can.reason || 'review not available')
    }
    const { feedbackSent } = await ysdk.feedback.requestReview()
    return feedbackSent
}
```

### Add to home screen (shortcut)

```javascript
async function canAddToHomeScreen(ysdk) {
    const prompt = await ysdk.shortcut.canShowPrompt()
    return prompt.canShow === true
}

async function addToHomeScreen(ysdk) {
    const result = await ysdk.shortcut.showPrompt()
    return result.outcome === 'accepted'
}
```

## Clipboard

```javascript
async function clipboardWrite(ysdk, text) {
    await ysdk.clipboard.writeText(text)
}
```

## Gameplay lifecycle, pause and audio

Yandex requires the game to bracket every active gameplay session with `GameplayAPI.start()` / `GameplayAPI.stop()`. This drives ad pacing and analytics.

```javascript
function onGameplayStarted(ysdk) {
    // level_started, level_resumed, gameplay_started
    ysdk.features.GameplayAPI?.start()
}

function onGameplayStopped(ysdk) {
    // level_paused, level_completed, level_failed, gameplay_stopped
    ysdk.features.GameplayAPI?.stop()
}
```

Listen for host-driven pause / resume:

```javascript
function bindPauseHandlers(ysdk, { onPause, onResume }) {
    ysdk.on('game_api_pause', () => {
        // Yandex covered the game (e.g. another tab, system overlay) — mute audio and pause
        onPause?.()
    })
    ysdk.on('game_api_resume', () => {
        onResume?.()
    })
}
```

Visibility / focus changes that are not announced by Yandex (player switches browser tabs) should still be handled with the standard `document.visibilitychange`, `window.blur` and `window.focus` events on top of these SDK callbacks.

## Catalog of other Yandex Games (optional)

`features.GamesAPI` lets the game list or look up other Yandex Games (used for cross-promotion screens). It is gated on availability per game; check for the namespace before calling it.

```javascript
async function getAllGames(ysdk) {
    const { games } = await ysdk.features.GamesAPI.getAllGames()
    return games
}

async function getGameById(ysdk, gameId) {
    const { game, isAvailable } = await ysdk.features.GamesAPI.getGameByID(gameId)
    return { ...game, isAvailable }
}
```
