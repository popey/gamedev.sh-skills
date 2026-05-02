---
name: msn-js-sdk
description: Use this skill to integrate the MSN JavaScript SDK into a JavaScript game.
---

# MSN JavaScript SDK

## Overview

MSN Casual Games run on the Microsoft Start gaming surface and expose a single global named `window.$msstart` once the official `msstart` script has loaded. The SDK is Promise-based — every feature is invoked as `window.$msstart.<method>Async(...)` and resolves with the host response (or rejects on failure).

Supported features (mirrors what the msstart SDK actually exposes):

- Initialization by injecting the official `msstart` script and waiting for `$msstart`
- Player sign-in (`getSignedInUserAsync`, `signInAsync`) returning `playerId`, `playerDisplayName`, `userAccountType`
- Cloud save key/value storage (`cloudSave.getDataAsync`, `cloudSave.saveDataAsync`) scoped by `gameId`
- Display banners (`showDisplayAdsAsync` / `hideDisplayAdsAsync`) with `position:WxH` placement strings
- Interstitial and rewarded ads via the `loadAdsAsync` / `showAdsAsync` / `showAdsCompletedAsync` lifecycle
- Native sharing (`shareAsync`)
- Native leaderboards (`submitGameResultsAsync`)
- In-app purchases (`iap.getAllAddOnsAsync`, `iap.purchaseAsync`, `iap.consumeAsync`, `iap.getAllPurchasesAsync`)

Personal Microsoft Accounts are required for IAP — payments are only available when the signed-in user has `userAccountType === 'personal'`.

Not exposed by msstart and intentionally omitted: invite friends, join community, create post, add to home screen / favorites, rate, achievements, remote config, server time API, and dedicated lifecycle / pause / audio events. Use standard browser events (`document.visibilitychange`, `window.blur`, `window.focus`) for pause / audio control, and pause gameplay while ads are `OPENED`.

## Installation

There is no npm package — load the official `msstart` script tag from Microsoft's CDN and wait for the `$msstart` global before calling any API.

```html
<script src='https://assets.msn.com/staticsb/statics/latest/msstart-games-sdk/msstart-v1.0.0-rc.21.min.js'></script>
```

Or inject it dynamically:

```js
function addJavaScript(url) {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script')
        script.src = url
        script.onload = resolve
        script.onerror = reject
        document.head.appendChild(script)
    })
}

function waitFor(globalName, timeoutMs = 10000) {
    return new Promise((resolve, reject) => {
        const startedAt = Date.now()
        const tick = () => {
            if (window[globalName]) {
                resolve(window[globalName])
                return
            }
            if (Date.now() - startedAt > timeoutMs) {
                reject(new Error(`Timed out waiting for window.${globalName}`))
                return
            }
            window.setTimeout(tick, 50)
        }
        tick()
    })
}
```

## Initialization

Load the script, wait for `$msstart`, then probe the signed-in user. The result of `getSignedInUserAsync()` decides whether cloud save is usable from the start (only authorized players can read / write the platform store).

```js
const SDK_URL = 'https://assets.msn.com/staticsb/statics/latest/msstart-games-sdk/msstart-v1.0.0-rc.21.min.js'

const options = {
    gameId: 'YOUR_MSN_GAME_ID',
    // Optional product catalog used by IAP
    payments: [
        // { id: 'gem_pack_small', msn: { id: 'msstore-product-id' } }
    ],
}

let msstart = null
let isInitialized = false
let isPlayerAuthorized = false
let playerId = null
let playerName = null
let playerExtra = null
let isPaymentsSupported = false

function initialize() {
    if (isInitialized) {
        return Promise.resolve()
    }

    return addJavaScript(SDK_URL)
        .then(() => waitFor('$msstart'))
        .then(() => {
            msstart = window.$msstart
            return msstart.getSignedInUserAsync()
                .then((data) => updatePlayerInfo(data))
                .catch(() => updatePlayerInfo(null))
        })
        .then(() => {
            isInitialized = true
        })
}

function updatePlayerInfo(data) {
    if (data) {
        isPlayerAuthorized = true
        playerId = data.playerId
        playerName = data.playerDisplayName
        playerExtra = data
        // IAP is only available for personal Microsoft Accounts
        isPaymentsSupported = data.userAccountType.toLowerCase() === 'personal'
    } else {
        isPlayerAuthorized = false
        playerId = null
        playerName = null
        playerExtra = null
        isPaymentsSupported = false
    }
}
```

The minimum recommended delay before the first interstitial is `60` seconds after initialization — match that on your side to avoid showing an ad immediately on launch.

## Player

Sign-in is supported; if the user is already signed in, `getSignedInUserAsync` resolves during `initialize` and no further action is required. To prompt the sign-in UI on demand, call `signInAsync`.

```js
function authorizePlayer() {
    if (isPlayerAuthorized) {
        return Promise.resolve()
    }

    return msstart.signInAsync()
        .then((data) => {
            updatePlayerInfo(data)
        })
        .catch((error) => {
            updatePlayerInfo(null)
            throw error
        })
}
```

The signed-in user object contains at least `playerId`, `playerDisplayName`, and `userAccountType` (`'personal'` for consumer Microsoft Accounts, otherwise work / school).

## Storage

MSN Casual Games provides a per-game cloud save bucket accessed through `$msstart.cloudSave`. All operations require an authorized player and are scoped by your `gameId`. There is no per-key endpoint — you read the whole object once and write back a merged object on every save.

```js
let cachedData = null

function isCloudSaveAvailable() {
    return isPlayerAuthorized
}

function getStorageData(key) {
    if (!isPlayerAuthorized) {
        return Promise.reject()
    }

    const loadCache = cachedData
        ? Promise.resolve(cachedData)
        : msstart.cloudSave.getDataAsync({ gameId: options.gameId })
            .then((data) => {
                cachedData = data ?? {}
                return cachedData
            })

    return loadCache.then((data) => {
        if (Array.isArray(key)) {
            return key.map((k) => data[k])
        }
        return data[key]
    })
}

function setStorageData(key, value) {
    if (!isPlayerAuthorized) {
        return Promise.reject()
    }

    const data = cachedData ? { ...cachedData } : {}
    if (Array.isArray(key)) {
        for (let i = 0; i < key.length; i++) {
            data[key[i]] = value[i]
        }
    } else {
        data[key] = value
    }

    return msstart.cloudSave.saveDataAsync({ data, gameId: options.gameId })
        .then(() => {
            cachedData = data
        })
}

function deleteStorageData(key) {
    if (!isPlayerAuthorized) {
        return Promise.reject()
    }

    // Setting a key to null on msstart is the deletion pattern
    const payload = {}
    if (Array.isArray(key)) {
        key.forEach((k) => {
            payload[k] = null
            if (cachedData) {
                delete cachedData[k]
            }
        })
    } else {
        payload[key] = null
        if (cachedData) {
            delete cachedData[key]
        }
    }

    return msstart.cloudSave.saveDataAsync({ data: payload, gameId: options.gameId })
}
```

For unauthorized players, fall back to `window.localStorage` in your own code.

## Advertisement

### Display banners

Display banners are placed via `showDisplayAdsAsync(placements)` where each placement is a string of the form `'position:WIDTHxHEIGHT'`. Hide them all with `hideDisplayAdsAsync()`.

Allowed positions and sizes:

```js
const MSN_SIZES_BY_POSITION = {
    top: [[728, 90], [970, 250], [320, 50]],
    bottom: [[320, 50]],
    left: [[300, 250], [300, 600], [320, 50], [160, 600]],
    right: [[300, 250], [300, 600], [320, 50], [160, 600]],
    topleft: [[300, 250]],
    topright: [[300, 250]],
    bottomleft: [[300, 250]],
    bottomright: [[300, 250]],
}
```

Simple top / bottom banner:

```js
function showBanner(position) {
    let placement
    switch (position) {
        case 'top':
            placement = 'top:728x90'
            break
        case 'bottom':
        default:
            placement = 'bottom:320x50'
            break
    }

    return msstart.showDisplayAdsAsync([placement])
}

function hideBanner() {
    return msstart.hideDisplayAdsAsync()
}
```

Multiple banners at once (max 2 placements per call). Pick the largest size that fits the requested rectangle:

```js
function parseDimension(value, screenSize) {
    if (typeof value !== 'string') {
        return null
    }
    if (value.endsWith('%')) {
        const percent = parseFloat(value)
        if (Number.isNaN(percent)) {
            return null
        }
        return Math.floor((screenSize * percent) / 100)
    }
    const px = parseInt(value, 10)
    return Number.isNaN(px) ? null : px
}

function resolvePosition(banner) {
    const hasTop = banner.top !== undefined
    const hasBottom = banner.bottom !== undefined
    const hasLeft = banner.left !== undefined
    const hasRight = banner.right !== undefined

    if (hasTop && hasLeft) return 'topleft'
    if (hasTop && hasRight) return 'topright'
    if (hasBottom && hasLeft) return 'bottomleft'
    if (hasBottom && hasRight) return 'bottomright'
    if (hasTop) return 'top'
    if (hasBottom) return 'bottom'
    if (hasLeft) return 'left'
    if (hasRight) return 'right'
    return null
}

function bannerToPlacement(banner) {
    const position = resolvePosition(banner)
    if (!position) return null

    const sizes = MSN_SIZES_BY_POSITION[position]
    if (!sizes || sizes.length === 0) return null

    const width = parseDimension(banner.width, window.innerWidth)
    const height = parseDimension(banner.height, window.innerHeight)
    if (width === null || height === null) return null

    const fitting = sizes.filter((s) => s[0] <= width && s[1] <= height)
    if (fitting.length === 0) return null

    let best = fitting[0]
    let bestArea = best[0] * best[1]
    fitting.forEach((s) => {
        const area = s[0] * s[1]
        if (area > bestArea) {
            best = s
            bestArea = area
        }
    })

    return `${position}:${best[0]}x${best[1]}`
}

function showAdvancedBanners(banners) {
    const placements = [...new Set(
        banners.map(bannerToPlacement).filter(Boolean),
    )].slice(0, 2)

    if (placements.length === 0) {
        return Promise.reject(new Error('No valid placements'))
    }

    return msstart.showDisplayAdsAsync(placements)
}

function hideAdvancedBanners() {
    return msstart.hideDisplayAdsAsync()
}
```

### Interstitial and rewarded ads

Interstitials and rewarded ads share one lifecycle: `loadAdsAsync(isRewarded)` returns an `adInstance`, then `showAdsAsync(adInstance.instanceId)` displays it and resolves with the instance again, whose `showAdsCompletedAsync` Promise settles when the user closes the ad.

Pass `false` for interstitial, `true` for rewarded.

```js
function showInterstitial({ onOpened, onClosed, onFailed } = {}) {
    return msstart.loadAdsAsync(false)
        .then((adInstance) => msstart.showAdsAsync(adInstance.instanceId))
        .then((adInstance) => {
            // Pause game audio and gameplay here
            if (onOpened) onOpened()
            return adInstance.showAdsCompletedAsync
        })
        .then(() => {
            // Resume game
            if (onClosed) onClosed()
        })
        .catch((error) => {
            if (onFailed) onFailed(error)
        })
}

function showRewarded({ onOpened, onRewarded, onClosed, onFailed } = {}) {
    return msstart.loadAdsAsync(true)
        .then((adInstance) => msstart.showAdsAsync(adInstance.instanceId))
        .then((adInstance) => {
            if (onOpened) onOpened()
            return adInstance.showAdsCompletedAsync
        })
        .then(() => {
            // showAdsCompletedAsync resolving for a rewarded ad means the reward was earned
            if (onRewarded) onRewarded()
            if (onClosed) onClosed()
        })
        .catch((error) => {
            if (onFailed) onFailed(error)
        })
}
```

Interstitial states to drive in your own state machine: `'loading'`, `'opened'`, `'closed'`, `'failed'`.
Rewarded states: `'loading'`, `'opened'`, `'closed'`, `'failed'`, `'rewarded'`.

## Social — Share

```js
function share(shareOptions) {
    // shareOptions are passed straight through to the msstart SDK
    return msstart.shareAsync(shareOptions)
}
```

Refer to msstart's documentation for the exact `shareOptions` shape (typically title, text, url).

## Leaderboards

MSN exposes a native leaderboard write-only API via `submitGameResultsAsync(score)`. There is no client-side read-back endpoint — leaderboards are rendered by the host UI.

```js
function leaderboardsSetScore(score) {
    return msstart.submitGameResultsAsync(score)
}
```

## Payments / In-App Purchases

IAP is only available when the signed-in user has `userAccountType === 'personal'`. All four standard operations are exposed under `$msstart.iap`. Map your local product ids to msstart product ids in your own catalog and merge the host response back with your local id.

Failure responses are reported in-band: each call may resolve with `{ code: 'IAP_*_FAILURE', description }` instead of throwing. Always check the `code` field.

### Get catalog

```js
function paymentsGetCatalog() {
    if (!options.payments) {
        return Promise.reject()
    }

    return msstart.iap.getAllAddOnsAsync({ productId: options.gameId })
        .then((msnProducts) => {
            if (msnProducts.code === 'IAP_GET_ALL_ADD_ONS_FAILURE') {
                throw new Error(msnProducts.description)
            }

            return options.payments.map((product) => {
                const platformId = product.msn?.id ?? product.id
                const msnProduct = msnProducts.find((p) => p.productId === platformId)

                return {
                    id: product.id,
                    title: msnProduct.title,
                    description: msnProduct.description,
                    publisherName: msnProduct.publisherName,
                    inAppOfferToken: msnProduct.inAppOfferToken,
                    isConsumable: msnProduct.isConsumable,
                    price: `${msnProduct.price.listPrice} ${msnProduct.price.currencyCode} `,
                    priceCurrencyCode: msnProduct.price.currencyCode,
                    priceValue: msnProduct.price.listPrice,
                }
            })
        })
}
```

### Purchase

The host returns `{ receipt, receiptSignature }` (or `{ code: 'IAP_PURCHASE_FAILURE', description }`). Cache the merged purchase so it can be matched later in `consume`.

```js
const purchases = []

async function paymentsPurchase(id) {
    const product = options.payments.find((p) => p.id === id)
    if (!product) {
        return Promise.reject()
    }

    if (!isPlayerAuthorized) {
        await authorizePlayer()
    }

    const platformProductId = product.msn?.id ?? product.id
    const purchase = await msstart.iap.purchaseAsync({ productId: platformProductId })

    if (purchase.code === 'IAP_PURCHASE_FAILURE') {
        throw new Error(purchase.description)
    }

    const merged = {
        id,
        ...purchase.receipt,
        receiptSignature: purchase.receiptSignature,
    }
    purchases.push(merged)
    return merged
}
```

### Consume purchase

Consume by msstart `productId` (taken from the cached purchase, not your local id). The host returns `{ consumptionReceipt, consumptionSignature }` or `{ code: 'IAP_CONSUME_FAILURE', description }`.

```js
async function paymentsConsumePurchase(id) {
    const idx = purchases.findIndex((p) => p.id === id)
    if (idx < 0) {
        return Promise.reject()
    }

    if (!isPlayerAuthorized) {
        await authorizePlayer()
    }

    const response = await msstart.iap.consumeAsync({
        productId: purchases[idx].productId,
    })

    if (response.code === 'IAP_CONSUME_FAILURE') {
        throw new Error(response.description)
    }

    purchases.splice(idx, 1)
    const result = {
        id,
        ...response.consumptionReceipt,
        consumptionSignature: response.consumptionSignature,
    }
    delete result.productId
    return result
}
```

### Get purchases

`iap.getAllPurchasesAsync({ productId })` returns `{ receipts, receiptSignature }` for unconsumed purchases. The error `{ code: 'IAP_GET_ALL_PURCHASES_FAILURE' }` should be treated as an empty list.

```js
async function paymentsGetPurchases() {
    if (!isPlayerAuthorized) {
        await authorizePlayer()
    }

    try {
        const response = await msstart.iap.getAllPurchasesAsync({
            productId: options.gameId,
        })

        purchases.length = 0
        response.receipts.forEach((purchase) => {
            const product = options.payments.find(
                (p) => (p.msn?.id ?? p.id) === purchase.productId,
            )
            purchases.push({
                id: product ? product.id : purchase.productId,
                ...purchase,
                receiptSignature: response.receiptSignature,
            })
        })

        return purchases.slice()
    } catch (error) {
        if (error && error.code === 'IAP_GET_ALL_PURCHASES_FAILURE') {
            return []
        }
        throw error
    }
}
```

## Lifecycle / Visibility / Audio / Pause

The msstart SDK does not push pause / audio / visibility events to the game. Use the standard browser surfaces and gate gameplay on them, plus on the interstitial / rewarded states above.

```js
document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden') {
        // Pause gameplay and mute audio
    } else {
        // Resume
    }
})

window.addEventListener('blur', () => {
    // Pause gameplay and mute audio
})

window.addEventListener('focus', () => {
    // Resume
})
```

## Server time

The msstart SDK does not expose a server-time endpoint. Use `Date.now()` for non-authoritative timing; for authoritative timing call your own backend.

## MSN-specific notes

- The single global is `window.$msstart`; everything is namespaced under it (`cloudSave.*`, `iap.*`, `loadAdsAsync`, `showAdsAsync`, `showDisplayAdsAsync`, `shareAsync`, `submitGameResultsAsync`, `getSignedInUserAsync`, `signInAsync`).
- Pin the SDK version in your script tag; MSN ships release-candidate builds at versioned URLs such as `msstart-v1.0.0-rc.21.min.js`.
- IAP is gated on `userAccountType === 'personal'` — work / school accounts cannot purchase add-ons.
- Banner placements are strings of the form `'position:WxH'`. Allowed sizes per position are fixed (see `MSN_SIZES_BY_POSITION`); requesting an unsupported size will fail. `showDisplayAdsAsync` accepts an array of placements but at most two simultaneous placements are recommended.
- Interstitial / rewarded share the `loadAdsAsync(isRewarded)` -> `showAdsAsync(instanceId)` -> `showAdsCompletedAsync` flow. For rewarded ads, `showAdsCompletedAsync` resolving means the reward is earned.
- Cloud save is a single per-game JSON object scoped by `gameId`. Cache it on first read and write the merged object back on every save. Deletion is performed by writing `null` for the deleted keys.
- IAP failure responses are returned as `{ code, description }` resolutions, not as rejections — always inspect `code` before treating a response as success.
- Wait for `window.$msstart` to be defined before calling any API; the script load completing does not guarantee the global is ready.
