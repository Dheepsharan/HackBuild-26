# LocalLoop backend integration

The Firebase backend from the supplied ZIP is included in this repository and connected to the Craft Pulse browser prototype by `backend/connect-ui.js`. With blank Firebase settings, the sample demo remains available. With valid settings, the UI uses Firebase Auth, Firestore reads, and callable Functions for supported changes.

## What owns what

The eventual web, React Native, or Expo UI owns device-local presentation state: the selected tab and mode, drawers, search text and filters, map/list preference, device location and pin drag, and unsaved form input. It renders clear loading, empty, signed-out, validation, permission, offline, configuration, and retry states.

This backend owns identity, persisted records, validation, authorization, order transitions, service-duration snapshots, queue values, ready estimates, reviews, and rating aggregates. There is deliberately no production `localStorage` fallback for Firebase records. The prototype keeps a visibly labelled browser-only demo mode for evaluation; once Firebase settings are enabled, cloud records are held in page memory and mutations go through the API.

## Architecture

- Firebase Auth establishes the caller UID.
- Firestore provides authenticated, real-time reads only.
- Callable Cloud Functions perform every mutation with Admin SDK transactions.
- `firestore.rules` denies all direct client writes and all unspecified paths.
- `functions/src/domain.js` is Firebase-free. It validates inputs, derives service minutes, enforces the exact transition matrix, derives geohashes, and calculates work-hour-aware queues.
- `backend/` exposes the stable plain-object API. It is the only layer a UI should import.

The API adapter is created with:

```js
import { bootstrapLocalLoopBackend } from "./backend/bootstrap.js";

const localLoop = bootstrapLocalLoopBackend({
  environment: {
    FIREBASE_API_KEY: "…",
    FIREBASE_AUTH_DOMAIN: "…",
    FIREBASE_PROJECT_ID: "…",
    FIREBASE_APP_ID: "…",
  },
  functionsRegion: "asia-south1",
});
```

The adapter returns plain values and stable errors only. Subscription methods require `(onData, onError)` and return an unsubscribe function.

`auth.signInGoogle()` uses a browser popup by default. A native adapter can instead pass `{ idToken, accessToken? }` from its platform Google library, so the domain API does not depend on web UI details.

| Area | Methods |
| --- | --- |
| `auth` | `observeSession`, `signUpEmail`, `signInEmail`, `signInGoogle`, `signOut` |
| `users` | `getMine`, `updateMine`, `updateSavedMakers` |
| `providers` | `completeOnboarding`, `get`, `subscribe`, `update` |
| `discovery` | `list`, `subscribe` |
| `orders` | `create`, `subscribeOne`, `subscribeCustomerOrders`, `subscribeMakerOrders`, `acceptFixedPrice`, `sendQuote`, `confirmQuote`, `cancel`, `reject`, `start`, `markReady`, `markCollected` |
| `reviews` | `create`, `subscribeForProvider` |

Callers branch only on the normalized error code: `configuration`, `signed-out`, `permission-denied`, `validation`, `not-found`, `conflict`, `unavailable`, or `unknown`.

## Server-enforced workflow

The only legal transitions are:

| From | To | Actor | Condition |
| --- | --- | --- | --- |
| Requested | Accepted | Maker | The currently offered service has a fixed price |
| Requested | Quoted | Maker | The currently offered service is price-on-request |
| Requested | Rejected | Maker | A nonempty reason is supplied |
| Quoted | Accepted | Customer | A complete quote exists |
| Requested, Quoted, Accepted | Cancelled | Customer | Work has not started |
| Accepted | In progress | Maker | Server writes `startedAt` |
| In progress | Ready | Maker | Server writes `readyAt` |
| Ready | Collected | Maker | Server writes `collectedAt` |

Nothing can be deleted, reopened, moved backward, or skip a stage. Incoming order payloads cannot supply `serviceMinutes`, quotes, status, timestamps, queue fields, rating fields, or customer snapshots. The function derives the service duration from the selected current provider service and persists that snapshot once.

The queue counts Accepted and In progress orders. An in-progress job keeps at least 10% of its duration. The server divides remaining working minutes by worker count, skips closing hours and closed days, and calculates the newly accepted order’s ready time before adding that order to the persisted queue.

Provider geohashes are derived server-side from latitude and longitude. Moving a shop requires `relocate: true` and both new coordinates; it preserves `movedFrom` and sets `movedAt` on the server.

## Privacy decision: order contact snapshots

`customerName` and `customerPhone` are deliberately copied into each order when it is created. The maker may read an order but may not read a customer's private `users/{uid}` document, so these immutable fulfillment snapshots are necessary for the current product’s call/WhatsApp workflow. Do not put any additional private profile data in providers, reviews, or category records.

## Configure and deploy

1. Create a Firebase project and register the intended app.
2. Enable Firestore and Firebase Authentication. Enable Email/Password; configure Google only if Google sign-in is used.
3. Fill the public Firebase Web App settings in `firebase-config.js`; the browser adapter reads that file. Do not add an Admin service-account key to the site or repository.
4. Create Firestore category documents before maker onboarding. Use document IDs `tailoring`, `footwear`, `cakes`, and `carpentry`, with a `name` string in each. The Functions code checks the category document but does not seed it.
5. From the repository root, authenticate the Firebase CLI and deploy with `firebase deploy --only firestore:rules,firestore:indexes,functions`. The `functions/package.json` specifies Node.js 22; the Firebase project must use the Blaze billing plan to deploy Cloud Functions.
6. The prototype imports the modular Firebase Web SDK from Google’s CDN through the import map in `index.html`. For production, consider adding a bundler and configure Firebase App Check before launch.

No Firebase project identifiers, credentials, category seed records, emulator deployment, KYC endpoint, Cloud Storage upload endpoint, or production deployment are included here.

## Tests

Run the pure, dependency-free checks:

```powershell
node --check backend/contracts.js
node --check backend/firebase/FirebaseLocalLoopApi.js
node --check functions/src/handlers.js
node --test backend/tests/contracts.test.js functions/test/domain.test.js
```

`backend/tests/firestore.rules.test.js` is opt-in and covers authentication, private users, provider/review reads, constrained order reads, direct-write denial, and unknown collections. Start the Emulator Suite and set `RUN_FIRESTORE_RULES_TESTS=true` to run it.

`functions/test/handlers.integration.test.js` is also opt-in through `RUN_FUNCTIONS_INTEGRATION_TESTS=true` and a `FIRESTORE_EMULATOR_HOST`. It exercises profile initialization, provider onboarding, fixed and quoted workflows, illegal ownership and transition attempts, concurrent queue acceptance, a single collected-order review, and rating aggregates against an Admin Firestore Emulator. This local machine currently has Java 8, while the Firebase Emulator Suite requires Java 11 or newer, so emulator execution is intentionally not claimed as verified.
