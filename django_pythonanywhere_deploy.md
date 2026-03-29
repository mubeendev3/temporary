# Django Deployment Guide (PythonAnywhere) — Todo App

## Project Info
- Repo: https://github.com/mubeendev3/Todo-App-Using-Django
- Live URL: https://mubeendev3.pythonanywhere.com/
- Username: mubeendev3

---

## Step 1: Clone Repo
```bash
cd ~
git clone https://github.com/mubeendev3/Todo-App-Using-Django.git
cd Todo-App-Using-Django
```

---

## Step 2: Create Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

## Step 3: Fix requirements.txt (Encoding Issue)
If error like `a\x00s\x00g...` appears:
- Open file
- Save as UTF-8
- Or rewrite manually

---

## Step 4: Run Migrations & Static
```bash
python manage.py migrate
python manage.py collectstatic
```

---

## Step 5: settings.py Changes

### DEBUG
```python
DEBUG = False
```

### ALLOWED_HOSTS
```python
ALLOWED_HOSTS = ['mubeendev3.pythonanywhere.com']
```

### CSRF
```python
CSRF_TRUSTED_ORIGINS = ['https://mubeendev3.pythonanywhere.com']
```

### STATIC
```python
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

### MEDIA (optional)
```python
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

---

## Step 6: Web Tab Config

### Virtualenv
/home/mubeendev3/Todo-App-Using-Django/venv

### Source Code
/home/mubeendev3/Todo-App-Using-Django

### Static Mapping
URL: /static/
Directory: /home/mubeendev3/Todo-App-Using-Django/staticfiles

---

## Step 7: WSGI Fix (IMPORTANT)

```python
import sys
import os

path = '/home/mubeendev3/Todo-App-Using-Django'
if path not in sys.path:
    sys.path.append(path)

os.environ['DJANGO_SETTINGS_MODULE'] = 'todoproject.settings'

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

---

## Step 8: Reload Web App
Go to Web tab → Reload

---

## Common Errors & Fixes

### ModuleNotFoundError: todo_django
Fix:
```python
todoproject.settings
```

### Static not loading
Run:
```bash
python manage.py collectstatic
```

### requirements.txt error
Fix encoding to UTF-8

---

## Final Result
Your site should work at:
https://mubeendev3.pythonanywhere.com/

---

## Pro Tips
- Always match project name in WSGI
- Use collectstatic for production
- Use .env for secrets (advanced)

---

🔥 Deployment Completed Successfully!
