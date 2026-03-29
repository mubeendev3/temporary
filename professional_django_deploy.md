# 🚀 Django Deployment Guide (PythonAnywhere) — Todo App

## 📌 Project Overview
- **Project**: Todo App Using Django  
- **GitHub Repo**: https://github.com/mubeendev3/Todo-App-Using-Django  
- **Live URL**: https://mubeendev3.pythonanywhere.com/  
- **Hosting**: PythonAnywhere  

---

# 🧠 Architecture Overview

```
User → Browser → PythonAnywhere → WSGI → Django App → Database (SQLite)
```

---

# ⚙️ Step 1: Clone Repository

```bash
cd ~
git clone https://github.com/mubeendev3/Todo-App-Using-Django.git
cd Todo-App-Using-Django
```

---

# 🐍 Step 2: Virtual Environment Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### ❗ Fix Encoding Issue (IMPORTANT)
If you see:
```
a\x00s\x00g...
```

👉 Fix by:
- Re-saving file as UTF-8  
- OR recreate manually  

---

# 🗄️ Step 3: Database Setup

```bash
python manage.py migrate
```

---

# 📦 Step 4: Static Files Collection

```bash
python manage.py collectstatic
```

---

# ⚙️ Step 5: Django Settings Configuration

## 🔒 Production Settings

```python
DEBUG = False
ALLOWED_HOSTS = ['mubeendev3.pythonanywhere.com']
CSRF_TRUSTED_ORIGINS = ['https://mubeendev3.pythonanywhere.com']
```

---

## 📁 Static & Media

```python
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

---

# 🌐 Step 6: PythonAnywhere Web Configuration

## 🔹 Virtual Environment
```
/home/mubeendev3/Todo-App-Using-Django/venv
```

## 🔹 Source Code
```
/home/mubeendev3/Todo-App-Using-Django
```

---

## 🔹 Static Files Mapping

| URL | Directory |
|-----|----------|
| /static/ | /home/mubeendev3/Todo-App-Using-Django/staticfiles |
| /media/  | /home/mubeendev3/Todo-App-Using-Django/media |

---

# 🔥 Step 7: WSGI Configuration (CRITICAL FIX)

```python
import sys
import os

project_home = '/home/mubeendev3/Todo-App-Using-Django'

if project_home not in sys.path:
    sys.path.append(project_home)

os.environ['DJANGO_SETTINGS_MODULE'] = 'todoproject.settings'

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

---

# 🔄 Step 8: Reload Application

👉 PythonAnywhere → Web Tab → Reload

---

# ❌ Common Errors & Fixes

## 1. ModuleNotFoundError: todo_django

**Cause:**
Wrong settings module

**Fix:**
```python
todoproject.settings
```

---

## 2. Static Files Not Loading

**Fix:**
```bash
python manage.py collectstatic
```

---

## 3. requirements.txt Error

**Cause:**
Wrong encoding (UTF-16)

**Fix:**
Convert to UTF-8

---

## 4. 500 Internal Server Error

👉 Check logs:
```
/var/log/...error.log
```

---

# 🧪 Verification Checklist

- [ ] Site loads without error  
- [ ] CSS applied correctly  
- [ ] Admin panel accessible  
- [ ] No console errors  
- [ ] Static files working  

---

# 🎯 Final Output

👉 Live App:
https://mubeendev3.pythonanywhere.com/

---

# 💡 Best Practices

- Use `.env` for secrets  
- Never commit SECRET_KEY  
- Use PostgreSQL in production  
- Add logging for debugging  

---

# 🏁 Conclusion

Deployment completed successfully 🎉  
Your Django app is now live on PythonAnywhere.

---

# 👨‍💻 Author

**Mubeen Mehmood**  
Software Engineer | Django Developer  

