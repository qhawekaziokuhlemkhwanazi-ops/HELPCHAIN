# Deploy the HelpChain API to Render

This deploys the existing API from the repository's root `Dockerfile.api`. The app screens and API routes stay the same. Render's Free web service is suitable for testing, but it can sleep when idle and is not intended for production workloads.

## Before you start

You need:

- Access to the HelpChain GitHub repository and its default branch.
- The Firebase project used by the app. Confirm that `FIREBASE_PROJECT_ID` below matches `EXPO_PUBLIC_FIREBASE_PROJECT_ID` in the app configuration.
- A Firebase Admin service-account JSON for that same project.
- An email-sending account with a verified sender domain. This guide uses Resend; Render Free blocks outbound SMTP on ports 25, 465, and 587, so the configured SMTP port is Resend's supported port `2587`.

Do not put Firebase Admin credentials, email API keys, or generated secrets in source files, chat messages, or commits. Enter them only in Render's secret-value prompts or its Environment settings. The Firebase client values prefixed `EXPO_PUBLIC_` are for the app; never use the Admin service-account JSON in the app.

## 1. Prepare Firebase and email delivery

1. In Firebase Console, select the same project used by the mobile app. Confirm **Authentication → Sign-in method → Email/Password** is enabled and that Firestore is set up with this repository's rules and indexes.
2. Under **Project settings → Service accounts → Firebase Admin SDK**, generate a private key for the service account. Keep the downloaded JSON private; it will be entered as `FIREBASE_SERVICE_ACCOUNT_JSON` in Render. If the key-generation option is unavailable, ask a Firebase project administrator to provide a credential securely.
3. Record the Firebase **Project ID**; enter this same ID for both the app's `EXPO_PUBLIC_FIREBASE_PROJECT_ID` and Render's `FIREBASE_PROJECT_ID`.

4. Create a Resend account and add a domain you control.
5. Follow Resend's instructions to add and verify the domain's DNS records.
6. Create a Resend API key. Keep it private; it will be entered as `SMTP_PASSWORD` in Render.
7. Choose a sender address on that verified domain, for example `HelpChain <donations@your-verified-domain>`. Use that exact address for `SMTP_FROM`.

Resend requires a verified domain for SMTP sending. Its documented SMTP settings are host `smtp.resend.com`, username `resend`, and port `2587` with STARTTLS. The Blueprint already sets these non-secret connection values.

## 2. Create the Render service

1. Sign in to [Render](https://dashboard.render.com/) and connect the GitHub account that can access the HelpChain repository.
2. Push or merge the `render.yaml` file at the repository root to the branch you want to deploy.
3. In Render, choose **New → Blueprint**, select the HelpChain repository and branch, then create the Blueprint. Render reads the root `render.yaml` and creates the `helpchain-api` Free web service from `Dockerfile.api`.
4. When prompted for the Blueprint's unsynced environment values (some are not secrets), enter:
   - `FIREBASE_PROJECT_ID`: the project ID recorded in step 1 (not a secret).
   - `FIREBASE_SERVICE_ACCOUNT_JSON`: the full private JSON for the Firebase Admin service account. Paste the intact JSON document.
   - `EMAIL_OTP_SECRET`: a new private random value of at least 32 characters.
   - `SMTP_PASSWORD`: the Resend API key.
   - `SMTP_FROM`: the sender address on the verified domain.
   - `DONATION_EXPIRY_SECRET`: a separate new private random value of at least 32 characters.
5. Start the first deploy and wait for it to finish successfully. The Firebase service account must belong to the same Firebase project as `FIREBASE_PROJECT_ID` and have the Firebase Authentication and Firestore permissions needed by the API.

If you need random values, generate them locally with a password manager or a command such as `openssl rand -hex 32`. Do not paste generated values into this repository or send them in chat. If a Blueprint is already created, add or update secrets in **Render → helpchain-api → Environment**; Blueprint `sync: false` prompts appear only during initial creation.

## 3. Check the API

Copy the service's public `onrender.com` URL from the Render Dashboard. Open:

```text
https://YOUR-SERVICE.onrender.com/api/healthz
```

It should return:

```json
{"status":"ok"}
```

The first request after the service has been idle can take longer because a Free instance spins down after 15 minutes without traffic.

## 4. Connect the mobile app

In `artifacts/helpchain/.env`, set `EXPO_PUBLIC_API_BASE_URL` to the service origin only; do not add `/api`:

```env
EXPO_PUBLIC_API_BASE_URL=https://YOUR-SERVICE.onrender.com
```

Keep the app's six `EXPO_PUBLIC_FIREBASE_*` values configured for the same Firebase project. Restart Expo after changing `.env`; for a published app, set the same public app variables in the EAS build environment and create a new build. Sign up with a test account and confirm the verification email arrives before testing donation registration.

## 5. Keep donation-request expiration enabled

The API includes a protected endpoint that expires stale reservations and sends notifications. The included GitHub Actions workflow calls it every five minutes. In the repository, open **Settings → Secrets and variables → Actions** and add:

- **Variable** `DONATION_EXPIRY_URL`: `https://YOUR-SERVICE.onrender.com/api/donation-requests/expire`
- **Secret** `DONATION_EXPIRY_SECRET`: the exact same value entered for the Render environment variable of that name.

Do not put the secret in the URL, workflow YAML, or a normal Actions variable. You can run **Actions → Expire donation requests → Run workflow** once to test it; scheduled runs then continue automatically. The workflow reports an error if either setting is missing or the API rejects the request.

## Free-service limitations

Render's Free web services can spin down after 15 minutes of inactivity, have limited monthly usage, and are not recommended for production. Email sending uses port `2587` to avoid Render's blocked outbound SMTP ports. If the free instance is suspended, sleeping, or otherwise unavailable, app requests to the API will fail until it is available again. Check Render's current [Free instance limitations](https://render.com/docs/free) before relying on it for real donations.

For a paid, always-on deployment, use the existing [Cloud Run deployment guide](./DEPLOY_CLOUD_RUN.md).
