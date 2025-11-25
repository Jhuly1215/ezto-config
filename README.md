# ezto-config

`ezto-config` is the **central configuration repository** for the [EzTo Gym Platform](https://github.com/ChristianMendozaa/ezto).

Instead of spreading environment variables and secrets across multiple places, this repo acts as the **single source of truth** for:

- Environment-specific settings (development, staging, production).
- Backend (FastAPI) configuration for the EzTo API.
- Frontend (web client) configuration for the EzTo UI.
- Firebase / Firestore project references and service account wiring (without exposing real secrets).
- CI/CD configuration templates used by deployment pipelines.

> ⚠️ **Important:** This repository should not contain real secrets (API keys, private keys, etc.).  
> Only templates, examples and non-sensitive defaults should live here.  
> Real values must be injected via your CI/CD tool, Vercel, or server secret manager.

---

## Table of Contents

1. [Repository Role in the EzTo Architecture](#repository-role-in-the-ezto-architecture)
2. [Branches and Environments](#branches-and-environments)
3. [Suggested Directory Structure](#suggested-directory-structure)
4. [Configuration Types](#configuration-types)
   - [Backend (FastAPI API)](#backend-fastapi-api)
   - [Frontend (Web Client)](#frontend-web-client)
   - [Firebase / Firestore](#firebase--firestore)
5. [How EzTo Uses `ezto-config`](#how-ezto-uses-ezto-config)
6. [Local Development Workflow](#local-development-workflow)
7. [CI/CD & Deployment Workflow](#cicd--deployment-workflow)
8. [Adding New Environment Variables](#adding-new-environment-variables)
9. [Security Guidelines](#security-guidelines)
10. [Troubleshooting](#troubleshooting)

---

## Repository Role in the EzTo Architecture

`ezto-config` belongs to a multi-repo architecture:

- **Main app:** [`ezto`](https://github.com/ChristianMendozaa/ezto)
  - `server/`: FastAPI backend.
  - `client/`: React/Next.js web app.

- **Config repo:** `ezto-config` _(this repo)_
  - Holds **environment configuration** for the EzTo stack.
  - Keeps all environments (dev, prod, etc.) consistent and version-controlled.

The idea is:

- Developers and DevOps can see **which variables exist** and **how they differ per environment**.
- CI/CD pipelines pull the appropriate config from this repo and inject it as **environment variables** into:
  - The EzTo **backend** container / process.
  - The EzTo **frontend** build (e.g. Vercel env vars).
- The EzTo repos themselves can remain clean of secrets and per-environment noise.

---

## Branches and Environments

This repo currently has at least two branches:

- `main`  
  Default branch, typically used for **development / staging** configuration.

- `prod`  
  Production configuration (used for live deployments).

A common mapping:

| Branch | Environment      | Usage                                 |
|--------|------------------|---------------------------------------|
| `main` | `development`    | Local dev, test deployments, staging. |
| `prod` | `production`     | Live EzTo instance used by real users.|

You can add more branches if you need additional environments (`qa`, `demo`, etc.), but these two cover the usual workflow.

---

## Suggested Directory Structure

> ⚠️ **Note:** Adjust this README to match your actual file layout.  
> The following structure is a **recommended pattern** for how to organise this repo:

```text
ezto-config/
├── backend/
│   ├── .env.backend.dev.example
│   ├── .env.backend.staging.example
│   └── .env.backend.prod.example
├── frontend/
│   ├── .env.client.dev.example
│   ├── .env.client.staging.example
│   └── .env.client.prod.example
├── firebase/
│   ├── README.md              # How Firebase projects map to environments
│   └── (no real service account JSON here!)
└── docs/
    ├── CONFIG_OVERVIEW.md     # Optional in-depth doc
    └── SECRETS_GUIDE.md       # How and where secrets are stored
````

You don’t have to use this exact structure, but it gives a clean separation between:

* **Backend** config (`backend/`).
* **Frontend** config (`frontend/`).
* **Cloud / Firebase mapping** (`firebase/`).
* **Documentation** (`docs/`).

---

## Configuration Types

### Backend (FastAPI API)

Backend config is used by `ezto/server` and typically exposes variables such as:

* **General**

  * `ENVIRONMENT`
    (`development`, `staging`, `production`)

  * `API_PORT`, `API_HOST`

* **CORS**

  * `ALLOWED_ORIGINS`
    CSV list of allowed frontend origins.

* **Firebase / Firestore**

  * `FIREBASE_PROJECT_ID`
  * `FIREBASE_CLIENT_EMAIL`
  * `FIREBASE_PRIVATE_KEY`
    (Provided via secret manager in real deployments.)
  * `FIREBASE_DATABASE_URL` (if needed)

* **Auth & Security**

  * `JWT_SECRET`
  * `JWT_ALGORITHM`
  * `ACCESS_TOKEN_EXPIRE_MINUTES`

* **Integrations (optional)**

  * `PAYMENTS_PROVIDER_API_KEY`
  * `NFC_GATEWAY_URL`
  * Any other external services used by EzTo.

> In this repo, backend config files are usually stored as **templates**
> such as `.env.backend.dev.example` and `.env.backend.prod.example`.
> Developers copy them to `.env` in the backend repo for local runs, while CI/CD reads them to inject environment variables.

---

### Frontend (Web Client)

Frontend config is used by `ezto/client` and mostly contains public-facing or build-time variables, for example:

* **API base URL**

  * `NEXT_PUBLIC_API_URL`
    (`http://localhost:8000` in dev, backend URL in prod)

* **Firebase client config (if used directly)**

  * `NEXT_PUBLIC_FIREBASE_API_KEY`
  * `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`
  * `NEXT_PUBLIC_FIREBASE_PROJECT_ID`
  * `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`
  * `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
  * `NEXT_PUBLIC_FIREBASE_APP_ID`

Typical templates:

* `.env.client.dev.example`
* `.env.client.prod.example`

On Vercel, these values normally end up as **project environment variables**.

---

### Firebase / Firestore

This repo should document which Firebase project is used per environment and how to configure credentials.

Recommended content for `firebase/README.md` (adapt to your needs):

* **Dev environment**

  * Firebase project ID (e.g. `ezto-dev`)
  * Firestore mode: `Native`
  * Simple access rules for local testing.
  * A note that service account JSON should be stored in:

    * Local `.env` (for dev) using `FIREBASE_PRIVATE_KEY`, etc.
    * A secret manager / CI variable in production.

* **Prod environment**

  * Firebase project ID (e.g. `ezto-prod`)
  * Stricter Firestore security rules.
  * Monitoring / logging settings.

> Again: **do not** store real `*.json` service account files here.
> Only document the mapping and how to configure them.

---

## How EzTo Uses `ezto-config`

At a high level, the workflow looks like this:

1. **Developer or pipeline selects a branch** in `ezto-config`:

   * `main` → dev/staging config
   * `prod` → production config

2. **Config files are read** from this repo:

   * For local dev: developers manually copy a template to `.env` in the `ezto` repo.
   * For CI/CD: a pipeline reads this repo’s files and exports their values as environment variables.

3. **EzTo services start**:

   * The backend (FastAPI) reads environment variables and connects to:

     * Firebase project for that environment.
     * Any external services configured in `ezto-config`.
   * The frontend build (Vercel or local) uses `NEXT_PUBLIC_...` variables to talk to the correct backend.

This ensures that when you switch environment, **you only need to change `ezto-config` and redeploy**.

---

## Local Development Workflow

1. **Clone repositories**

```bash
# Main app
git clone https://github.com/ChristianMendozaa/ezto.git

# Config repo
git clone https://github.com/Jhuly1215/ezto-config.git
```

2. **Check out the `main` branch** (dev config):

```bash
cd ezto-config
git checkout main
```

3. **Copy template environment files** into the EzTo repo:

Example (adapt filenames to your actual structure):

```bash
# Backend
cp backend/.env.backend.dev.example ../ezto/server/.env

# Frontend
cp frontend/.env.client.dev.example ../ezto/client/.env.local
```

4. **Fill in any missing secrets locally**
   (Firebase private key, JWT secret, etc.) in the copied `.env` files.

5. **Run EzTo normally** (see EzTo README):

   * Start backend (FastAPI).
   * Start frontend (Next.js).
   * Use local URLs defined by your `.env` files.

---

## CI/CD & Deployment Workflow

A typical deployment pipeline using `ezto-config` might:

1. **Check out the application repo** (`ezto`).
2. **Check out the config repo** (`ezto-config`) at the appropriate branch:

   * `prod` for production deployments.
3. **Load configuration files** and export environment variables:

   * For backend container / process.
   * For frontend build step (e.g. Vercel, Docker image build).
4. **Deploy services**:

   * Backend to your server / container platform.
   * Frontend to Vercel or similar, with environment variables set.

Because configuration is versioned:

* You can track **when** a config changed.
* You can roll back to previous config versions by reverting commits in `ezto-config`.
* Config changes can be reviewed via **pull requests** like any other code.

---

## Adding New Environment Variables

When EzTo requires new configuration:

1. **Add the variable to all relevant environment templates** in this repo:

   * `backend/.env.backend.dev.example`
   * `backend/.env.backend.prod.example`
   * `frontend/.env.client.dev.example`
   * `frontend/.env.client.prod.example`
2. **Document the variable** in a short comment or in `docs/CONFIG_OVERVIEW.md`:

   * What the variable does.
   * Which values are valid.
   * Which service uses it.
3. **Update EzTo code** to read the new variable:

   * Backend: settings class / FastAPI config.
   * Frontend: `process.env.NEXT_PUBLIC_...` or similar.
4. **Update CI/CD configuration** to set the real value in each environment.

This keeps configuration consistent and discoverable.

---

## Security Guidelines

* **Do NOT commit secrets**

  * No real Firebase service account JSON.
  * No real API keys, passwords or JWT secrets.
  * Only use `*.example` files with placeholder values.

* **Use a secret manager**

  * In production, secrets should come from:

    * Vercel project environment variables.
    * Your cloud provider’s secret manager.
    * GitHub Actions secrets, etc.

* **Limit access**

  * Only team members who actually manage deployments should have write access to the `prod` branch.
  * Use pull requests and reviews for config changes, especially in production.

* **Audit config changes**

  * Treat this repo like code: review and approve changes.
  * Keep a changelog in `docs/` if necessary.

---

## Troubleshooting

**“Backend can’t connect to Firestore”**

* Check that the correct Firebase project ID is documented in this repo.
* Ensure the runtime environment has valid `FIREBASE_*` variables.

**“Frontend calls wrong API URL”**

* Verify `NEXT_PUBLIC_API_URL` in the appropriate frontend config template.
* Confirm the correct branch (`main` vs `prod`) is being used by the pipeline.

**“Environments are mixed up”**

* Double-check which branch of `ezto-config` your CI/CD pipeline checks out.
* Make sure production deployments always use the `prod` branch.

---

## License

This repository is part of the EzTo project.
Use and access are intended for the EzTo team and collaborators.

