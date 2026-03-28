# 🚀 Django To-Do App — Zero se Hero Guide

> **Beginner-Friendly | Modern Practices | Step-by-Step**  
> Yeh guide follow karke tum ek complete To-Do web app banao ge — CRUD operations, Django Forms, Class-Based Views, aur deployment tips ke sath.

---

## 📋 Table of Contents

1. [Project Overview](#1-project-overview)
2. [Environment Setup](#2-environment-setup)
3. [Django Project Structure](#3-django-project-structure)
4. [Database Models](#4-database-models)
5. [Django Admin Setup](#5-django-admin-setup)
6. [Forms — User Input Handle Karna](#6-forms)
7. [Views — Business Logic](#7-views)
8. [URLs — Routing](#8-urls)
9. [Templates — HTML Pages](#9-templates)
10. [Static Files — CSS & JS](#10-static-files)
11. [CRUD Operations — Full Walkthrough](#11-crud-operations)
12. [Common Beginner Mistakes](#12-common-beginner-mistakes)
13. [Deployment Tips](#13-deployment-tips)
14. [Final Checklist](#14-final-checklist)

---

## 1. Project Overview

### Hum kya banayenge?

Ek **To-Do Web App** jisme:
- ✅ Tasks add kar sako (Create)
- 📋 Saari tasks dekh sako (Read)
- ✏️ Tasks edit kar sako (Update)
- 🗑️ Tasks delete kar sako (Delete)
- ✔️ Task complete/incomplete mark kar sako

### Tech Stack

| Technology | Kaam |
|---|---|
| **Python 3.10+** | Programming language |
| **Django 4.x** | Web framework |
| **SQLite** | Database (default, beginner ke liye perfect) |
| **Bootstrap 5** | CSS styling (CDN se) |
| **Django Templates** | HTML rendering |

---

## 2. Environment Setup

### Step 2.1 — Python Check Karo

```bash
python --version
# Ya:
python3 --version
```

> ⚠️ **Common Mistake:** Python 2 installed hoga to Django kaam nahi karega. Python 3.8+ chahiye.

---

### Step 2.2 — Virtual Environment Banao

**Virtual environment kyun?**  
Har project ke apne alag dependencies hote hain. Venv ek isolated box hai jisme sirf us project ka setup hota hai — system Python ko affect nahi karta.

```bash
# Project folder banao
mkdir django-todo
cd django-todo

# Virtual environment create karo
python -m venv venv

# Activate karo:
# Windows:
venv\Scripts\activate

# Mac/Linux:
source venv/bin/activate
```

Activate hone ke baad terminal me `(venv)` dikhega:

```
(venv) C:\Users\YourName\django-todo>
```

> ⚠️ **Common Mistake:** Har baar naya terminal kholo to `venv` activate karna bhool jaate hain. Agar `django-admin` command nahi milti — check karo venv activate hai ya nahi.

---

### Step 2.3 — Django Install Karo

```bash
pip install django

# Verify karo:
django-admin --version
# Output: 4.x.x
```

Dependencies save karo (team ya deployment ke liye):

```bash
pip freeze > requirements.txt
```

---

### Step 2.4 — Django Project Start Karo

```bash
django-admin startproject todoproject .
```

> **Note:** End me `.` (dot) lagana mat bhulo! Isse project files seedha current folder me banti hain — extra nested folder nahi banta.

Ab `todo` naam ki app banao:

```bash
python manage.py startapp todo
```

---

## 3. Django Project Structure

Abhi tumhara folder structure aisa dikhega:

```
django-todo/
│
├── venv/                    ← Virtual environment (git me mat dalna)
│
├── todoproject/             ← Main project settings folder
│   ├── __init__.py
│   ├── settings.py          ← Saari settings (database, apps, etc.)
│   ├── urls.py              ← Root URL configuration
│   ├── wsgi.py              ← Deployment ke liye
│   └── asgi.py
│
├── todo/                    ← Hamari To-Do app
│   ├── migrations/          ← Database migration files
│   ├── templates/           ← HTML files (hum khud banayenge)
│   ├── static/              ← CSS/JS files (hum khud banayenge)
│   ├── __init__.py
│   ├── admin.py             ← Admin panel configuration
│   ├── apps.py
│   ├── forms.py             ← Django Forms (hum banayenge)
│   ├── models.py            ← Database models
│   ├── urls.py              ← App-level URLs (hum banayenge)
│   └── views.py             ← Views (business logic)
│
├── manage.py                ← Django ka main command tool
└── requirements.txt
```

### App ko Project me Register Karo

`todoproject/settings.py` kholo aur `INSTALLED_APPS` me apni app add karo:

```python
# todoproject/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    'todo',  # ← Yeh add karo
]
```

**Kyun?** Django ko pata hona chahiye ki kaunsi apps use karni hain — tabhi woh models, templates, aur migrations ko dhundh payega.

---

## 4. Database Models

**Model kya hai?**  
Model Python class hai jo database table represent karta hai. Ek field = ek column.

`todo/models.py` kholo aur yeh likho:

```python
# todo/models.py

from django.db import models


class Category(models.Model):
    """Task categories jaise: Work, Personal, Shopping"""
    name = models.CharField(max_length=100)
    
    class Meta:
        verbose_name_plural = "Categories"
    
    def __str__(self):
        return self.name


class Task(models.Model):
    """Main To-Do task model"""
    
    PRIORITY_CHOICES = [
        ('low', 'Low'),
        ('medium', 'Medium'),
        ('high', 'High'),
    ]
    
    title = models.CharField(max_length=200)               # Task ka naam
    description = models.TextField(blank=True, null=True)  # Optional detail
    completed = models.BooleanField(default=False)         # Done hai ya nahi
    priority = models.CharField(
        max_length=10,
        choices=PRIORITY_CHOICES,
        default='medium'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='tasks'
    )
    created_at = models.DateTimeField(auto_now_add=True)   # Automatic timestamp
    updated_at = models.DateTimeField(auto_now=True)       # Update pe automatic
    due_date = models.DateField(null=True, blank=True)     # Optional deadline
    
    class Meta:
        ordering = ['-created_at']  # Nayi tasks pehle dikhayega
    
    def __str__(self):
        return self.title
```

**Fields Explanation:**

| Field | Type | Matlab |
|---|---|---|
| `title` | CharField | Chhoti text (max length required) |
| `description` | TextField | Badi text, koi limit nahi |
| `completed` | BooleanField | True/False |
| `priority` | CharField + choices | Fixed options me se ek |
| `category` | ForeignKey | Doosre model se relation (Many-to-One) |
| `created_at` | DateTimeField | Auto-fill on create |
| `updated_at` | DateTimeField | Auto-fill on every save |

### Migrations Run Karo

```bash
# Migration files banao (changes detect karta hai)
python manage.py makemigrations

# Database me changes apply karo
python manage.py migrate
```

> ⚠️ **Common Mistake:** Model change karne ke baad `makemigrations` aur `migrate` dono run karna zaruri hai. Sirf ek se kaam nahi chalega.

---

## 5. Django Admin Setup

Django ka built-in admin panel bahut powerful hai — bina extra code ke database manage kar sakte ho.

```python
# todo/admin.py

from django.contrib import admin
from .models import Task, Category


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name']
    search_fields = ['name']


@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = ['title', 'priority', 'completed', 'category', 'created_at']
    list_filter = ['completed', 'priority', 'category']
    search_fields = ['title', 'description']
    list_editable = ['completed']  # Seedha list me edit kar sako
    date_hierarchy = 'created_at'
```

### Superuser Banao

```bash
python manage.py createsuperuser
# Username, email, password daalo
```

Server chalao aur admin panel access karo:

```bash
python manage.py runserver
```

Browser me jao: `http://127.0.0.1:8000/admin/`

---

## 6. Forms

**Django Forms kyun?**  
- User input ko validate karta hai automatically
- HTML form generate karta hai
- Security (CSRF protection) built-in hai

`todo/forms.py` file banao:

```python
# todo/forms.py

from django import forms
from .models import Task, Category


class TaskForm(forms.ModelForm):
    """Task create/edit karne ke liye form"""
    
    class Meta:
        model = Task
        fields = ['title', 'description', 'priority', 'category', 'due_date']
        
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Task ka naam likho...'
            }),
            'description': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 3,
                'placeholder': 'Optional description...'
            }),
            'priority': forms.Select(attrs={
                'class': 'form-select'
            }),
            'category': forms.Select(attrs={
                'class': 'form-select'
            }),
            'due_date': forms.DateInput(attrs={
                'class': 'form-control',
                'type': 'date'
            }),
        }
    
    def clean_title(self):
        """Custom validation — title empty nahi hona chahiye"""
        title = self.cleaned_data.get('title')
        if len(title.strip()) < 3:
            raise forms.ValidationError("Title kam se kam 3 characters ka hona chahiye.")
        return title.strip()


class CategoryForm(forms.ModelForm):
    class Meta:
        model = Category
        fields = ['name']
        widgets = {
            'name': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Category naam...'
            })
        }
```

---

## 7. Views

**Class-Based Views (CBV) kyun?**  
- Kam code me zyada kaam
- Reusable aur organized
- Django best practice hai

`todo/views.py` kholo:

```python
# todo/views.py

from django.shortcuts import render, get_object_or_404, redirect
from django.views import View
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.urls import reverse_lazy
from django.contrib import messages
from .models import Task, Category
from .forms import TaskForm, CategoryForm


# ─── TASK LIST VIEW ────────────────────────────────────────────────────────────

class TaskListView(ListView):
    """Saari tasks dikhao — home page"""
    model = Task
    template_name = 'todo/task_list.html'
    context_object_name = 'tasks'
    
    def get_queryset(self):
        """Filter aur search support"""
        queryset = Task.objects.all()
        
        # Filter by completion status
        status = self.request.GET.get('status')
        if status == 'completed':
            queryset = queryset.filter(completed=True)
        elif status == 'pending':
            queryset = queryset.filter(completed=False)
        
        # Filter by priority
        priority = self.request.GET.get('priority')
        if priority in ['low', 'medium', 'high']:
            queryset = queryset.filter(priority=priority)
        
        # Search
        search = self.request.GET.get('search')
        if search:
            queryset = queryset.filter(title__icontains=search)
        
        return queryset
    
    def get_context_data(self, **kwargs):
        """Template ko extra data do"""
        context = super().get_context_data(**kwargs)
        context['total_tasks'] = Task.objects.count()
        context['completed_tasks'] = Task.objects.filter(completed=True).count()
        context['pending_tasks'] = Task.objects.filter(completed=False).count()
        return context


# ─── TASK CREATE VIEW ──────────────────────────────────────────────────────────

class TaskCreateView(CreateView):
    """Nayi task banao"""
    model = Task
    form_class = TaskForm
    template_name = 'todo/task_form.html'
    success_url = reverse_lazy('todo:task_list')
    
    def form_valid(self, form):
        messages.success(self.request, '✅ Task successfully add ho gayi!')
        return super().form_valid(form)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['page_title'] = 'Nayi Task Add Karo'
        context['button_text'] = 'Task Add Karo'
        return context


# ─── TASK UPDATE VIEW ──────────────────────────────────────────────────────────

class TaskUpdateView(UpdateView):
    """Existing task edit karo"""
    model = Task
    form_class = TaskForm
    template_name = 'todo/task_form.html'
    success_url = reverse_lazy('todo:task_list')
    
    def form_valid(self, form):
        messages.success(self.request, '✏️ Task update ho gayi!')
        return super().form_valid(form)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['page_title'] = 'Task Edit Karo'
        context['button_text'] = 'Update Karo'
        return context


# ─── TASK DELETE VIEW ──────────────────────────────────────────────────────────

class TaskDeleteView(DeleteView):
    """Task delete karo"""
    model = Task
    template_name = 'todo/task_confirm_delete.html'
    success_url = reverse_lazy('todo:task_list')
    
    def form_valid(self, form):
        messages.success(self.request, '🗑️ Task delete ho gayi!')
        return super().form_valid(form)


# ─── TOGGLE COMPLETE ───────────────────────────────────────────────────────────

class TaskToggleView(View):
    """Task complete/incomplete toggle karo"""
    
    def post(self, request, pk):
        task = get_object_or_404(Task, pk=pk)
        task.completed = not task.completed
        task.save()
        status = "complete" if task.completed else "incomplete"
        messages.info(request, f'Task "{task.title}" {status} mark ho gayi!')
        return redirect('todo:task_list')
```

---

## 8. URLs

### App-level URLs

`todo/urls.py` file banao (yeh file exist nahi karti, khud banao):

```python
# todo/urls.py

from django.urls import path
from . import views

app_name = 'todo'  # Namespace — URLs ko unique banata hai

urlpatterns = [
    path('', views.TaskListView.as_view(), name='task_list'),
    path('task/new/', views.TaskCreateView.as_view(), name='task_create'),
    path('task/<int:pk>/edit/', views.TaskUpdateView.as_view(), name='task_update'),
    path('task/<int:pk>/delete/', views.TaskDeleteView.as_view(), name='task_delete'),
    path('task/<int:pk>/toggle/', views.TaskToggleView.as_view(), name='task_toggle'),
]
```

### Project-level URLs

`todoproject/urls.py` update karo:

```python
# todoproject/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('todo.urls', namespace='todo')),  # Todo app ki URLs include karo
]
```

> ⚠️ **Common Mistake:** `app_name` aur `namespace` dono milne chahiye — warna `NoReverseMatch` error aayega.

---

## 9. Templates

### Folder Structure Banao

```bash
mkdir -p todo/templates/todo
```

Final template structure:

```
todo/
└── templates/
    └── todo/
        ├── base.html              ← Parent template
        ├── task_list.html         ← Home page
        ├── task_form.html         ← Create/Edit form
        └── task_confirm_delete.html ← Delete confirmation
```

---

### base.html — Parent Template

```html
<!-- todo/templates/todo/base.html -->
<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}My To-Do App{% endblock %}</title>
    
    <!-- Bootstrap 5 CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    
    {% load static %}
    <link rel="stylesheet" href="{% static 'todo/css/style.css' %}">
</head>
<body class="bg-light">

    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand fw-bold" href="{% url 'todo:task_list' %}">📝 My To-Do App</a>
            <div class="navbar-nav ms-auto">
                <a class="btn btn-light btn-sm" href="{% url 'todo:task_create' %}">+ Nayi Task</a>
            </div>
        </div>
    </nav>

    <!-- Messages / Notifications -->
    <div class="container mt-3">
        {% if messages %}
            {% for message in messages %}
                <div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
                    {{ message }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            {% endfor %}
        {% endif %}
    </div>

    <!-- Main Content -->
    <main class="container my-4">
        {% block content %}{% endblock %}
    </main>

    <!-- Footer -->
    <footer class="text-center py-3 text-muted">
        <small>Django To-Do App &copy; 2024</small>
    </footer>

    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

---

### task_list.html — Home Page

```html
<!-- todo/templates/todo/task_list.html -->
{% extends 'todo/base.html' %}

{% block title %}My Tasks{% endblock %}

{% block content %}

<!-- Stats Cards -->
<div class="row mb-4">
    <div class="col-md-4">
        <div class="card text-white bg-primary mb-3">
            <div class="card-body text-center">
                <h2 class="card-title">{{ total_tasks }}</h2>
                <p class="card-text">Total Tasks</p>
            </div>
        </div>
    </div>
    <div class="col-md-4">
        <div class="card text-white bg-success mb-3">
            <div class="card-body text-center">
                <h2 class="card-title">{{ completed_tasks }}</h2>
                <p class="card-text">Completed</p>
            </div>
        </div>
    </div>
    <div class="col-md-4">
        <div class="card text-white bg-warning mb-3">
            <div class="card-body text-center">
                <h2 class="card-title">{{ pending_tasks }}</h2>
                <p class="card-text">Pending</p>
            </div>
        </div>
    </div>
</div>

<!-- Search & Filter Form -->
<div class="card mb-4 shadow-sm">
    <div class="card-body">
        <form method="GET" class="row g-3">
            <div class="col-md-4">
                <input type="text" name="search" class="form-control"
                       placeholder="Task search karo..." value="{{ request.GET.search }}">
            </div>
            <div class="col-md-3">
                <select name="status" class="form-select">
                    <option value="">Sab Tasks</option>
                    <option value="pending" {% if request.GET.status == 'pending' %}selected{% endif %}>Pending</option>
                    <option value="completed" {% if request.GET.status == 'completed' %}selected{% endif %}>Completed</option>
                </select>
            </div>
            <div class="col-md-3">
                <select name="priority" class="form-select">
                    <option value="">Sab Priority</option>
                    <option value="high" {% if request.GET.priority == 'high' %}selected{% endif %}>High</option>
                    <option value="medium" {% if request.GET.priority == 'medium' %}selected{% endif %}>Medium</option>
                    <option value="low" {% if request.GET.priority == 'low' %}selected{% endif %}>Low</option>
                </select>
            </div>
            <div class="col-md-2">
                <button type="submit" class="btn btn-primary w-100">Filter</button>
            </div>
        </form>
    </div>
</div>

<!-- Task List -->
{% if tasks %}
    <div class="list-group shadow-sm">
        {% for task in tasks %}
        <div class="list-group-item list-group-item-action {% if task.completed %}bg-light{% endif %}">
            <div class="d-flex align-items-center justify-content-between">
                
                <!-- Left Side: Checkbox + Title -->
                <div class="d-flex align-items-center gap-3">
                    <form method="POST" action="{% url 'todo:task_toggle' task.pk %}">
                        {% csrf_token %}
                        <button type="submit" class="btn btn-sm {% if task.completed %}btn-success{% else %}btn-outline-secondary{% endif %} rounded-circle">
                            {% if task.completed %}✓{% else %}○{% endif %}
                        </button>
                    </form>
                    
                    <div>
                        <h6 class="mb-0 {% if task.completed %}text-decoration-line-through text-muted{% endif %}">
                            {{ task.title }}
                        </h6>
                        <small class="text-muted">
                            {% if task.category %}{{ task.category.name }} &bull;{% endif %}
                            {% if task.due_date %}Due: {{ task.due_date }}{% endif %}
                        </small>
                    </div>
                </div>
                
                <!-- Right Side: Priority Badge + Actions -->
                <div class="d-flex align-items-center gap-2">
                    <span class="badge 
                        {% if task.priority == 'high' %}bg-danger
                        {% elif task.priority == 'medium' %}bg-warning text-dark
                        {% else %}bg-secondary{% endif %}">
                        {{ task.get_priority_display }}
                    </span>
                    
                    <a href="{% url 'todo:task_update' task.pk %}" class="btn btn-sm btn-outline-primary">✏️</a>
                    <a href="{% url 'todo:task_delete' task.pk %}" class="btn btn-sm btn-outline-danger">🗑️</a>
                </div>
                
            </div>
        </div>
        {% endfor %}
    </div>

{% else %}
    <!-- Empty State -->
    <div class="text-center py-5">
        <h2>📋</h2>
        <h4 class="text-muted">Koi task nahi mili!</h4>
        <p class="text-muted">Nayi task add karo ya filter clear karo.</p>
        <a href="{% url 'todo:task_create' %}" class="btn btn-primary">+ Pehli Task Add Karo</a>
    </div>
{% endif %}

{% endblock %}
```

---

### task_form.html — Create/Edit Form

```html
<!-- todo/templates/todo/task_form.html -->
{% extends 'todo/base.html' %}

{% block title %}{{ page_title }}{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-7">
        <div class="card shadow-sm">
            <div class="card-header bg-primary text-white">
                <h4 class="mb-0">{{ page_title }}</h4>
            </div>
            <div class="card-body">
                
                <form method="POST" novalidate>
                    {% csrf_token %}
                    
                    {% for field in form %}
                    <div class="mb-3">
                        <label for="{{ field.id_for_label }}" class="form-label fw-semibold">
                            {{ field.label }}
                            {% if field.field.required %}<span class="text-danger">*</span>{% endif %}
                        </label>
                        
                        {{ field }}
                        
                        {% if field.errors %}
                            {% for error in field.errors %}
                                <div class="text-danger small mt-1">⚠️ {{ error }}</div>
                            {% endfor %}
                        {% endif %}
                        
                        {% if field.help_text %}
                            <small class="text-muted">{{ field.help_text }}</small>
                        {% endif %}
                    </div>
                    {% endfor %}
                    
                    <div class="d-flex gap-2">
                        <button type="submit" class="btn btn-primary">{{ button_text }}</button>
                        <a href="{% url 'todo:task_list' %}" class="btn btn-outline-secondary">Cancel</a>
                    </div>
                    
                </form>
                
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

### task_confirm_delete.html — Delete Confirmation

```html
<!-- todo/templates/todo/task_confirm_delete.html -->
{% extends 'todo/base.html' %}

{% block title %}Task Delete Karo{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <div class="card shadow-sm border-danger">
            <div class="card-header bg-danger text-white">
                <h4 class="mb-0">🗑️ Task Delete Karo</h4>
            </div>
            <div class="card-body text-center">
                <p class="fs-5">Kya aap sach me yeh task delete karna chahte ho?</p>
                <p class="fw-bold fs-4">"{{ object.title }}"</p>
                <p class="text-muted">Yeh action undo nahi hoga!</p>
                
                <form method="POST">
                    {% csrf_token %}
                    <button type="submit" class="btn btn-danger me-2">Haan, Delete Karo</button>
                    <a href="{% url 'todo:task_list' %}" class="btn btn-outline-secondary">Cancel</a>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## 10. Static Files

### Folder Structure Banao

```bash
mkdir -p todo/static/todo/css
```

### Custom CSS Banao

```css
/* todo/static/todo/css/style.css */

/* ─── General ─────────────────────────────── */
body {
    font-family: 'Segoe UI', sans-serif;
}

/* ─── Navbar ──────────────────────────────── */
.navbar-brand {
    font-size: 1.4rem;
    letter-spacing: 0.5px;
}

/* ─── Cards ───────────────────────────────── */
.card {
    border-radius: 12px;
    border: none;
}

/* ─── Task List Item ──────────────────────── */
.list-group-item {
    border-radius: 8px !important;
    margin-bottom: 8px;
    border: 1px solid #e0e0e0 !important;
    transition: box-shadow 0.2s ease;
}

.list-group-item:hover {
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

/* ─── Completed Task ──────────────────────── */
.text-decoration-line-through {
    opacity: 0.6;
}

/* ─── Priority Badges ─────────────────────── */
.badge {
    font-size: 0.75rem;
    padding: 6px 10px;
    border-radius: 20px;
}

/* ─── Buttons ─────────────────────────────── */
.btn {
    border-radius: 8px;
}
```

### Settings me Static Files Configure Karo

`todoproject/settings.py` me check karo:

```python
# todoproject/settings.py

STATIC_URL = '/static/'
STATICFILES_DIRS = [BASE_DIR / 'static']  # Optional global static folder ke liye
```

> ⚠️ **Common Mistake:** Template me `{% load static %}` aur `{% static '...' %}` banana zaruri hai. Bhoolne par CSS load nahi hogi.

---

## 11. CRUD Operations — Full Walkthrough

### CREATE — Nayi Task Banana

1. User `/task/new/` URL pe jata hai
2. `TaskCreateView` empty form bhejta hai
3. User form fill karta hai aur submit karta hai
4. Django form validate karta hai (`TaskForm.clean_title()` bhi chalta hai)
5. Valid hone par `Task` object database me save hota hai
6. User redirect hota hai task list pe with success message

### READ — Tasks Dekhna

1. User `/` URL pe aata hai
2. `TaskListView.get_queryset()` chalta hai
3. GET parameters ke hisaab se filter hoti hain tasks
4. `get_context_data()` extra info (counts) bhejta hai
5. `task_list.html` render hota hai

### UPDATE — Task Edit Karna

1. User `/task/5/edit/` pe jata hai (`pk=5`)
2. `TaskUpdateView` existing task data se form pre-fill karta hai
3. User changes karta hai, submit karta hai
4. Validate hone par database me update hota hai
5. Redirect to list with message

### DELETE — Task Mitana

1. User `/task/5/delete/` pe jata hai
2. Confirmation page dikhta hai
3. User "Haan Delete Karo" press karta hai — POST request jata hai
4. Task database se delete hoti hai
5. Redirect to list

### TOGGLE — Complete/Incomplete

1. Task ke saamne circle button press hota hai
2. POST request jata hai `/task/5/toggle/`
3. `TaskToggleView` task dhoondh ke `completed` field flip karta hai
4. Same page pe redirect — task ka status change dikhta hai

---

## 12. Common Beginner Mistakes

### ❌ Mistake 1: `makemigrations` ke baad `migrate` bhool gaye

```bash
# Galat — sirf ek step:
python manage.py makemigrations

# Sahi — dono steps:
python manage.py makemigrations
python manage.py migrate
```

---

### ❌ Mistake 2: App `INSTALLED_APPS` me add nahi ki

Agar yeh bhool gaye toh templates, models kuch bhi kaam nahi karega.

```python
# settings.py me zarur add karo:
INSTALLED_APPS = [
    ...
    'todo',  # ← Yeh line zaroori hai
]
```

---

### ❌ Mistake 3: Template me `{% csrf_token %}` bhool gaye

Har POST form me yeh token zaroori hai — Django security feature hai.

```html
<form method="POST">
    {% csrf_token %}  {# ← Kabhi mat bhoolna! #}
    ...
</form>
```

---

### ❌ Mistake 4: URL namespace mismatch

```python
# urls.py me:
app_name = 'todo'

# Template me aur include() me same naam:
{% url 'todo:task_list' %}
path('', include('todo.urls', namespace='todo'))
```

---

### ❌ Mistake 5: Static files me `{% load static %}` bhool gaya

```html
{# Template ke top me yeh line zaroori hai: #}
{% load static %}

{# Tab yeh kaam karega: #}
<link href="{% static 'todo/css/style.css' %}">
```

---

### ❌ Mistake 6: Virtual environment activate nahi kiya

```bash
# Har naye terminal session me activate karo:
# Windows:
venv\Scripts\activate

# Mac/Linux:
source venv/bin/activate
```

---

### ❌ Mistake 7: Server chalate waqt `.` (dot) wali directory galat hai

```bash
# Sahi jagah se chalao — jahan manage.py hai:
python manage.py runserver

# Agar manage.py nahi mil raha — pehle sahi folder me jao:
cd django-todo
python manage.py runserver
```

---

## 13. Deployment Tips

### Pythonanywhere pe Free Deployment

1. [pythonanywhere.com](https://www.pythonanywhere.com) pe free account banao
2. Files upload karo ya Git use karo
3. Virtual environment banao aur requirements install karo
4. `settings.py` me changes karo:

```python
# settings.py (production ke liye)

DEBUG = False

ALLOWED_HOSTS = ['yourusername.pythonanywhere.com']

# Static files collect karne ke liye:
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

5. Static files collect karo:

```bash
python manage.py collectstatic
```

### Environment Variables Use Karo (Security)

Secret key aur database credentials code me mat rakho:

```bash
pip install python-decouple
```

```python
# settings.py
from decouple import config

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
```

`.env` file banao (git me add mat karo):

```
SECRET_KEY=your-very-secret-key-here
DEBUG=False
```

`.gitignore` me add karo:

```
venv/
*.pyc
__pycache__/
.env
db.sqlite3
staticfiles/
```

---

## 14. Final Checklist

Project complete karne ke liye in sab steps follow karo:

```
✅ Step 1:  Python install hai (3.8+)
✅ Step 2:  Virtual environment banaya aur activate kiya
✅ Step 3:  Django install kiya
✅ Step 4:  django-admin startproject todoproject .
✅ Step 5:  python manage.py startapp todo
✅ Step 6:  'todo' INSTALLED_APPS me add kiya
✅ Step 7:  models.py me Task aur Category model banaya
✅ Step 8:  makemigrations aur migrate run kiya
✅ Step 9:  admin.py configure kiya
✅ Step 10: superuser banaya
✅ Step 11: forms.py me TaskForm banaya
✅ Step 12: views.py me CBVs banaye
✅ Step 13: todo/urls.py banaya
✅ Step 14: todoproject/urls.py me include kiya
✅ Step 15: 4 templates banaye (base, list, form, delete)
✅ Step 16: static/todo/css/style.css banaya
✅ Step 17: python manage.py runserver
✅ Step 18: Browser me http://127.0.0.1:8000/ khola
✅ Step 19: Task add, edit, delete, toggle test kiya
✅ Step 20: 🎉 App complete!
```

---

## 🔥 Aage Kya Seekhein?

App complete karne ke baad yeh features add kar sakte ho:

1. **User Authentication** — Har user ke apne tasks (`django.contrib.auth` use karo)
2. **Pagination** — Bahut saari tasks hone par pages me divide karo
3. **REST API** — Django REST Framework se mobile app ke liye API banao
4. **PostgreSQL** — Production ke liye SQLite se upgrade karo
5. **HTMX** — Page reload ke bina dynamic updates
6. **Task Sharing** — Ek task multiple users ke saath share karo

---

*Guide complete karne ke baad ek screenshot lo — tumhara pehla Django project ready hai!* 🚀
