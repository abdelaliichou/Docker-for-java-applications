# Keycloak Realm JSON Files — Purpose & Documentation

## Why do we have `appusers.json`, `awsusers.json`, and `o2s.json` in the tenant project?

These realm JSON files are **pre-configured Keycloak realms** that get auto-imported when you start the dev environment. They serve as a local identity provider so you can develop and test without connecting to a real Keycloak server.

---

## What each realm represents

| Realm | Purpose |
|-------|---------|
| `appusers.json` | The main realm for application end-users (`alice`, `axel`, `laure`) + service accounts for inter-service communication |
| `awsusers.json` | Realm for AWS/platform-level users (likely internal Harvest platform services) |
| `o2s.json` | Realm for O2S (another Harvest product/service) that interacts with the tenant API |

---

## Why they're in the project

1. **Local dev** — `docker-compose` imports them into Keycloak on startup (`--import-realm`), giving you a fully working auth setup with users, clients, roles, and permissions out of the box
2. **Integration tests** — The `test-component` also has realm files so tests can spin up a Keycloak devservice with the same config
3. **Self-contained** — Any developer can clone the repo, run `docker-compose up`, and have a working environment without manual Keycloak configuration

Essentially, it's the **"infrastructure as code"** approach for your local identity layer.

---

## Keycloak — Full Breakdown

### Core Concepts

#### Realm

A tenant/namespace in Keycloak. Everything lives inside a realm. Think of it as a completely isolated identity universe.

- `AppUsers` realm = your application's end-users
- `O2S` realm = another product's users
- `AWSUsers` realm = platform-level users

Each realm has its own users, clients, roles, flows — completely independent.

#### Users

Actual people (or service accounts) that can authenticate. A user belongs to one realm.

- `alice`, `axel`, `laure` = human users in AppUsers realm
- `service-account-tenant-manager` = a machine identity (no human behind it)

#### Clients

An application that wants to authenticate users or itself. Every app that talks to Keycloak is a client.

**Types:**

- **Public client** — frontend apps (SPA, mobile). Can't keep a secret. Uses PKCE.
  - Example: `tenant-swagger-client` (Swagger UI)
- **Confidential client** — backend services. Has a `client_secret`.
  - Example: `app-user-tenant-manager` (your backend calling Keycloak admin API)
- **Service account client** — machine-to-machine, no user involved. Uses `client_credentials` grant.
  - Example: `tenant-api-caller-service-account`

#### Roles

Permissions attached to users. Two levels:

- **Realm roles** — global across all clients in the realm (e.g., `admin`, `user`)
- **Client roles** — scoped to a specific client (e.g., `tenant-management/MANAGE_TENANTS`)

Your app uses `@RolesAllowed` annotations that check:

```
role-claim-path:
  - realm_access/roles
  - resource_access/tenant-management/roles
```

#### Groups

A way to organize users and assign roles in bulk. Instead of giving 500 users the same 5 roles individually, you put them in a group and assign roles to the group.

#### Attributes

Custom key-value metadata on users (e.g., `tenantId: 123`, `department: engineering`). Can be included in JWT tokens via mappers.

#### Identity Providers (IdP)

External authentication sources. Keycloak can federate login to:

- Google, Microsoft, GitHub (social login)
- Another Keycloak instance
- SAML/LDAP corporate directories

User clicks "Login with Google" → Keycloak delegates to Google → maps the user back into its realm.

#### Authentication Flows

The sequence of steps a user goes through to authenticate:

- Username + password
- Username + password + OTP (MFA)
- Browser redirect + consent
- Client credentials (no user interaction)

You can customize flows: require MFA for admins, allow passwordless for regular users, etc.

#### MFA (Multi-Factor Authentication)

A second factor beyond password:

- OTP (TOTP app like Google Authenticator)
- WebAuthn (hardware keys, biometrics)
- Email verification

Configured per-realm or per-group via authentication flows.

---

### How They Connect

```
Realm
├── Users (alice, axel)
│   ├── has Roles (admin, viewer)
│   ├── belongs to Groups (tenant-admins)
│   └── has Attributes (tenantId=123)
├── Clients (tenant-client, tenant-swagger-client)
│   ├── has Client Roles (MANAGE_TENANTS)
│   └── has Service Account (for machine-to-machine)
├── Identity Providers (Google, corporate LDAP)
├── Authentication Flows (browser, direct-grant, MFA)
└── Role Mappings (group → roles, user → roles)
```

---

## How Tenant Backend Uses Keycloak

1. User authenticates via Bruno/frontend → redirected to Keycloak login page
2. Keycloak validates credentials (+ MFA if configured)
3. Keycloak issues a JWT containing: user info, roles, attributes, tenantId
4. Bruno/frontend sends JWT to tenant-backend API
5. Quarkus OIDC filter validates the JWT signature against Keycloak's public keys
6. `@RolesAllowed` checks if the user has the required role
7. HVS context (`api-context` library) extracts tenantId from the JWT for multi-tenant logic

```
Browser/Bruno → Keycloak (login) → JWT → tenant-backend (validates + extracts roles/tenant)
```

---

## Local JSON Files vs. Production Keycloak

| | Local (JSON files) | Production |
|---|---|---|
| **Where** | Docker container on `localhost:9090` | Shared Keycloak cluster (`keycloak-k8s.aws-dev.harvest.fr`) |
| **Data** | Fake users (`alice/alice`), test clients, permissive config | Real users, real clients, strict security |
| **MFA** | Disabled | Likely enforced |
| **Realms** | Imported from JSON on startup | Managed by platform team, persistent |
| **Clients** | Hardcoded secrets (`secret`) | Rotated secrets, proper redirect URIs |
| **Purpose** | Dev/test without dependencies | Real authentication for real users |
| **Lifecycle** | Ephemeral (recreated with `docker-compose`) | Persistent, versioned, audited |

The JSON files are a snapshot/mock of what production looks like, simplified for local development. In prod, the Keycloak admin team manages realms, clients, and roles — your app just validates tokens against it.

---

## The Flow in One Sentence

> **Keycloak manages WHO can access WHAT — your app just receives a signed JWT and trusts it, checking roles to decide what the user is allowed to do.**
