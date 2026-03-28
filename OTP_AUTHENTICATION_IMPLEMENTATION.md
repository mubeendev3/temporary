# OTP-Based Authentication — Implementation Guide (Noola Backend)

This document is tailored to **this repository**: Django **5.1**, DRF, **custom `accounts.User`** (`USERNAME_FIELD = "email"`), **`djangorestframework-simplejwt`** already used in `accounts/api/views.py`, and OpenAPI via **`drf-spectacular`** at `/api/docs/`.

You will implement everything yourself by following the steps below.

---

## 0. Current project snapshot (what exists today)

| Area | Location | Notes |
|------|----------|--------|
| User model | `accounts/models.py` | `AbstractBaseUser`; `email` unique; **`is_verified`** already exists (default `False`). |
| Auth API | `accounts/api/views.py` | `UserViewSet`; **`login`** returns JWT access + refresh; **`signup`** `@action` is **commented out**. |
| Serializers | `accounts/api/serializers.py` | `SignupSerializer`, `LoginSerializer`, CRUD serializers. |
| URLs | `accounts/urls.py` + `noola/urls.py` | Router: `api/accounts/users/`. |
| Settings | `noola/settings.py` | `AUTH_USER_MODEL = "accounts.User"`; **`REST_FRAMEWORK`** only sets `DEFAULT_SCHEMA_CLASS` (no global JWT auth class yet). |
| Permissions | `accounts/api/permissions.py` | Empty file. |

**Implication for you:** After OTP flows return JWTs, protected endpoints (e.g. `GET/PATCH /api/accounts/users/me/`) only work with a Bearer token if you add **`JWTAuthentication`** to DRF defaults (see §6 and the summary).

---

## 1. Overview

### 1.1 What you are building

A **database-backed OTP** system: short-lived codes sent by **email**, tied to a **purpose** (signup verification vs password reset), with **expiry**, **single use**, and **safe storage** (hashed codes).

### 1.2 Use case A — Email verification during signup

1. User submits registration data (email, password, profile fields).
2. Backend creates the user (initially **unverified**) and stores a **hashed** OTP linked to that user and purpose `SIGNUP`.
3. User receives email with the **plain** code (only in the email, not stored in DB).
4. User submits email + code; backend verifies, marks OTP used, sets **`User.is_verified = True`**, returns **JWT** (optional but matches your existing `login` response shape).

### 1.3 Use case B — OTP-based password reset

1. User requests reset with **email**.
2. Backend issues OTP with purpose `PASSWORD_RESET` (for an **existing** user).
3. User submits email + OTP + **new password**; backend verifies OTP, sets new password, marks OTP used, invalidates concern.

---

## 2. Architecture decisions

### 2.1 Recommended approach: database-driven OTP

Store OTP **metadata** in Postgres (via your existing `DATABASE_URL`), not only in memory:

- **Auditable**: you can inspect rows in admin / DB during development.
- **Fits your stack**: no extra infrastructure required for the learning path (Redis optional later for rate limits).
- **Aligns with `User.is_verified`**: signup verification is a persistent state on `accounts.User`.

### 2.2 Why this fits Noola’s `accounts` app

- You already have **`is_verified`** on `User` — OTP verification should flip this flag.
- Auth endpoints naturally live next to **`UserViewSet`** in `accounts/api/` (same router, consistent URLs).
- You already depend on **Celery + Redis** in `requirements.txt`; later you can move **send email** and **cleanup** async without changing the core model design.

### 2.3 Security choices (learning → production)

| Topic | Recommendation |
|--------|------------------|
| Store OTP in DB | Store **hashed** code (like passwords), verify with `check_password`. |
| Plain code | Only in email (and optionally logs in DEBUG). |
| Guessing | 6-digit numeric OTP + short expiry (e.g. 15 min) + rate limits. |
| Old OTPs | When issuing a new OTP for same user + purpose, **invalidate** previous active rows. |

---

## 3. Step-by-step implementation guide

Each step names **which file** to touch, **what** to build, and **why**.

---

### Step 3a — OTP model design

**File:** `accounts/models.py`  
**Why:** Single source of truth for OTP lifecycle (expiry, used, purpose).

**What:** Add an `EmailOTP` model (name can vary; keep it in `accounts`).

**Full implementation** (append to `accounts/models.py`; adjust imports at top if needed):

```python
import secrets
import string
from django.conf import settings
from django.contrib.auth.hashers import check_password, make_password
from django.db import models
from django.utils import timezone


class EmailOTPPurpose(models.TextChoices):
    SIGNUP = "signup", "Signup email verification"
    PASSWORD_RESET = "password_reset", "Password reset"


class EmailOTP(models.Model):
    """
    One row per issued OTP. Code is stored hashed; plain code exists only in email.
    """

    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="email_otps",
    )
    purpose = models.CharField(
        max_length=32,
        choices=EmailOTPPurpose.choices,
        db_index=True,
    )
    code_hash = models.CharField(max_length=128)
    created_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField(db_index=True)
    used_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        indexes = [
            models.Index(fields=["user", "purpose", "-created_at"]),
        ]

    def __str__(self):
        return f"EmailOTP(user_id={self.user_id}, purpose={self.purpose})"

    @property
    def is_expired(self):
        return timezone.now() >= self.expires_at

    @property
    def is_used(self):
        return self.used_at is not None

    @classmethod
    def generate_plain_code(cls, length: int = 6) -> str:
        digits = string.digits
        return "".join(secrets.choice(digits) for _ in range(length))

    @classmethod
    def hash_code(cls, plain_code: str) -> str:
        return make_password(plain_code)

    def verify_code(self, plain_code: str) -> bool:
        if self.is_used or self.is_expired:
            return False
        return check_password(plain_code, self.code_hash)

    def mark_used(self):
        self.used_at = timezone.now()
        self.save(update_fields=["used_at"])
```

**Then run:**

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

**Why these fields**

- **`user` FK**: signup and reset both target a real `User` (signup creates the user first — see §3d).
- **`purpose`**: one table, two flows, clear queries.
- **`expires_at` / `used_at`**: explicit state machine without overloading `User`.
- **`code_hash`**: avoids leaking valid OTPs if the DB is copied.

---

### Step 3b — OTP generation and validation logic (service layer)

**File (create):** `accounts/services/__init__.py` (empty or with docstring)  
**File (create):** `accounts/services/otp.py`  
**Why:** Keeps views thin; easier to test and to swap email sending later (Celery).

**Full implementation — `accounts/services/__init__.py`:**

```python
# Package marker for OTP/email services.
```

**Full implementation — `accounts/services/otp.py`:**

```python
from datetime import timedelta

from django.conf import settings
from django.db import transaction
from django.utils import timezone

from accounts.models import EmailOTP, EmailOTPPurpose


def otp_ttl() -> timedelta:
    return timedelta(minutes=getattr(settings, "OTP_EXPIRY_MINUTES", 15))


@transaction.atomic
def issue_otp(user, purpose: str) -> str:
    """
    Invalidate previous active OTPs for this user+purpose, create a new one, return PLAIN code.
    Caller is responsible for sending email and never logging the code in production.
    """
    now = timezone.now()
    ttl = otp_ttl()
    expires_at = now + ttl

    # Invalidate older unused, unexpired OTPs for same user + purpose
    (
        EmailOTP.objects.filter(
            user=user,
            purpose=purpose,
            used_at__isnull=True,
            expires_at__gt=now,
        ).update(expires_at=now)
    )

    plain = EmailOTP.generate_plain_code()
    EmailOTP.objects.create(
        user=user,
        purpose=purpose,
        code_hash=EmailOTP.hash_code(plain),
        expires_at=expires_at,
    )
    return plain


def get_latest_valid_otp(user, purpose: str):
    now = timezone.now()
    return (
        EmailOTP.objects.filter(
            user=user,
            purpose=purpose,
            used_at__isnull=True,
            expires_at__gt=now,
        )
        .order_by("-created_at")
        .first()
    )


def verify_otp(user, purpose: str, plain_code: str) -> EmailOTP | None:
    otp = get_latest_valid_otp(user, purpose)
    if otp is None:
        return None
    if otp.verify_code(plain_code):
        return otp
    return None
```

**File:** `noola/settings.py`  
**Why:** Tunable expiry without code changes.

Add near other app settings:

```python
OTP_EXPIRY_MINUTES = env.int("OTP_EXPIRY_MINUTES", default=15)
```

(If you prefer not to use `env.int` yet, use a plain integer `OTP_EXPIRY_MINUTES = 15`.)

---

### Step 3c — Email sending setup and integration

**File:** `noola/settings.py`  
**Why:** Django’s email framework needs a backend; console backend is ideal for local learning.

Add (merge with existing settings; avoid duplicating keys):

```python
# Email — development default: print to console
EMAIL_BACKEND = env(
    "EMAIL_BACKEND",
    default="django.core.mail.backends.console.EmailBackend",
)

DEFAULT_FROM_EMAIL = env("DEFAULT_FROM_EMAIL", default="noreply@localhost")
```

For a real SMTP provider later, set in `.env`:

```env
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_HOST_USER=...
EMAIL_HOST_PASSWORD=...
EMAIL_USE_TLS=True
DEFAULT_FROM_EMAIL=Noola <noreply@yourdomain.com>
```

**File (create):** `accounts/emails.py`  
**Why:** Central place for subject/body; views call this.

```python
from django.conf import settings
from django.core.mail import send_mail


def send_otp_email(to_email: str, purpose: str, plain_code: str) -> None:
    if purpose == "signup":
        subject = "Verify your Noola account"
        message = (
            f"Your verification code is: {plain_code}\n\n"
            f"It expires in {getattr(settings, 'OTP_EXPIRY_MINUTES', 15)} minutes."
        )
    elif purpose == "password_reset":
        subject = "Reset your Noola password"
        message = (
            f"Your password reset code is: {plain_code}\n\n"
            f"It expires in {getattr(settings, 'OTP_EXPIRY_MINUTES', 15)} minutes."
        )
    else:
        subject = "Your security code"
        message = f"Your code is: {plain_code}"

    send_mail(
        subject,
        message,
        settings.DEFAULT_FROM_EMAIL,
        [to_email],
        fail_silently=False,
    )
```

**Why `send_mail`:** Built-in, no new dependencies; matches Django docs and is easy to replace with Celery tasks later.

---

### Step 3d — Signup flow with OTP

**Design choice for this repo:** Create the **`User` immediately** with **`is_verified=False`**, then send signup OTP. This reuses your existing `User.objects.create_user` path and maps cleanly to **`is_verified`**.

**File:** `accounts/api/serializers.py`  
**What:** Adjust signup so the created user is unverified; add serializers for verify / forgot / reset.

**Add imports:**

```python
from django.contrib.auth import get_user_model
from rest_framework import serializers

from accounts.models import EmailOTPPurpose

User = get_user_model()
```

**Replace or adjust `SignupSerializer.create`** so new users are not verified yet:

```python
class SignupSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)

    class Meta:
        model = User
        fields = (
            "email",
            "password",
            "first_name",
            "last_name",
            "phone_number",
            "company_name",
            "company_role",
        )

    def validate_email(self, value):
        value = value.lower().strip()
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError("A user with this email already exists.")
        return value

    def create(self, validated_data):
        password = validated_data.pop("password")
        user = User.objects.create_user(password=password, **validated_data)
        user.is_verified = False
        user.save(update_fields=["is_verified"])
        return user
```

**Add serializers:**

```python
class VerifyEmailSerializer(serializers.Serializer):
    email = serializers.EmailField()
    code = serializers.CharField(max_length=12, trim_whitespace=True)

    def validate_email(self, value):
        return value.lower().strip()


class ForgotPasswordSerializer(serializers.Serializer):
    email = serializers.EmailField()

    def validate_email(self, value):
        return value.lower().strip()


class ResetPasswordSerializer(serializers.Serializer):
    email = serializers.EmailField()
    code = serializers.CharField(max_length=12, trim_whitespace=True)
    new_password = serializers.CharField(write_only=True, min_length=8)

    def validate_email(self, value):
        return value.lower().strip()
```

**File:** `accounts/api/views.py`  
**What:** Uncomment/adapt **`signup`**: after `serializer.save()`, call `issue_otp` + `send_otp_email`. Add **`verify_email`**, **`forgot_password`**, **`reset_password`** actions.

**Imports to add:**

```python
from accounts.emails import send_otp_email
from accounts.models import EmailOTPPurpose
from accounts.services.otp import issue_otp, verify_otp
```

**Register new serializers in `get_serializer_class`:**

```python
from .serializers import (
    # ... existing ...
    VerifyEmailSerializer,
    ForgotPasswordSerializer,
    ResetPasswordSerializer,
)
```

Extend the method:

```python
    def get_serializer_class(self):
        if self.action in ["list", "retrieve", "me"]:
            return UserReadSerializer
        if self.action in ["create", "admin_create"]:
            return UserCreateSerializer
        if self.action in ["update", "partial_update"]:
            return UserUpdateSerializer
        if self.action == "signup":
            return SignupSerializer
        if self.action == "login":
            return LoginSerializer
        if self.action == "verify_email":
            return VerifyEmailSerializer
        if self.action == "forgot_password":
            return ForgotPasswordSerializer
        if self.action == "reset_password":
            return ResetPasswordSerializer
        return UserReadSerializer
```

**Extend `get_permissions`** so new actions are public:

```python
    def get_permissions(self):
        if self.action in ["signup", "login", "verify_email", "forgot_password", "reset_password"]:
            return [AllowAny()]
        if self.action == "create":
            return [IsAdminUser()]
        return [IsAuthenticated()]
```

**Implement `signup`** (issue OTP after user creation; **do not** return tokens until verified):

```python
    @action(detail=False, methods=["post"], permission_classes=[AllowAny])
    def signup(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        plain = issue_otp(user, EmailOTPPurpose.SIGNUP)
        send_otp_email(user.email, EmailOTPPurpose.SIGNUP, plain)
        return Response(
            {
                "detail": "Signup successful. Check your email for a verification code.",
                "user_id": str(user.user_id),
                "email": user.email,
            },
            status=status.HTTP_201_CREATED,
        )
```

**Why no JWT on signup:** Avoids granting API access before email ownership is proven; matches **`is_verified`** semantics.

---

### Step 3e — OTP verification flow (email verification)

**Still in** `accounts/api/views.py`.

```python
    @action(detail=False, methods=["post"], permission_classes=[AllowAny])
    def verify_email(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        email = serializer.validated_data["email"]
        code = serializer.validated_data["code"]

        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return Response(
                {"detail": "Invalid code or email."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        otp = verify_otp(user, EmailOTPPurpose.SIGNUP, code)
        if otp is None:
            return Response(
                {"detail": "Invalid or expired code."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        otp.mark_used()
        if not user.is_verified:
            user.is_verified = True
            user.save(update_fields=["is_verified"])

        tokens = RefreshToken.for_user(user)
        return Response(
            {
                "user": UserReadSerializer(user).data,
                "access": str(tokens.access_token),
                "refresh": str(tokens),
            },
            status=status.HTTP_200_OK,
        )
```

**Why generic error on wrong email:** Prevents user enumeration from public endpoints; for internal admin tools you can log the real reason.

---

### Step 3f — Forgot password flow

** serializers — optional hardening:** You can validate that the user exists in `ForgotPasswordSerializer`, but that leaks which emails are registered. For learning, two patterns:

- **Same response always** (recommended): always return `200` with “If an account exists, we sent a code.”
- **Strict:** 404 if email unknown (simpler UX, weaker privacy).

Below: **privacy-preserving** pattern.

```python
    @action(detail=False, methods=["post"], permission_classes=[AllowAny])
    def forgot_password(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        email = serializer.validated_data["email"]

        user = User.objects.filter(email=email).first()
        if user is not None and user.is_active:
            plain = issue_otp(user, EmailOTPPurpose.PASSWORD_RESET)
            send_otp_email(user.email, EmailOTPPurpose.PASSWORD_RESET, plain)

        return Response(
            {"detail": "If an account exists for this email, a reset code has been sent."},
            status=status.HTTP_200_OK,
        )

    @action(detail=False, methods=["post"], permission_classes=[AllowAny])
    def reset_password(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        email = serializer.validated_data["email"]
        code = serializer.validated_data["code"]
        new_password = serializer.validated_data["new_password"]

        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return Response(
                {"detail": "Invalid code or email."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        otp = verify_otp(user, EmailOTPPurpose.PASSWORD_RESET, code)
        if otp is None:
            return Response(
                {"detail": "Invalid or expired code."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        otp.mark_used()
        user.set_password(new_password)
        user.save(update_fields=["password"])

        tokens = RefreshToken.for_user(user)
        return Response(
            {
                "detail": "Password has been reset.",
                "access": str(tokens.access_token),
                "refresh": str(tokens),
            },
            status=status.HTTP_200_OK,
        )
```

---

### Step 3g — Block login until verified (recommended)

**File:** `accounts/api/serializers.py` — inside `LoginSerializer.validate`, after password check:

```python
        if not user.is_verified:
            raise serializers.ValidationError(
                "Please verify your email before logging in."
            )
```

**Why:** Forces clients through `verify_email` (or a future “resend code” path).

---

### Step 3h — Admin visibility (optional but useful for learning)

**File:** `accounts/admin.py`

```python
from django.contrib import admin
from .models import User, EmailOTP


@admin.register(EmailOTP)
class EmailOTPAdmin(admin.ModelAdmin):
    list_display = ("id", "user", "purpose", "created_at", "expires_at", "used_at")
    list_filter = ("purpose",)
    readonly_fields = ("code_hash", "created_at", "expires_at", "used_at")


admin.site.register(User)
```

(Remove duplicate `admin.site.register(User)` if you already register `User` — keep a single registration.)

---

## 4. Project structure integration

Suggested layout after implementation:

```text
accounts/
  __init__.py
  apps.py
  admin.py
  constants.py
  emails.py                 # NEW — send_otp_email
  managers.py
  models.py                 # User + EmailOTP
  urls.py
  services/
    __init__.py             # NEW
    otp.py                  # NEW — issue_otp, verify_otp
  api/
    serializers.py          # signup tweak + verify/forgot/reset serializers
    views.py                # signup, verify_email, forgot_password, reset_password
    permissions.py          # (optional future)
```

**Router URLs** (no change needed in `accounts/urls.py` if you use `@action` on `UserViewSet`):

| Method | Path |
|--------|------|
| POST | `/api/accounts/users/signup/` |
| POST | `/api/accounts/users/verify_email/` |
| POST | `/api/accounts/users/login/` |
| POST | `/api/accounts/users/forgot_password/` |
| POST | `/api/accounts/users/reset_password/` |
| GET/PATCH | `/api/accounts/users/me/` |

---

## 5. Edge cases and best practices

### 5.1 OTP expiry

- **`issue_otp`** sets `expires_at`; **`verify_otp`** only considers rows with `expires_at > now`.
- **`is_expired`** on the model is useful in admin/debug.

### 5.2 Invalid or used OTP

- **`verify_code`** returns `False` if `used_at` is set or expired.
- After success, always **`mark_used()`** before changing passwords or flipping `is_verified`, to prevent replay.

### 5.3 Replacing old OTPs

- **`issue_otp`** sets `expires_at=now` on previous active OTPs for the same user + purpose so only the latest is valid (your `get_latest_valid_otp` already orders by `-created_at`).

### 5.4 Rate limiting and resend

**Problem:** Attackers can spam signup/forgot and exhaust email quotas or harass users.

**Practices:**

1. **Resend endpoint** (you implement): `POST /api/accounts/users/resend_signup_otp/` with `{ "email" }` — only call `issue_otp` if user exists and `is_verified` is `False`; apply cooldown (e.g. 60s) using:
   - **Cache** (`django.core.cache` with Redis URL you already have in deps), or
   - **DB:** last `EmailOTP.created_at` for that user + purpose.
2. **Global throttling** (DRF): add to `REST_FRAMEWORK` in `noola/settings.py`:

```python
REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "30/minute",
    },
}
```

3. Per-view throttle classes for `signup` / `forgot_password` with stricter rates.

### 5.5 Brute-force on code

- 6-digit codes are guessable at scale: combine **short expiry**, **throttling**, and optionally **max attempts** (add `attempts` field on `EmailOTP` or count failures in cache per email).

---

## 6. Testing guide (no frontend)

### 6.1 Enable JWT on protected routes (required for `/users/me/`)

Your `login` and `verify_email` return JWTs, but **`REST_FRAMEWORK` does not yet declare JWT authentication**. Add to `noola/settings.py`:

```python
REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ),
}
```

Restart the server after changing settings.

### 6.2 Swagger UI (`/api/docs/`)

1. Open `http://127.0.0.1:8000/api/docs/`.
2. Execute **signup** → read server console for OTP (`console` email backend prints the email body).
3. Execute **verify_email** with the same email and code → copy `access` from response.
4. Click **Authorize** (lock icon), use `Bearer <access>` (or the UI scheme your Spectacular version expects — often prefix `Bearer ` in the token field).
5. Call **me** with GET/PATCH.

### 6.3 curl examples

**Signup:**

```bash
curl -X POST http://127.0.0.1:8000/api/accounts/users/signup/ \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"learner@example.com\",\"password\":\"securepass123\",\"first_name\":\"Test\",\"last_name\":\"User\"}"
```

**Verify (read code from runserver console):**

```bash
curl -X POST http://127.0.0.1:8000/api/accounts/users/verify_email/ \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"learner@example.com\",\"code\":\"123456\"}"
```

**Login (after verified):**

```bash
curl -X POST http://127.0.0.1:8000/api/accounts/users/login/ \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"learner@example.com\",\"password\":\"securepass123\"}"
```

**Me:**

```bash
curl http://127.0.0.1:8000/api/accounts/users/me/ \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 6.4 Inspecting OTP values during development

| Method | How |
|--------|-----|
| Console backend | OTP appears in the terminal running `runserver`. |
| Django admin | See `EmailOTP` rows (`used_at`, `expires_at`); **code is hashed** — you cannot read the plain code from DB. |
| Temporary DEBUG log | Only if you must: log `plain` inside `signup`/`forgot_password` when `DEBUG` is True — **remove before production**. |

### 6.5 Automated tests (outline)

**File:** `accounts/tests.py` (or `accounts/tests/test_otp.py` with pytest-style classes if you adopt pytest later).

- Create user, `issue_otp`, assert `verify_otp` succeeds, `mark_used`, assert second verify fails.
- Expired OTP: create row with `expires_at` in the past, assert verify fails.
- `forgot_password` does not error for unknown email (if you use privacy-preserving response).

Use `django.test.TestCase` and `APIClient` from `rest_framework.test`.

---

## 7. Final summary

### 7.1 End-to-end flows

**Signup + verification**

1. `POST .../signup/` → creates `User` (`is_verified=False`), new `EmailOTP` purpose `signup`, email sent.
2. `POST .../verify_email/` → validates code, marks OTP used, sets `is_verified=True`, returns JWTs.
3. `POST .../login/` → works only if verified (if you added the check).

**Password reset**

1. `POST .../forgot_password/` → if user exists and active, new `password_reset` OTP, email sent; response always generic.
2. `POST .../reset_password/` → validates OTP, marks used, `set_password`, returns JWTs.

### 7.2 Key design decisions

| Decision | Rationale |
|----------|-----------|
| DB-backed `EmailOTP` | Auditable, no extra services for the learning path. |
| Hashed codes | DB leaks do not expose active OTPs. |
| `User` created before verification | Fits existing `SignupSerializer` and `is_verified` flag. |
| JWT only after verify | Aligns access control with proven email ownership. |
| Generic forgot-password message | Reduces email enumeration. |

### 7.3 Production-oriented improvements

1. **Celery task** for `send_otp_email` + retries; view returns immediately after writing OTP.
2. **Redis** for rate limits and resend cooldowns (you already have `redis` in requirements).
3. **HTML email templates** (`templates/emails/otp.html`) with `django.core.mail.EmailMultiAlternatives`.
4. **django-anymail** (or SES API) if you outgrow SMTP.
5. **Separate refresh token rotation / blacklist** (SimpleJWT settings) if sessions must be invalidated on password reset.
6. **Monitoring**: metrics for OTP issue/verify counts and failure rates.

---

You now have a single document that mirrors **your** `accounts` layout, **`User.is_verified`**, router URLs under **`api/accounts/users/`**, and your existing **JWT + Spectacular** stack. Implement in order §3a → §3h, then validate with §6.
