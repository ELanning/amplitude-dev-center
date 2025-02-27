---
title: Experiment React Native SDK
description: Official documentation for Amplitude Experiment's Client-side React Native SDK.
icon: simple/react
---

Official documentation for Amplitude Experiment's Client-side React Native SDK.

!!!info "SDK Resources"

    [:material-github: Github](https://github.com/amplitude/experiment-react-native-client) · [:material-code-tags-check: Releases](https://github.com/amplitude/experiment-react-native-client/releases) · [:material-book: API Reference](https://amplitude.github.io/experiment-react-native-client/)

## Install

![npm version](https://img.shields.io/npm/v/@amplitude/experiment-react-native-client)

Install the Experiment JavaScript Client SDK. This library depends on `@react-native-async-storage/async-storage` which you also need to install.

!!!info "Web and Expo Support"
    This SDK can be used for react-native apps built for web or built using [Expo](https://expo.dev/) (Expo Go not yet supported).

=== "npm"

    ```bash
    npm install @amplitude/experiment-react-native-client
    npm install @react-native-async-storage/async-storage
    ```

=== "yarn"

    ```bash
    yarn add @amplitude/experiment-react-native-client
    yarn add @react-native-async-storage/async-storage
    ```

=== "expo"

    ```bash
    expo install @amplitude/experiment-react-native-client
    expo install @react-native-async-storage/async-storage
    ```

You need to install the native modules to run the SDK on iOS.

```bash
cd ios
pod install
```

!!!tip "Quick Start"

    1. [Initialize the experiment client](#initialize)
    2. [Fetch variants for the user](#fetch)
    3. [Access a flag's variant](#variant)

    ```js
    import { Experiment } from '@amplitude/experiment-react-native-client';

    // (1) Initialize the experiment client
    const experiment = Experiment.initialize('<DEPLOYMENT_KEY>');

    // (2) Fetch variants for a user
    const user = {
        user_id: 'user@company.com',
        device_id: 'abcdefg',
        user_properties: {
            'premium': true,
        },
    };
    await experiment.fetch(user);

    // (3) Lookup a flag's variant
    const variant = experiment.variant('<FLAG_KEY>');
    if (variant.value === 'on') {
        // Flag is on
    } else {
        // Flag is off
    }
    ```

    **Not getting the expected variant result for your flag?** Make sure your flag [is activated](../guides/getting-started/create-a-flag.md#activate-the-flag), has a [deployment set](../guides/getting-started/create-a-flag.md#add-a-deployment), and has [users allocated](../guides/getting-started/create-a-flag.md#configure-targeting-rules).

## Core functions

The following functions make up the core of the Experiment client-side SDK.

---

### Initialize

The SDK client should be initialized in your application on startup. The [deployment key](../general/data-model.md#deployments) argument passed into the `apiKey` parameter must live within the same project that you are sending analytics events to.

```js
initialize(apiKey: string, config?: ExperimentConfig): ExperimentClient
```

| Parameter | Requirement | Description |
| --- | --- | --- |
| `apiKey` | required | The [deployment key](../general/data-model.md#deployments) which authorizes fetch requests and determines which flags should be evaluated for the user. |
| `config` | optional | The client [configuration](#configuration) used to customize SDK client behavior. |

The initializer returns a singleton instance, so subsequent initializations for the same instance name will always return the initial instance. To create multiple instances, use the `instanceName` [configuration](#configuration).

```js
import { Experiment } from '@amplitude/experiment-react-native-client';

const experiment = Experiment.initialize('<DEPLOYMENT_KEY>');
```

#### Integrations

If you use either Amplitude or Segment Analytics SDKs to track events into Amplitude, you'll want to set up an integration on initialization. Integrations automatically implement [provider](#providers) interfaces to enable a more streamlined developer experience by making it easier to **manage user identity** and **track exposures events**.

!!!amplitude "Amplitude Integration"

    **This SDK only integrates with the [next generation Amplitude Analytics React Native SDK](../../../data/sdks/typescript-react-native/).**

    The Amplitude Experiment SDK is set up to integrate seamlessly with the Amplitude Analytics SDK. All you need to do is update your SDK versions to the latest, and use the special integration initialization function.

    ```js hl_lines="2"
    init('<API_KEY>');
    const experiment = Experiment.initializeWithAmplitudeAnalytics('<DEPLOYMENT_KEY>');
    ```

    Note that, if you are using a custom instance name for analytics, you will need to set the same value in the `instanceName` [configuration option](#configuration) in the experiment SDK.

    Using the integration initializer will automatically configure implementations of the [user provider](#user-provider) and [exposure tracking provider](#exposure-tracking-provider) interfaces to pull user data from the Amplitude Analytics SDK and track exposure events.

???segment "Segment Integration"

    Experiment's integration with Segment Analytics is still a manual implementation at this point. Copy the exposure tracking provider implementation into your app code base and initialize the Experiment SDK with the provider instances in the configuration.

    ```js title="SegmentExposureTrackingProvider"
    class SegmentExposureTrackingProvider implements ExposureTrackingProvider {
        private analytics: Analytics;
        constructor(analytics: Analytics) {
            this.analytics = analytics;
        }
        track(exposure: Exposure) {
            this.analytics.track('$exposure', exposure);
        }
    }
    ```

    The Experiment SDK must then be configured on initialization with an instance of the the exposure tracking provider. Make sure this happens _after_ the analytics SDK has been loaded an initialized.

    ```js
    analytics.ready(() => {
        const experiment =  Experiment.initialize('<DEPLOYMENT_KEY>', {
            exposureTrackingProvider: new SegmentExposureTrackingProvider(analytics),
        });
    });
    ```

    When [fetching variants](#fetch), pass the segment anonymous ID and user ID for the device ID and user ID, respectively.

    ```js
    const userId = analytics.user().id()
    const deviceId = analytics.user().analyticsId()
    await experiment.fetch({
        user_id: userId,
        device_id: deviceId,
    });
    ```

#### Configuration

The SDK client can be configured once on initialization.

???config "Configuration Options"
    | <div class="big-column">Name</div> | Description | Default Value |
    | --- | --- | --- |
    | `debug` | Enable additional debug logging within the SDK. Should be set to false in production builds. | `false` |
    | `fallbackVariant` | The default variant to fall back if a variant for the provided key doesn't exist. | `{}` |
    | `initialVariants` | An initial set of variants to access. This field is valuable for bootstrapping the client SDK with values rendered by the server using server-side rendering (SSR). | `{}` |
    | `serverUrl` | The host to fetch variants from. | `https://api.lab.amplitude.com` |
    | `fetchTimeoutMillis` | The timeout for fetching variants in milliseconds. | `10000` |
    | `retryFetchOnFailure` | Whether to retry variant fetches in the background if the request doesn't succeed. | `true` |
    | `automaticExposureTracking` | If true, calling [`variant()`](#variant) will track an exposure event through the configured `exposureTrackingProvider`. If no exposure tracking provider is set, this configuration option does nothing.  | `true` |
    | `automaticFetchOnAmplitudeIdentityChange` | Only matters if you use the `initializeWithAmplitudeAnalytics` initialization function to seamlessly integrate with the Amplitude Analytics SDK. If `true` any change to the user ID, device ID or user properties from analytics will trigger the experiment SDK to fetch variants and update it's cache. | `false` |
    | `userProvider` | An interface used to provide the user object to `fetch()` when called. See [Experiment User](https://developers.experiment.amplitude.com/docs/experiment-user#user-providers) for more information. | `null` |
    | `exposureTrackingProvider` | Implement and configure this interface to track exposure events through the experiment SDK, either automatically or explicitly. | `null` |
    | `instanceName` | Custom instance name for experiment SDK instance. **The value of this field is case-sensitive.** | `null` |

---

!!!info "EU Data Center"
    If you're using Amplitude's EU data center, configure the `serverUrl` option on initialization to `https://api.lab.eu.amplitude.com`

### Fetch

Fetches variants for a [user](../general/data-model.md#users) and store the results in the client for fast access. This function [remote evaluates](../general/evaluation/remote-evaluation.md) the user for flags associated with the deployment used to initialize the SDK client.

```js
fetch(user?: ExperimentUser): Promise<Client>
```

| Parameter  | Requirement | Description |
| --- | --- | --- |
| `user` | optional | Explicit [user](../general/data-model.md#users) information to pass with the request to evaluate. This user information is merged with user information provided from [integrations](#integrations) via the [user provider](#user-provider), preferring properties passed explicitly to `fetch()` over provided properties. |

Amplitude Experiment recommends calling `fetch()` during application start up so that the user gets the most up-to-date variants for the application session. Furthermore, should wait for the fetch request to return a result before rendering the user experience to avoid the interface "flickering".

```js
const user = {
    user_id: 'user@company.com',
    device_id: 'abcdefg',
    user_properties: {
        'premium': true,
    },
};
await experiment.fetch(user);
```

If you're using an [integration](#integrations) or a custom [user provider](#user-provider) then you can fetch without inputting the user.

```js
await experiment.fetch();
```

???tip "Fetch When User Identity Changes"
    If you want the most up-to-date variants for the user, it's recommended that you call `fetch()` whenever the user state changes in a meaningful way. For example, if the user logs in and receives a user ID, or has a user property set which may effect flag or experiment targeting rules.

    In the case of **user properties**, Amplitude recommends passing new user properties explicitly to `fetch()` instead of relying on user enrichment prior to [remote evaluation](../general/evaluation/remote-evaluation.md). This is because user properties that are synced remotely through a separate system have no timing guarantees with respect to `fetch()`--i.e. a race.

!!!info "Timeout & Retries"
    If `fetch()` times out (default 10 seconds) or fails for any reason, the SDK client will return and retry in the background with back-off. You may configure the timeout or disable retries in the [configuration options](#configuration) when the SDK client is initialized.

---

### Variant

Access a [variant](../general/data-model.md#variants) for a [flag or experiment](../general/data-model.md#flags-and-experiments) from the SDK client's local store.

!!!info "Automatic Exposure Tracking"
    When an [integration](#integrations) is used or a custom [exposure tracking provider](#exposure-tracking-provider) is set, `variant()` will automatically track an exposure event through the tracking provider. To disable this functionality, [configure](#configuration) `automaticExposureTracking` to be `false`, and track exposures manually using [`exposure()`](#exposure).

```js
variant(key: string, fallback?: string | Variant): Variant
```

| Parameter | Requirement | Description |
| --- | --- | --- |
| `key` | required | The **flag key** to identify the [flag or experiment](../general/data-model.md#flags-and-experiments) to access the variant for. |
| `fallback` | optional | The value to return if no variant was found for the given `flagKey`. |

When determining which variant a user has been bucketed into, you'll want to compare the variant `value` to a well-known string.

```js
const variant = experiment.variant('<FLAG_KEY>');
if (variant.value === 'on') {
    // Flag is on
} else {
    // Flag is off
}
```

???info "Accessing the variant's payload"
    A variant may also be configured with a dynamic [payload](../general/data-model.md#variants) of arbitrary data. Access the `payload` field from the variant object after checking the variant's `value`.

    ```js
    const variant = experiment.variant('<FLAG_KEY>');
    if (variant.value === 'on') {
        const payload = variant.payload;
    }
    ```

A `null` variant `value` means that the user hasn't been bucketed into a variant. You may use the built in **fallback** parameter to provide a variant to return if the store doesn't contain a variant for the given flag key.

```js
const variant = experiment.variant('<FLAG_KEY>', { value: 'control' });
if (variant === 'control') {
    // Control
} else if (variant === 'treatment') {
    // Treatment
}
```

---

### All

Access all [variants](../general/data-model.md#variants) stored by the SDK client.

```js
all(): Variants
```

---

### Clear

Clear all [variants](../general/data-model.md#variants) in the cache and storage.

```js
clear(): void
```

You can call `clear` after user logout to clear the variants in cache and storage.

```js
experiment.clear();
```

---

### Exposure

Manually track an [exposure event](../general/experiment-event-tracking.md#exposure-events) for the current variant of the given flag key through configured [integration](#integrations) or custom [exposure tracking provider](#exposure-tracking-provider). Generally used in conjunction with setting the `automaticExposureTracking` [configuration](#configuration) optional to `false`.

```js
exposure(key: string): void
```

| Parameter | Requirement | Description |
| --- | --- | --- |
| `key` | required | The **flag key** to identify the [flag or experiment](../general/data-model.md#flags-and-experiments) variant to track an [exposure event](../general/experiment-event-tracking.md#exposure-events) for. |

```js
const variant = experiment.variant('<FLAG_KEY>');

// Do other things...

experiment.exposure('<FLAG_KEY>');
if (variant === 'control') {
    // Control
} else if (variant === 'treatment') {
    // Treatment
}
```

---

## Providers

!!!tip "Integrations"
    If you use Amplitude or Segment analytics SDKs along side the Experiment Client SDK, Amplitude recommends you use an [integration](#integrations) instead of implementing custom providers.

Provider implementations enable a more streamlined developer experience by making it easier to manage user identity and track exposures events.

### User provider

The user provider is used by the SDK client to access the most up-to-date user information only when it's needed (for example, when [`fetch()`](#fetch) is called). This provider is optional, but helps if you have a user information store already set up in your application. This way, you don't need to manage two separate user info stores in parallel, which may result in a divergent user state if the application user store is updated and experiment isn't (or vice versa).

```js title="ExperimentUserProvider"
interface ExperimentUserProvider {
  getUser(): ExperimentUser;
}
```

To use your custom user provider, set the `userProvider` [configuration](#configuration) option with an instance of your custom implementation on SDK initialization.

```js
const experiment = Experiment.initialize('<DEPLOYMENT_KEY>', {
    userProvider: new CustomUserProvider(),
});
```

### Exposure tracking provider

Implementing an exposure tracking provider is highly recommended. [Exposure tracking](../general/experiment-event-tracking.md#exposure-events) increases the accuracy and reliability of experiment results and improves visibility into which flags and experiments a user is exposed to.

```js title="ExposureTrackingProvider"
export interface ExposureTrackingProvider {
  track(exposure: Exposure): void;
}
```

The implementation of `track()` should track an event of type `$exposure` (a.k.a name) with two event properties, `flag_key` and `variant`, corresponding to the two fields on the `Exposure` object argument. Finally, the event tracked must eventually end up in Amplitude Analytics for the same project that the [deployment] used to [initialize](#initialize) the SDK client lives within, and for the same user that variants were [fetched](#fetch) for.

To use your custom user provider, set the `exposureTrackingProvider` [configuration](#configuration) option with an instance of your custom implementation on SDK initialization.

```js
const experiment = Experiment.initialize('<DEPLOYMENT_KEY>', {
    exposureTrackingProvider: new CustomExposureTrackingProvider(),
});
```

## Bootstrapping

You may want to bootstrap the experiment client with an initial set of flags and variants when variants are obtained from an external source (for example, not from calling `fetch()` on the SDK client). Use cases include [local evaluation](../general/evaluation/local-evaluation.md), [server-side rendering](../guides/server-side-rendering.md), or integration testing on specific variants.

To bootstrap the client, set the flags and variants in the `initialVariants` [configuration](#configuration) object, then set the `source` to `Source.InitialVariants` so that the SDK client prefers the bootstrapped variants over any previously fetched & stored variants for the same flags.

```js
const experiment = Experiment.initialize('<DEPLOYMENT_KEY>', {
    initialVariants: { /* Flags and variants */ },
    source: Source.InitialVariants,
});
```
