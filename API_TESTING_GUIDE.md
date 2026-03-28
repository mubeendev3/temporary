# Noola API Testing Guide (Swagger / OpenAPI)

This guide matches **this repository** as implemented under Django **5.1**, Django REST Framework, **`djangorestframework-simplejwt`**, and **`drf-spectacular`**. It is written for beginners: you can follow it step by step with Swagger open beside this document.

---

## 1. Project API overview

### What exists in this project

| Piece | Role |
|--------|------|
| **`noola`** | Django project (settings, root `urls.py`). |
| **`accounts`** | Custom user model (`accounts.User`), OTP tables, email helpers, and **all HTTP APIs** exposed for this guide. |
| **`rest_framework`** | API framework (views, serializers, permissions). |
| **`drf_spectacular`** | Generates **OpenAPI schema** and **Swagger UI**. |
| **`django.contrib.admin`** | Admin site at `/admin/` (not covered in detail here). |
| Other apps in `INSTALLED_APPS` (`unfold`, `storages`, `corsheaders`, etc.) | Support admin, file storage, etc.; they do **not** add extra public REST endpoints in this repo’s `urls.py`. |

### Where the APIs live (base path)

All user/auth endpoints are mounted under:

- **Base URL (API prefix):** `/api/accounts/`

The router registers **`UserViewSet`** under the **`users`** prefix, so every endpoint path starts with:

- **`/api/accounts/users/`**

### Swagger and schema URLs (from your code)

These are defined in `noola/urls.py`:

| What | URL path | Notes |
|------|-----------|--------|
| **Swagger UI** | `/api/docs/` | Interactive API explorer. |
| **OpenAPI schema** | `/api/schema/` | Machine-readable description of all operations. |
| **Site root** | `/` | Redirects to **`/api/docs/`** (so opening the server root sends you to Swagger). |

There is **no** `/swagger/` route in this project; use **`/api/docs/`**.

### Typical full URLs when developing locally

If you run the dev server on the default port **8000**:

- Swagger: `http://127.0.0.1:8000/api/docs/`
- Schema: `http://127.0.0.1:8000/api/schema/`
- Accounts API: `http://127.0.0.1:8000/api/accounts/users/...`

(Replace host/port if you use something else.)

### What you will see grouped in Swagger

Swagger groups operations by **tags**. For this project, `drf-spectacular` will typically show one main group for the **`UserViewSet`** (often labeled in a way that reflects the **users** resource, e.g. **users** or **User**—the exact label can vary slightly by version, but all paths still start with `/api/accounts/users/`).

### Which operations belong to authentication / accounts

Everything under **`/api/accounts/users/`** is defined in `accounts/api/views.py` inside **`UserViewSet`**. That includes:

- Signup (OTP + completion)
- Login
- Password reset (OTP)
- CRUD-style user endpoints (with permissions)
- **`me`** (current profile)

---

## 2. Authentication flow (how it really works here)

### What kind of authentication this API uses

In `noola/settings.py`, DRF is configured with:

- **`rest_framework_simplejwt.authentication.JWTAuthentication`** as the **only** default authentication class.

That means:

- Protected endpoints expect a **JSON Web Token (JWT)** in the **`Authorization`** header.
- **Session login** and **DRF’s built-in `Token` model** are **not** used for these APIs.
- There is **no** custom API-key middleware in `MIDDLEWARE` for JWT; the token is read by DRF’s **`JWTAuthentication`** from the header.

`MIDDLEWARE` in `settings.py` is standard Django (including **`CsrfViewMiddleware`**). Your API views rely on **JWT**, not on **session** authentication, so you do **not** need a CSRF token for typical JWT calls from Swagger (CSRF mainly matters when using **session** authentication with browsers).

### How login works (conceptually)

1. You **POST** email + password to **`/api/accounts/users/login/`**.
2. The server checks credentials (`LoginSerializer` in `accounts/api/serializers.py`).
3. If valid, the server creates JWTs with **`RefreshToken.for_user(user)`** and returns:
   - **`access`** — short-lived JWT used for **`Authorization: Bearer ...`**
   - **`refresh`** — longer-lived refresh JWT (string form of the refresh token)
   - **`user`** — public user fields (`UserReadSerializer`)

The same **`access` / `refresh` / `user`** shape is returned when signup completes successfully (`signup_complete`).

### Where the token comes from and which field to copy

After **login** or **signup complete**, the JSON body includes:

```json
{
  "user": { "...": "..." },
  "access": "<YOUR_ACCESS_JWT>",
  "refresh": "<YOUR_REFRESH_JWT>"
}
```

For Swagger testing of protected routes, you need the **`access`** value (unless you add a refresh endpoint later—this project does **not** register SimpleJWT’s refresh URL in `noola/urls.py`, so renewing access is done by **logging in again** after the access token expires).

### Default token lifetime (not overridden in your `settings.py`)

This project does **not** define **`SIMPLE_JWT`** in `settings.py`, so **`djangorestframework-simplejwt`** defaults apply. By library default:

- **Access token** lifetime is **about 5 minutes**.
- **Refresh token** lifetime is **about 1 day** (still returned in JSON, but **no refresh HTTP endpoint** is wired in `urls.py` here).

If you get **401** suddenly, your **access** token may have expired—log in again.

### Exact header format to use

**Send:**

```http
Authorization: Bearer <access>
```

Replace `<access>` with the **full** string from the **`access`** field **without** extra quotes.

**Notes:**

- The word **`Bearer`** (with a space after it) is required.
- Do **not** paste the word `Bearer` into the token field twice. In Swagger’s **Authorize** dialog for HTTP bearer schemes, you usually paste **only the JWT**; Swagger adds the `Bearer ` prefix for you. If your Swagger build asks for the full value, use exactly: `Bearer ` + token.

### How to use Swagger’s **Authorize** button (from zero)

1. Call **login** (or complete signup) and **Execute** so you see the response body.
2. Find **`access`** in the JSON. Select and **copy** the long string (starts with `eyJ` typically).
3. In Swagger UI, click **Authorize** (lock icon).
4. For **JWT / Bearer** authentication, paste the **`access`** token into the token field (per the UI hint: often **only the token**, not the word Bearer).
5. Click **Authorize**, then **Close**.
6. Try a protected endpoint again (e.g. **GET** `/api/accounts/users/me/`). Swagger will attach the header automatically.

---

## 3. Step zero: how to start testing APIs in Swagger

### Open Swagger

1. Start your Django server (however you usually run it, e.g. `python manage.py runserver`).
2. In a browser, open **`http://127.0.0.1:8000/api/docs/`** (or your host/port).

### Find an endpoint

- Scroll the page or use the filter/search box if your Swagger UI provides one.
- Endpoints are grouped (tags). Expand a group to see methods (**GET**, **POST**, etc.).

### Try a request

1. Click an operation (e.g. **POST** `/api/accounts/users/login/`).
2. Click **Try it out**.
3. **Parameters**: path/query fields appear at the top; fill them if required.
4. **Request body**: edit the JSON example (Swagger shows a schema). Use valid JSON (double quotes on keys and string values).
5. Click **Execute**.

### Read the result

- **Code** (e.g. **200**, **201**, **400**, **401**): HTTP **status code**.
- **Response body**: JSON (or HTML for rare server errors).
- **Response headers**: metadata (e.g. content type).

### Quick meaning of common status codes

| Code | Meaning (simplified) |
|------|----------------------|
| **200** | OK; read success or action succeeded with a normal body. |
| **201** | Created; something new was stored (e.g. new user on signup complete). |
| **204** | Success with no body (not heavily used in your `UserViewSet` responses shown). |
| **400** | Bad request—often validation errors or business rule failure (wrong OTP, etc.). |
| **401** | Not authenticated—missing/invalid/expired JWT for a protected route. |
| **403** | Authenticated but not allowed—e.g. not staff for admin-only **create**. |
| **404** | Not found—wrong URL or **user_id** not in your allowed queryset. |
| **500** | Server error—bug or misconfiguration. |

### Copy as cURL (optional)

Swagger often shows a **cURL** command after execution. You can copy it into a terminal to repeat the same request outside the browser.

---

## 4. API-by-API testing guide

Below, **`{user_id}`** means a **UUID** string—the public identifier on `User.user_id` (not the integer database primary key). Example shape: `3fa85f64-5717-4562-b3fc-2c963f66afa6`.

**Content-Type:** For JSON bodies, use **`application/json`** (Swagger usually sets this automatically).

---

### POST `/api/accounts/users/signup/request-otp/`

#### Purpose

Starts email signup: if the email is **not** already registered, the server creates a **6-digit OTP** (valid **15 minutes**, see `accounts/constants.py`) and emails it. If the email **already exists**, the API still responds **200** with the **same generic message** (it does not reveal whether the email is taken).

#### Authentication required?

**No** (`AllowAny`).

#### Prerequisites

- A real inbox **or** (in **`DEBUG`**) watch the **runserver console**: email backend defaults to **console** in development (`settings.py`), so the OTP text is printed there instead of sending real mail when `EMAIL_BACKEND` is not overridden.

#### Request headers

- No auth header.

#### Request body (serializer: `SignupRequestOTPSerializer`)

| Field | Type | Required |
|-------|------|----------|
| `email` | string (email) | Yes |

#### Example request

```json
{
  "email": "new.user@example.com"
}
```

#### Expected success response

**200 OK**

```json
{
  "detail": "If this email can be used for registration, a verification code has been sent."
}
```

(Exact wording is in `UserViewSet._SIGNUP_OTP_SENT`.)

#### Common errors

- **400** — invalid email format (`{"email": ["Enter a valid email address."]}` style).

#### How to fix

- Use a valid email string in JSON.

---

### POST `/api/accounts/users/signup/verify-otp/`

#### Purpose

Checks the **signup** OTP for the email. If valid, the OTP is **consumed** (one-time) and the server returns a **`signup_token`** (signed, time-limited; default max age **30 minutes** via `SIGNUP_COMPLETION_TOKEN_MAX_AGE_SECONDS`).

#### Authentication required?

**No**.

#### Prerequisites

- You must have called **signup/request-otp** for the same email and received the **6-digit** code.

#### Request body (`SignupVerifyOTPSerializer`)

| Field | Type | Required |
|-------|------|----------|
| `email` | string | Yes |
| `code` | string (max 32 chars) | Yes — use the **6-digit** OTP |

#### Example request

```json
{
  "email": "new.user@example.com",
  "code": "123456"
}
```

#### Expected success response

**200 OK**

```json
{
  "signup_token": "<signed_token_string>",
  "detail": "Email verified. Submit this token with your password and profile to complete signup."
}
```

#### Common errors

- **400** — `{"detail": "Invalid or expired verification code."}` (wrong code, expired, too many failures, or already used).

#### How to fix

- Re-request OTP with **signup/request-otp** if expired or lost.
- Enter the **latest** code carefully (OTP length is **6** digits from `accounts/services/otp.py`).
- After **5** wrong attempts for that OTP row, verification can fail until a **new** OTP is issued (`OTP_MAX_FAILED_ATTEMPTS`).

---

### POST `/api/accounts/users/signup/complete/`

#### Purpose

Creates the **User** with **`is_verified=True`**, using the **`signup_token`** to recover the verified **email**. Returns **JWT** **`access`** + **`refresh`** and **`user`** profile.

#### Authentication required?

**No**.

#### Prerequisites

- Valid **`signup_token`** from **signup/verify-otp** (not expired; not tampered with).
- Email must **not** already exist as a user.

#### Request body (`SignupCompleteSerializer`)

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `signup_token` | string | Yes | From previous step |
| `password` | string | Yes | Min length **8** |
| `first_name` | string | Yes | Model required field |
| `last_name` | string | Yes | Model required field |
| `phone_number` | string | No | |
| `company_name` | string | No | |
| `company_role` | string | No | |

#### Example request

```json
{
  "signup_token": "<paste_signup_token_here>",
  "password": "securepass123",
  "first_name": "Ada",
  "last_name": "Lovelace",
  "phone_number": "+15551234567",
  "company_name": "Noola",
  "company_role": "Engineer"
}
```

#### Expected success response

**201 Created**

```json
{
  "user": {
    "user_id": "...",
    "email": "new.user@example.com",
    "first_name": "Ada",
    "last_name": "Lovelace",
    "full_name": "Ada Lovelace",
    "avatar": null,
    "phone_number": "+15551234567",
    "company_name": "Noola",
    "company_role": "Engineer",
    "is_google_user": false,
    "two_factor_enabled": false,
    "notifications_config": { "...": "..." },
    "ui_preferences": { "...": "..." },
    "subscription_status": null,
    "is_active": true,
    "is_staff": false,
    "created_at": "...",
    "updated_at": "...",
    "is_verified": true
  },
  "access": "<jwt_access>",
  "refresh": "<jwt_refresh>"
}
```

(`notifications_config` / `ui_preferences` default shapes come from `accounts/constants.py`.)

#### Common errors

- **400** — `signup_token` errors:
  - `"This signup link has expired."` (signature expired)
  - `"Invalid signup token."` (bad signature)
  - `"An account with this email already exists."`
- **400** — field validation (e.g. password too short).

#### How to fix

- Run **verify-otp** again to get a **fresh** `signup_token` if expired.
- Do not reuse a token after the account already exists.

---

### POST `/api/accounts/users/login/`

#### Purpose

Authenticates with **email** + **password**; returns **`access`**, **`refresh`**, and **`user`**.

#### Authentication required?

**No**.

#### Prerequisites

- A user that already exists (e.g. after **signup/complete** or superuser created via `createsuperuser`).

#### Request body (`LoginSerializer`)

| Field | Type | Required |
|-------|------|----------|
| `email` | string | Yes |
| `password` | string | Yes |

#### Example request

```json
{
  "email": "new.user@example.com",
  "password": "securepass123"
}
```

#### Expected success response

**200 OK** — same token + user shape as signup complete (`access`, `refresh`, `user`).

#### Common errors

- **400** — `{"non_field_errors": ["Invalid email or password."]}` for wrong credentials.
- **400** — `["User account is disabled."]` if `is_active` is false.

#### How to fix

- Verify email/password.
- For testing, ensure the user is active.

---

### POST `/api/accounts/users/password-reset/request/`

#### Purpose

If an account exists for the email, sends a **password reset** OTP (same **6-digit**, **15-minute** validity). Always returns **200** with a **generic message** whether or not the user exists (no email enumeration).

#### Authentication required?

**No**.

#### Prerequisites

- User must already exist for an email to actually receive/use a code.

#### Request body (`PasswordResetRequestSerializer`)

| Field | Type | Required |
|-------|------|----------|
| `email` | string | Yes |

#### Example request

```json
{
  "email": "existing@example.com"
}
```

#### Expected success response

**200 OK**

```json
{
  "detail": "If an account exists for this email, a password reset code has been sent."
}
```

#### Common errors

- **400** — invalid email format.

#### How to fix

- Use valid JSON email.

---

### POST `/api/accounts/users/password-reset/confirm/`

#### Purpose

Validates **reset** OTP, sets **new_password**, consumes OTP.

#### Authentication required?

**No**.

#### Prerequisites

- **password-reset/request** succeeded for an existing user.
- Valid **code** in time; not locked out by failed attempts.

#### Request body (`PasswordResetConfirmSerializer`)

| Field | Type | Required |
|-------|------|----------|
| `email` | string | Yes |
| `code` | string | Yes |
| `new_password` | string | Yes, min **8** chars |

#### Example request

```json
{
  "email": "existing@example.com",
  "code": "654321",
  "new_password": "anothersecure1"
}
```

#### Expected success response

**200 OK**

```json
{
  "detail": "Your password has been reset. You can sign in now."
}
```

#### Common errors

- **400** — `{"detail": "Invalid or expired reset code."}` (includes missing user edge case mapped to same message in code).

#### How to fix

- Request a new code with **password-reset/request**.
- Use the latest OTP; avoid typos; watch lockout after failed attempts.

---

### GET `/api/accounts/users/me/`

#### Purpose

Returns the **current** authenticated user as **`UserReadSerializer`**.

#### Authentication required?

**Yes** — JWT **`access`** (`IsAuthenticated`).

#### Prerequisites

- Valid **`Authorization: Bearer <access>`** (via Swagger Authorize).

#### Request headers

```http
Authorization: Bearer <access>
```

#### Request body

None.

#### Example request

No body; ensure **Authorize** is set.

#### Expected success response

**200 OK** — single user object (same fields as in **user** from login).

#### Common errors

- **401** — missing/invalid/expired token.

#### How to fix

- Log in again; re-authorize Swagger with new **`access`**.

---

### PATCH `/api/accounts/users/me/`

#### Purpose

Partially updates the **current** user (`UserUpdateSerializer`, `partial=True`). Optional **`password`** updates credentials if provided.

#### Authentication required?

**Yes**.

#### Prerequisites

- Valid JWT.

#### Request headers

```http
Authorization: Bearer <access>
```

#### Request body (`UserUpdateSerializer` — all optional for PATCH)

Writable fields include:

- `first_name`, `last_name`, `avatar`, `phone_number`, `company_name`, `company_role`
- `two_factor_enabled`, `notifications_config`, `ui_preferences`, `subscription_status`
- `password` (optional; min **8** if sent)

#### Example request (JSON-only; no file)

```json
{
  "first_name": "Ada",
  "phone_number": "+15550001111",
  "notifications_config": {
    "email": true,
    "consumption_alert": true,
    "bill_reminder": true,
    "marketplace_offers": false,
    "new_recommendation": true,
    "product_news": false
  }
}
```

**Note:** Updating **`avatar`** is an **image** field on the model. In Swagger you may need **multipart/form-data** for file uploads depending on how the UI renders it. If you only send JSON, skip `avatar` or use the appropriate file upload control if shown.

#### Expected success response

**200 OK** — updated user via `UserReadSerializer`.

#### Common errors

- **401** — auth issues.
- **400** — validation (e.g. password too short).

#### How to fix

- Match serializer constraints; use PATCH with only fields you want to change.

---

### GET `/api/accounts/users/`

#### Purpose

**List** users. **Staff** users see **all** users; **non-staff** authenticated users only see **themselves** (queryset filtered in `get_queryset`).

#### Authentication required?

**Yes** (`IsAuthenticated`). Not public.

#### Prerequisites

- JWT.

#### Request headers

```http
Authorization: Bearer <access>
```

#### Request body

None.

#### Expected success response

**200 OK** — paginated **list** format depends on DRF settings (this project does not override pagination in `settings.py`, so you may get a **JSON array** of users or a default paginated object if you add pagination later). As of current `settings.py`, expect a plain list from the viewset.

#### Common errors

- **401** — not logged in.

#### How to fix

- Authorize with **`access`**.

---

### POST `/api/accounts/users/`

#### Purpose

**Admin-only** user creation (`IsAdminUser`). Creates a user with **`UserCreateSerializer`** (includes **`password`**).

#### Authentication required?

**Yes**, and the user must have **`is_staff`** true (staff).

#### Prerequisites

- Staff JWT (e.g. from a superuser login).

#### Request headers

```http
Authorization: Bearer <access>
```

#### Request body (`UserCreateSerializer`)

| Field | Type | Notes |
|-------|------|--------|
| `email` | string | Unique |
| `password` | string | Min **8** |
| `first_name`, `last_name` | string | Required by model |
| `avatar` | file | Optional |
| `phone_number`, `company_name`, `company_role` | string | Optional |
| `is_google_user` | boolean | Optional |
| `two_factor_enabled` | boolean | Optional |
| `notifications_config`, `ui_preferences` | object | Optional JSON |
| `subscription_status` | string | Optional (`active` / `inactive` / `trial`) |
| `is_active`, `is_staff` | boolean | Optional |

#### Example request (minimal JSON)

```json
{
  "email": "admin.created@example.com",
  "password": "longpassword1",
  "first_name": "Test",
  "last_name": "User",
  "is_active": true,
  "is_staff": false
}
```

#### Expected success response

**201 Created** — `UserReadSerializer` payload for the new user.

#### Common errors

- **403** — not staff.
- **401** — not authenticated.
- **400** — validation (duplicate email, etc.).

#### How to fix

- Use a staff account; fix validation errors shown in response.

---

### GET `/api/accounts/users/{user_id}/`

#### Purpose

Retrieve one user by **public UUID** (`lookup_field = "user_id"`).

#### Authentication required?

**Yes**.

#### Access rules

- **Staff:** any user.
- **Non-staff:** only **your own** `user_id` (otherwise queryset returns nothing → **404**).

#### Request headers

```http
Authorization: Bearer <access>
```

#### Path parameters

| Param | Description |
|--------|-------------|
| `user_id` | UUID string |

#### Example

`GET /api/accounts/users/3fa85f64-5717-4562-b3fc-2c963f66afa6/`

#### Expected success response

**200 OK** — `UserReadSerializer` object.

#### Common errors

- **404** — not found or not allowed (hidden as not found).
- **401** — no/invalid token.

#### How to fix

- Copy **`user_id`** from your **`/me/`** or login **`user`** object.
- Use staff account to read others.

---

### PUT `/api/accounts/users/{user_id}/`

#### Purpose

Full update with **`UserUpdateSerializer`**. In DRF, **PUT** is **not** partial by default, so typically **all serializer fields** are expected unless marked optional in the serializer. Practically, prefer **PATCH** for partial updates to avoid having to send every field.

#### Authentication required?

**Yes**.

#### Prerequisites

- Permission to access that `user_id` (same as GET: self or staff).

#### Request headers

```http
Authorization: Bearer <access>
```

#### Request body

All `UserUpdateSerializer` fields (see PATCH section). **`password`** optional.

**Note:** **`avatar`** may require multipart if you set a file.

#### Common errors

- **400** validation, **404**, **401**, **403** (staff-only create does not apply here, but 403 can still appear in edge permission cases).

#### How to fix

- Prefer **PATCH** for simpler testing, or supply a complete valid body for **PUT**.

---

### PATCH `/api/accounts/users/{user_id}/`

#### Purpose

Partial update of the given user (same serializer as PUT, partial update).

#### Authentication required?

**Yes**.

#### Prerequisites

- Access to that `user_id` (self or staff).

#### Request headers

```http
Authorization: Bearer <access>
```

#### Request body

Subset of update fields (example):

```json
{
  "company_name": "Acme"
}
```

#### Expected success response

**200 OK** — updated `UserReadSerializer`.

#### Common errors

Same family as PUT.

---

### DELETE `/api/accounts/users/{user_id}/`

#### Purpose

Deletes the user row (default `ModelViewSet` **destroy**).

#### Authentication required?

**Yes**.

#### Access rules

- Object must be visible in **`get_queryset`**: non-staff users effectively only **self**; staff can delete users they can list.

#### Request headers

```http
Authorization: Bearer <access>
```

#### Expected success response

**204 No Content** (DRF default for successful delete).

#### Common errors

- **404** / **401**.

#### How to fix

- Ensure correct `user_id` and permissions; be careful—this removes the account.

---

## 5. API testing order (recommended)

Follow this order so each step has the data the next step needs.

1. **POST** `/api/accounts/users/signup/request-otp/` — send email; read OTP (**console** in `DEBUG` or inbox).
2. **POST** `/api/accounts/users/signup/verify-otp/` — email + code → copy **`signup_token`**.
3. **POST** `/api/accounts/users/signup/complete/` — token + password + names → copy **`access`** (and note **`user.user_id`**).
4. In Swagger, **Authorize** with **`access`**.
5. **GET** `/api/accounts/users/me/` — confirm protected access works.
6. **PATCH** `/api/accounts/users/me/` — try a small profile change.
7. **GET** `/api/accounts/users/` — see yourself (non-staff) or everyone (staff).
8. **GET** `/api/accounts/users/{user_id}/` — use your UUID from step 3.
9. **PATCH** `/api/accounts/users/{user_id}/` — optional alternative to `/me/`.
10. **POST** `/api/accounts/users/login/` — test login path (same tokens as signup complete).
11. **POST** `/api/accounts/users/password-reset/request/` → **password-reset/confirm/** — only for **existing** accounts.

**Admin-only (after creating a superuser in the shell):**

12. **POST** `/api/accounts/users/` — create another user as staff.

---

## 6. Troubleshooting

| Symptom | Likely cause | What to do |
|---------|----------------|------------|
| **401 Unauthorized** on `/me/` | Missing header, wrong token, expired **access** | Click **Authorize** again; **login** to get fresh **`access`** (~5 min default lifetime). |
| **403 Forbidden** on **POST** `/api/accounts/users/` | Not **staff** | Log in as `is_staff` superuser or staff user. |
| **404** on **GET** `/api/accounts/users/{user_id}/` | Wrong UUID or viewing someone else’s ID as non-staff | Use your **`user_id`** from **`/me/`** or login response. |
| Swagger sends no **Authorization** | Did not authorize or cleared session | Re-open **Authorize** and paste **`access`**. |
| Double **Bearer** | Pasted `Bearer eyJ...` into a field that already adds Bearer | Paste **only** the JWT (or follow the placeholder text exactly). |
| **400** with `Invalid or expired verification code` | OTP wrong, expired, used, or locked | Request a **new** OTP; enter the **new** code within **15 minutes**. |
| **400** on signup token | Token **older than 30 minutes** (default) or altered | Run **verify-otp** again for a new **`signup_token`**. |
| **400** `Invalid email or password` on login | Wrong credentials or inactive user | Reset password flow or check `is_active`. |
| OTP email “not received” | Console backend in **`DEBUG`** | Read **terminal** where `runserver` runs; OTP is printed there. |
| CSRF errors | Rare for pure JWT from Swagger on same origin | Ensure you are not mixing **session** auth; use JWT as documented. |
| CORS errors from a separate frontend | `corsheaders` is installed but **`CorsMiddleware`** is **not** added in `settings.py` in this repo | For browser apps on another origin, configure CORS in settings (outside Swagger same-origin testing). |

---

## 7. Real testing walkthrough example

**Goal:** New user signs up with OTP, gets JWT, authorizes Swagger, and reads **`/me/`**.

1. Open **`http://127.0.0.1:8000/api/docs/`**.
2. **POST** `/api/accounts/users/signup/request-otp/`  
   Body: `{"email": "demo@example.com"}` → **Execute** → **200**.
3. Read **6-digit** code from **runserver console** (if `DEBUG` and console email backend).
4. **POST** `/api/accounts/users/signup/verify-otp/`  
   Body: `{"email": "demo@example.com", "code": "######"}` → **200** → copy **`signup_token`**.
5. **POST** `/api/accounts/users/signup/complete/`  
   Body includes `signup_token`, `password` (8+ chars), `first_name`, `last_name` → **201** → copy **`access`**.
6. Click **Authorize** in Swagger → paste **`access`** → **Authorize** → **Close**.
7. **GET** `/api/accounts/users/me/` → **Execute** → **200** with your profile.

You can then **PATCH** `/api/accounts/users/me/` with a small JSON change to confirm updates.

---

## Appendix: where this behavior is defined in code

| Topic | Location |
|--------|-----------|
| Swagger + schema routes | `noola/urls.py` |
| JWT as default auth | `noola/settings.py` → `REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"]` |
| All endpoints + permissions | `accounts/api/views.py` → `UserViewSet` |
| Request/response shapes | `accounts/api/serializers.py` |
| Router (`/users/...`) | `accounts/urls.py` |
| OTP rules | `accounts/constants.py`, `accounts/services/otp.py` |
| Signup token signing | `accounts/services/signup_token.py` |
| Email sending | `accounts/services/email.py` |

This guide is accurate for the code in this repository; if you add routes (e.g. SimpleJWT **token refresh**), update the **Authentication** and **Testing order** sections to match.
