# Craft Pulse

A neighbourhood-maker prototype with two ways to run it:

- **Demo mode:** sample accounts and listings stored only in this browser.
- **Firebase mode:** the supplied backend handles real sign-in, maker listings, saved makers, orders, queue estimates and reviews after you connect a Firebase project.

## 1. Open the demo

Open `index.html` in a browser. Choose **Customer demo** to place an order, or choose a maker button such as **Sole Stories · Footwear repair** to open that shop’s inbox. The demo has a separate inbox for each sample shop, and an order appears only in the inbox for the shop selected by the customer. Demo records stay in this browser; they are not real accounts.

To see an order appear in real time, open the **same page address** in two tabs. Use **Customer demo** in one tab and the maker inbox for the ordered shop in the other. For reliable sharing, run the page at `http://localhost:8000` with the command below instead of opening two different copies of the HTML file.

Every customer order now has a **View invoice** button. After the maker accepts a priced order, **Payment options** offers simulated UPI/card and Pay at pickup. The invoice can be printed or saved as a PDF from the browser print dialog. Demo UPI and card do not charge money. The supplied Firebase backend does not include a real payment provider, so live orders must arrange payment with the maker.

## 2. Run the Firebase-connected version

Firebase needs the page to run from a web address. On Windows, open PowerShell in this repository folder and run:

```powershell
py -m http.server 8000
```

Then open [http://localhost:8000](http://localhost:8000). Keep that PowerShell window open while using the page. Firebase sign-in and location permissions need a secure web address such as localhost or HTTPS; opening the file directly is only for the demo.

### Connect your Firebase project

1. Create a Firebase project and register a **Web app** in it.
2. In Firebase Console, enable **Authentication → Email/Password** and create a **Cloud Firestore** database.
3. Open `firebase-config.js`. Copy the web-app values (`apiKey`, `authDomain`, `projectId`, `appId`, and any optional values) from **Project settings → Your apps** into the matching fields.
4. In Firestore, create a collection named `categories`. Add these documents, each with a `name` field of type string:

   | Document ID | `name` |
   | --- | --- |
   | `tailoring` | `Tailoring` |
   | `footwear` | `Footwear repair` |
   | `cakes` | `Cakes` |
   | `carpentry` | `Carpentry` |

   The supplied backend checks that a category exists before it accepts a maker registration.
5. Install the Firebase command-line tool if needed, then sign in and choose your project from this repository folder:

   ```powershell
   npm install -g firebase-tools
   firebase login
   firebase use --add
   ```

6. Deploy the security rules, database indexes and order functions:

   ```powershell
   firebase deploy --only firestore:rules,firestore:indexes,functions
   ```

   The functions use Node.js 22. Firebase Cloud Functions require a Firebase **Blaze pay-as-you-go plan** with a billing account. The plan may have no-cost quotas, but it is still pay-as-you-go; set budget alerts and review current pricing before deploying. Nothing in this repository enables billing or deploys services for you.

7. Reload the page on `http://localhost:8000`. The top status should say **Firebase ready · sign in**. Create an account, then register a maker profile. Use the location button to capture the maker's coordinates.

The web configuration in `firebase-config.js` identifies the Firebase web app; it is not an Admin service-account key. **Never put a service-account JSON file or private key in the website or GitHub.** The backend writes data through Cloud Functions, and Firestore rules deny direct browser writes.

## What the backend supports

- Email/password sign-in and account creation
- Maker discovery, categories, rates, hours, location and worker count
- Saved makers
- Order requests, quotes, secure status changes and queue estimates
- Reviews after an order is collected
- Google Maps search links and device-based distance estimates

## Features still needing backend work

The ZIP does not contain a KYC/identity-check service or Firebase Storage photo-upload code. The KYC screen therefore tells you it is not connected, and shop-logo/work-photo previews are not uploaded. The provider data model also has no shop introduction or individual product-photo fields; it stores services and their rates/times. Do not enter identity documents into this prototype. Google Maps links open Maps using maker coordinates; this prototype does not embed the Google Maps API map.

When Firebase settings are blank, the app remains in demo mode. When settings are filled, use a real Firebase account; the demo-account buttons are hidden.

## Repository contents

- `index.html` — the Craft Pulse UI and demo data
- `backend/connect-ui.js` — connects supported UI actions to the backend
- `backend/` — Firebase web API, validation and data mapping
- `functions/` — trusted Cloud Functions for writes and order transitions
- `firestore.rules` and `firestore.indexes.json` — database security and query setup
- `firebase.json` — Firebase CLI deployment configuration
- `firebase-config.js` — your Firebase web-app settings
- `BACKEND_INTEGRATION.md` — backend architecture and API reference
