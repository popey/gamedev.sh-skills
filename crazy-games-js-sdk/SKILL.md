---
name: crazy-games-js-sdk
description: Use this skill to integrate the CrazyGames JavaScript SDK into a JavaScript game.
---

# CrazyGames JavaScript SDK

## Overview

CrazyGames exposes its SDK as a global object `window.CrazyGames.SDK` after a single script tag is loaded. The SDK covers:

- Initialization and language/device detection
- User account (display name, avatar, JWT)
- Persistent key/value storage (`SDK.data`)
- Interstitial (`midgame`) and rewarded ads with adblock detection
- Responsive banner ads (single and multi-slot)
- Gameplay lifecycle hooks (`gameplayStart` / `gameplayStop`, `loadingStart` / `loadingStop`, `happytime`)
- Analytics order tracking (used together with Xsolla payments)

In-app purchases are NOT a native CrazyGames feature; they are implemented through the Xsolla Pay Station widget on top of `SDK.user.getXsollaUserToken()`. Document them only if your project has an Xsolla project ID provisioned.

CrazyGames does not expose leaderboards, achievements, remote config, server time or social-share APIs in the SDK surface used here, so those sections are intentionally omitted.

## Installation

Load the SDK from the official CDN before bootstrapping the game.

```html
<script src='https://sdk.crazygames.com/crazygames-sdk-v3.js'></script>
```

If you load it dynamically, wait until `window.CrazyGames.SDK.init` is defined before calling it.

## Initialization

Call `SDK.init()` once. It resolves when the SDK is ready to be used. Only after that is `SDK.user`, `SDK.data`, `SDK.ad`, `SDK.banner`, `SDK.game` and `SDK.analytics` safe to call.

```javascript
async function initCrazyGames() {
    const sdk = window.CrazyGames.SDK
    await sdk.init()

    // SDK.user is available immediately after init
    const isUserAccountAvailable = sdk.user.isUserAccountAvailable

    // System info (only meaningful when an account is available)
    if (isUserAccountAvailable) {
        const language = sdk.user.systemInfo.countryCode.toLowerCase()
        const deviceType = sdk.user.systemInfo.device.type.toLowerCase() // 'desktop' | 'mobile' | 'tablet'
        console.log('lang/device', language, deviceType)
    }

    return sdk
}
```

## Player / Authorization

The SDK distinguishes between "user account available" (the player has a CrazyGames account on this domain) and "currently signed in". Use `showAuthPrompt()` to ask the player to sign in, then `getUser()` to read profile data and optionally `getUserToken()` to obtain a signed JWT for your backend.

```javascript
async function authorizePlayer(sdk, { useUserToken = false } = {}) {
    if (!sdk.user.isUserAccountAvailable) {
        // Guest session — no auth possible on this domain
        return null
    }

    await sdk.user.showAuthPrompt()

    const user = await sdk.user.getUser()
    if (!user) {
        return null
    }

    const profile = {
        name: user.username || null,
        photo: user.profilePictureUrl || null,
        extra: user,
    }

    if (useUserToken) {
        profile.extra.jwt = await sdk.user.getUserToken()
    }

    return profile
}
```

## Storage

CrazyGames provides synchronous key/value storage via `SDK.data`. Values are strings; serialize JSON yourself.

```javascript
function storageSet(sdk, key, value) {
    const data = typeof value === 'string' ? value : JSON.stringify(value)
    sdk.data.setItem(key, data)
}

function storageGet(sdk, key, tryParseJson = true) {
    const raw = sdk.data.getItem(key)
    if (!tryParseJson || raw == null) {
        return raw
    }
    try {
        return JSON.parse(raw)
    } catch (_) {
        return raw
    }
}

function storageRemove(sdk, key) {
    sdk.data.removeItem(key)
}

// Batch helpers — just iterate, the SDK has no native multi-key API
function storageSetMany(sdk, keys, values) {
    for (let i = 0; i < keys.length; i++) {
        storageSet(sdk, keys[i], values[i])
    }
}
```

## Advertisement

### Interstitial ad (`midgame`)

Call `SDK.ad.requestAd('midgame', callbacks)`. The callbacks fire synchronously as the ad lifecycle progresses.

```javascript
function showInterstitial(sdk) {
    sdk.ad.requestAd('midgame', {
        adStarted: () => {
            // Mute audio and pause the game here
        },
        adFinished: () => {
            // Resume audio and gameplay here
        },
        adError: (error) => {
            console.warn('interstitial failed', error)
        },
    })
}
```

### Rewarded ad

Same API, just pass `'rewarded'` as the type. `adFinished` means the user watched the ad to completion and should receive the reward.

```javascript
function showRewarded(sdk, onReward) {
    sdk.ad.requestAd('rewarded', {
        adStarted: () => {
            // Pause the game
        },
        adFinished: () => {
            onReward()
        },
        adError: (error) => {
            console.warn('rewarded failed', error)
        },
    })
}
```

### Adblock detection

```javascript
async function isAdblockActive(sdk) {
    return sdk.ad.hasAdblock()
}
```

### Responsive banners

Banners are bound to existing DOM containers by id. CrazyGames renders into them and picks an appropriate size on its own.

```javascript
async function showBanner(sdk, containerId) {
    // The container must already be in the DOM and visible
    try {
        await sdk.banner.requestResponsiveBanner([containerId])
    } catch (e) {
        console.warn('banner failed', e)
    }
}

async function showAdvancedBanners(sdk, containerIds) {
    // Request several banners in parallel
    await Promise.all(containerIds.map((id) => sdk.banner.requestResponsiveBanner(id)))
}

function hideBanners(sdk) {
    sdk.banner.clearAllBanners()
}
```

## Payments (optional, via Xsolla Pay Station)

CrazyGames itself does not sell IAPs. The integration uses CrazyGames-issued Xsolla user tokens with the Xsolla Pay Station widget and Xsolla Store API. Skip this section entirely if you do not have an Xsolla project ID.

Endpoints and assets used:

```javascript
const XSOLLA_PAYSTATION_EMBED_URL = 'https://cdn.xsolla.net/payments-bucket-prod/embed/1.5.0/widget.min.js'
const XSOLLA_SDK_URL = 'https://store.xsolla.com/api/v2/project'
```

Helpers:

```javascript
async function ensurePaystationLoaded() {
    if (window.XPayStationWidget) {
        return
    }
    await new Promise((resolve, reject) => {
        const script = document.createElement('script')
        script.src = XSOLLA_PAYSTATION_EMBED_URL
        script.onload = resolve
        script.onerror = reject
        document.head.appendChild(script)
    })
}

async function getXsollaToken(sdk) {
    return sdk.user.getXsollaUserToken()
}

async function getXsollaOrder(projectId, orderId, token) {
    const res = await fetch(
        `${XSOLLA_SDK_URL}/${projectId}/order/${orderId}`,
        { headers: { Authorization: `Bearer ${token}` } },
    )
    if (!res.ok) {
        throw new Error(`Xsolla order HTTP ${res.status}`)
    }
    return res.json()
}
```

### Catalog

```javascript
async function getCatalog(sdk, projectId) {
    const token = await getXsollaToken(sdk)
    const res = await fetch(
        `${XSOLLA_SDK_URL}/${projectId}/items?limit=50`,
        { headers: { Authorization: `Bearer ${token}` } },
    )
    if (!res.ok) {
        throw new Error(`Xsolla catalog HTTP ${res.status}`)
    }
    const data = await res.json()
    return (data.items || []).map((it) => ({
        id: it.sku,
        name: it.name,
        description: it.description,
        price: it.price?.amount ?? null,
        priceCurrencyCode: it.price?.currency ?? null,
    }))
}
```

### Purchase

```javascript
async function purchase(sdk, projectId, sku, { isSandbox = false } = {}) {
    await ensurePaystationLoaded()
    const userToken = await getXsollaToken(sdk)

    // Create the order on Xsolla
    const orderResponse = await fetch(
        `${XSOLLA_SDK_URL}/${projectId}/payment/item/${sku}`,
        {
            method: 'POST',
            headers: {
                Authorization: `Bearer ${userToken}`,
                'Content-Type': 'application/json',
            },
        },
    )

    if (!orderResponse.ok) {
        throw new Error(`Xsolla create order HTTP ${orderResponse.status}`)
    }

    const orderData = await orderResponse.json()
    const paymentToken = orderData.token
    const orderId = orderData.order_id

    const paystation = window.XPayStationWidget
    if (!paystation) {
        throw new Error('Xsolla Pay Station widget not loaded')
    }

    paystation.init({
        access_token: paymentToken,
        sandbox: isSandbox,
        childWindow: { target: '_blank' },
    })

    return new Promise((resolve, reject) => {
        let resolved = false

        const cleanup = () => {
            document.removeEventListener('visibilitychange', handleVisibilityChange)
        }

        const handleVisibilityChange = () => {
            // If the player closes the Pay Station window manually
            if (document.visibilityState === 'visible' && !resolved) {
                setTimeout(() => {
                    if (!resolved) {
                        resolved = true
                        cleanup()
                        reject(new Error('Purchase canceled/closed'))
                    }
                }, 1500)
            }
        }

        paystation.on(paystation.eventTypes.STATUS, (_evt, data) => {
            const info = (data && data.paymentInfo) || {}
            if (info.status && /done|charged|success/i.test(String(info.status))) {
                getXsollaOrder(projectId, orderId, userToken)
                    .then((order) => {
                        // Forward the order to the CrazyGames analytics pipeline
                        sdk.analytics.trackOrder('xsolla', order)

                        if (!resolved) {
                            resolved = true
                            cleanup()
                            resolve({ orderId, sku, ...order })
                        }
                    })
                    .catch((err) => {
                        if (!resolved) {
                            resolved = true
                            cleanup()
                            reject(err)
                        }
                    })
            }
        })

        paystation.on('close', () => {
            if (!resolved) {
                resolved = true
                cleanup()
                reject(new Error('Purchase canceled/closed'))
            }
        })

        paystation.open()
        document.addEventListener('visibilitychange', handleVisibilityChange)
    })
}
```

### Owned items and consumption

```javascript
async function getPurchases(sdk, projectId) {
    const token = await getXsollaToken(sdk)
    const res = await fetch(
        `${XSOLLA_SDK_URL}/${projectId}/user/inventory/items`,
        { headers: { Authorization: `Bearer ${token}` } },
    )
    if (!res.ok) {
        throw new Error(`Xsolla inventory HTTP ${res.status}`)
    }
    const data = await res.json()
    return data.items || []
}

async function consumePurchase(sdk, projectId, sku, quantity = 1) {
    const token = await getXsollaToken(sdk)
    const res = await fetch(
        `${XSOLLA_SDK_URL}/${projectId}/user/inventory/item/consume`,
        {
            method: 'POST',
            headers: {
                Authorization: `Bearer ${token}`,
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ sku, quantity }),
        },
    )
    if (!res.ok) {
        throw new Error(`Xsolla consume HTTP ${res.status}`)
    }
}
```

## Lifecycle / gameplay events

CrazyGames REQUIRES the host game to call gameplay and loading hooks at the right moments — they drive ad pacing, in-game UI dimming and analytics. Call them precisely.

```javascript
function onInGameLoadingStarted(sdk) {
    // The game has started loading a level / asset bundle
    sdk.game.loadingStart()
}

function onInGameLoadingStopped(sdk) {
    // The loading screen is done, user can interact
    sdk.game.loadingStop()
}

function onGameplayStarted(sdk) {
    // Active gameplay began (level start, level resumed)
    sdk.game.gameplayStart()
}

function onGameplayStopped(sdk) {
    // Gameplay paused, level finished or failed
    sdk.game.gameplayStop()
}

function onPlayerGotAchievement(sdk) {
    // Notify CrazyGames of a "happy" moment for replay clip selection
    sdk.game.happytime()
}
```

Suggested mapping from generic game lifecycle events:

- `level_started`, `level_resumed`, `gameplay_started` → `sdk.game.gameplayStart()`
- `level_paused`, `level_completed`, `level_failed`, `gameplay_stopped` → `sdk.game.gameplayStop()`
- `in_game_loading_started` → `sdk.game.loadingStart()`
- `in_game_loading_stopped` → `sdk.game.loadingStop()`
- `player_got_achievement` → `sdk.game.happytime()`

There are no separate `audio` / `pause` / `visibility` callbacks on the CrazyGames SDK — drive your audio and pause state from the ad callbacks (`adStarted` / `adFinished`) and the standard `document.visibilitychange` event.
