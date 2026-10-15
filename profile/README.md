# VerifyDoc README

| Field | Value |
|---|---|
| Version | 1.0 |
| Status | Draft |
| Last Updated | 15/10/2026 |
| Owner | VerifyDoc |


## Overview

VerifyDoc is a role-based digital credential verification platform for universities, polytechnics, bootcamps, employers, and citizens. This repository contains the backend REST API that powers the VerifyDoc MVP.

In VerifyDoc, approved institutions issue credentials and review citizen uploads. Citizens store credentials in a private wallet and control sharing with approved employer organizations. Employers review shared credentials and can request fresh confirmation from the issuing institution.

## Project Status

Core backend workflows have been implemented, and frontend integration is underway. Integration validation, device and browser testing, outstanding fixes, real-user acceptance testing, and production readiness checks remain in progress. This README does not describe a customer-facing launch.

## Features

- Registration and login for citizens, employer organizations, and institutions, with JWT authentication
- Email verification and a password reset flow that uses one-time codes
- Admin approval of employer organizations and institutions before they can use protected features
- Credential issuance by approved institutions
- Citizen credential upload, reviewed by the eligible issuing institution (verify or reject); credentials from unregistered institutions remain Pending in the wallet
- Credential revocation by the issuing institution
- A private citizen wallet with filtering and pagination
- Authenticated QR code generation and limited public credential verification through a signed token
- Credential sharing: citizens share one credential at a time with an approved employer, set a purpose and expiry (the documented maximum of 365 days requires confirmation against the current backend), and can revoke a share
- Employer access to shared credentials, with private document links left out
- Fresh confirmation: employers ask the issuing institution to reconfirm a credential, and the institution responds Confirmed or Disputed

Registered institution selection is an MVP requirement: citizens should be able to reference a registered institution during upload, while retaining a manually entered institution name for unregistered issuers. Selecting an institution must not automatically verify a credential. Completion of this integration requires confirmation.

The QR workflow above reflects the public signed-token implementation recorded in MVP - 2. The later security overview describes login-required scanning; the access policy must be reconciled across the implementation and documentation before release.

## Not Included in the MVP

- Search for citizens by name, email, or phone number
- Audit logs and notifications
- Advanced fraud detection and AI or OCR document tamper detection
- Live integrations with external bodies (for example NIMC, WAEC, or CAC)
- Payments or subscriptions
- Account deletion

## Tech Stack

- Node.js and Express
- MongoDB (MongoDB Atlas) with Mongoose
- JSON Web Tokens (JWT) and bcrypt
- An email service for password-reset and verification codes
- A QR code library
- Cloudinary for document storage (integration not finalized)


## Architecture

A request passes through these layers in order:

```text
Routes → Middleware → Controllers → Services → Models → MongoDB
```

| Layer | Responsibility |
|---|---|
| Routes | Map endpoints to middleware and controllers |
| Middleware | Handle JWT authentication, roles, approval status, ownership, request validation, uploads, and errors |
| Controllers | Read request input, call services, and send responses |
| Services | Hold business logic and database workflows |
| Models | Define MongoDB schemas, validation, and relationships |
| Utils | Provide helpers such as credential ID generation, QR token generation, and OTP hashing |


## Getting Started

Follow these steps to run the API locally.

### Prerequisites

You need Node.js [version TBC], npm, and access to a MongoDB database (VerifyDoc uses MongoDB Atlas).

### Installation

1. Clone the repository and move into it:

```bash
   git clone <repository-url>
   cd VerifyDoc-backend-system
```

2. Install dependencies:

```bash
   npm install
```

3. Create a `.env` file in the project root and add the variables listed below.
4. Start the development server:

```bash
   npm run dev
```

5. Send `GET /` to `http://localhost:5000` (or your configured port). The server replies with the plain text `Server is running`.

### Environment Variables

Set these in your `.env` file. Never commit real values. Confirm the exact variable names against the current backend configuration before setup; this list has not been freshly audited against the repository.

| Variable | Description |
|---|---|
| `PORT` | The port the server listens on (for example `5000`) |
| `MONGODB_URI` | Your MongoDB connection string |
| `JWT_SECRET` | A long, random secret used to sign access tokens |
| `JWT_EXPIRES_IN` | Access token lifetime (for example `1d`) |
| `FRONTEND_URL` | The frontend origin used to build QR verification links |
| `QR_VERIFICATION_SECRET` | The secret used to sign QR verification tokens |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Document storage credentials (integration not finalized) |
| `RESEND_API_KEY`, `EMAIL_FROM` | Email delivery for one-time codes |


### Scripts

Confirm these commands against the current `package.json`.

| Command | Purpose |
|---|---|
| `npm run dev` | Start the server in development mode |
| `npm start` | Start the server in production mode |
| `npm test` | Run the automated tests |


### Testing

Backend automated tests, including integration and security tests, were reported in the implementation history. Confirm the current test command, database requirements, and coverage against the repository; frontend syntax checks alone do not establish end-to-end correctness.

## API Overview

The draft API inventory lists 41 endpoints; the counts and exact paths below require confirmation against the current routes. Paths are relative to your deployed API origin, for example `http://localhost:5000/auth/login` locally. Protected endpoints require the header `Authorization: Bearer <access_token>`. Access-token expiry is determined by the backend configuration; confirm the active lifetime before documenting a fixed duration.

QR generation requires authentication and applicable permissions. The public verification route uses a signed token rather than a login token: `GET /credentials/verify/:credentialId?token=<signed_token>`. Its limited response includes credential title, issuer, issue date, verification/revocation status, and `isValid`; it excludes personal details, credential documents, Cloudinary identifiers, and internal database identifiers. Full credential details and employer sharing remain protected.

| Resource | Example paths | Endpoints |
|---|---|---|
| Authentication and users | `/auth/register`, `/auth/login`, `/auth/me`, `/users/me` | 10 |
| Onboarding | `/auth/register-organization`, `/auth/register-institution` | 2 |
| Admin | `/admin/approval-queue`, `/admin/organizations/:id/approve` | 9 |
| Credentials and wallet | `/credentials`, `/credentials/wallet` | 7 |
| QR verification | `/credentials/:credentialId/qr`, `/credentials/verify/:credentialId?token=<signed_token>` | 2 |
| Credential sharing | `/credential-shares`, `/credential-shares/mine` | 6 |
| Fresh confirmation | `/credential-shares/organization/:id/confirmations` | 4 |
| Health check | `/` | 1 |

For full request and response details, see the VerifyDoc API Documentation.

## Roles and Statuses

VerifyDoc has four roles:

| Role | Main responsibility |
|---|---|
| Citizen | Views their own credentials, uploads credentials for review, and shares them with approved employers |
| Employer organization | Views credentials shared with it and requests fresh confirmation |
| Institution | Issues credentials, reviews citizen uploads, revokes credentials, and responds to confirmation requests |
| Platform Admin | Approves or rejects employer organizations and institutions |

Credential and sharing records use these status values:

| Record | Field | Values |
|---|---|---|
| Credential | `verificationStatus` | `Pending`, `Verified`, `Rejected` |
| Credential | `status` | `Active`, `Revoked` |
| Share | `status` | `Active`, `Expired`, `Revoked`, `Unavailable` |
| Confirmation request | `status` | `Pending`, `Confirmed`, `Disputed` |

## Core Rules

- Only approved institutions can issue, verify, reject, or revoke credentials.
- Only approved employer organizations can receive shared credentials or request fresh confirmation.
- Citizens access their own private wallet, and employers can't search for citizens.
- Employers access shared credential details only while an active, unexpired share exists, and their view leaves out the private document link. The limited public QR verification result is separate from this access.
- A citizen-uploaded credential starts as `Pending`. An eligible issuing institution can verify or reject it; selecting a registered institution does not automatically verify it.
- Credentials from unregistered institutions remain `Pending` and can be retained in the citizen's wallet.
- Admin accounts can't be created through public registration.
- Institutions cannot edit an already-issued credential. Credentials are revoked, not deleted, and errors are corrected by revoking and reissuing.

## Security

- Passwords are hashed with bcrypt, and password-reset codes are hashed and expire.
- Access uses JWTs, with role-based authorization, approval status checks, ownership checks, and request validation on protected actions.
- Secrets are read from environment variables, never from source code.
- Production hardening is still in progress, including a refresh-token policy, network-level protections, and backup and recovery.

For details, see the Security Architecture Overview.

## Known Limitations

- Institution registration and verification-email bugs were previously reported. Their current resolution status requires confirmation before treating them as active limitations.
- Old-image cleanup when replacing a profile picture was reported as a limitation; confirm its current status.
- A refresh-token mechanism has not been confirmed. Without one, users must log in again when their access token expires; the active lifetime requires configuration confirmation.
- Rate-limiting coverage requires confirmation against the current middleware; the claim that only public verification is limited has not been verified.
- No endpoint exists to delete an account.
- A production base URL, Cloudinary integration, and production CORS rules remain subject to confirmation.
- The repository URL, supported Node.js version, API documentation link, contribution guidelines, team list, and license remain publication placeholders.

For the full list, see Section 7 of the API Documentation.

## Documentation

- [VerifyDoc API Documentation](https://docs.google.com/document/d/1NfgcazIqYFcM340yJ98HqJiwMsUWTTgMxd40L1UGCgw/edit?usp=sharing)
- [Security Architecture Overview](https://docs.google.com/document/d/1XJ7mU80C--aRIkqZEewFGj0WDO3GHoFGTglIkPjNqc4/edit?usp=sharing)
- [Data Privacy Policy](https://docs.google.com/document/d/1yJWnVVkSDYWXy123EUupszCdg2USfeDRLqdr_1_rl80/edit?usp=sharing)
- [Citizen User Guide](https://docs.google.com/document/d/1sTqzy4zRkc2jMmii-lvlTfeo3RPbqS_41xG3aQN7Ibc/edit?usp=sharing)
- [Employer Verification Guide](https://docs.google.com/document/d/1aTwabIQMn57Goa16QQGm_jxfqCbLQ35r4N2UZPiMT28/edit?usp=sharing)
- [Institutions Onboarding Guide](https://docs.google.com/document/d/1uaduRzlzWMGfuIzgYmoo9AK7OT092FyZ7AdoN6OCEP8/edit?usp=sharing)
- [FAQ](https://docs.google.com/document/d/1oNyp9LjCD3I7EB7yw0zGpnv3BwPQYL3trb7_1n0GJjc/edit?usp=sharing)
- [Release Notes](https://docs.google.com/document/d/17Jv_hQmOwMNhncImGjpE4jOZL-GBC8HifVbSm7-5e-k/edit?usp=sharing)


## Team
DEVS:
- [Koto Temiloluwa John](https://github.com/John-2TB)
- [Ogunyemi Michael](https://github.com/Mikedave001)

WRITERS:
- [Chibuikem Iwuh](https://github.com/chibuikemiwuh5-svg)
- [Dada Marcus](https://github.com/Oluwafisayomi2280)


