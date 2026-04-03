# Django Class-Based Views (CBVs) — Complete Guide

> **FBV se CBV tak** — Ek developer ka complete roadmap with real-world use case

---

## Table of Contents

1. [FBV vs CBV — Kya Fark Hai?](#1-fbv-vs-cbv--kya-fark-hai)
2. [URL reverse() aur get_absolute_url](#2-url-reverse-aur-get_absolute_url)
3. [Class-Based Views — Core Concepts](#3-class-based-views--core-concepts)
4. [CRUD with CBVs — Full Reference](#4-crud-with-cbvs--full-reference)
5. [End-to-End Use Case: Blog App](#5-end-to-end-use-case-blog-app)
6. [Mixins aur Permissions](#6-mixins-aur-permissions)
7. [CBV Customization Tips](#7-cbv-customization-tips)
8. [Quick Reference Cheat Sheet](#8-quick-reference-cheat-sheet)

---

## 1. FBV vs CBV — Kya Fark Hai?

### Function-Based View (FBV) — Purana Tarika

```python
# views.py
from django.shortcuts import render, get_object_or_404, redirect
from .models import Post
from .forms import PostForm

def post_list(request):
    posts = Post.objects.all()
    return render(request, 'blog/post_list.html', {'posts': posts})

def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    return render(request, 'blog/post_detail.html', {'post': post})

def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('post-list')
    else:
        form = PostForm()
    return render(request, 'blog/post_form.html', {'form': form})
```

### Class-Based View (CBV) — Naya Tarika

```python
# views.py
from django.views.generic import ListView, DetailView, CreateView
from django.urls import reverse_lazy
from .models import Post

class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'

class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'

class PostCreateView(CreateView):
    model = Post
    fields = ['title', 'content', 'author']
    template_name = 'blog/post_form.html'
    success_url = reverse_lazy('post-list')
```

### Comparison Table

| Feature | FBV | CBV |
|---|---|---|
| Code length | Zyada | Kam |
| Reusability | Kam | Zyada (inheritance) |
| Readability (simple) | Behtar | Thoda complex |
| Readability (complex) | Complex | Structured |
| Mixins support | Manual | Built-in |
| Django ka recommendation | Chal sakta hai | Preferred for CRUD |

---

## 2. URL `reverse()` aur `get_absolute_url`

Yeh do cheezein CBVs ke saath bohot zarori hain. Inhe samjhe bina redirects aur links kaam nahi karenge.

---

### 2.1 `reverse()` — URL ko Name se Generate Karo

`reverse()` function URL patterns ka naam lekar actual URL string return karta hai.

**urls.py mein URL ka naam define karo:**

```python
# blog/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.PostListView.as_view(), name='post-list'),
    path('<int:pk>/', views.PostDetailView.as_view(), name='post-detail'),
    path('create/', views.PostCreateView.as_view(), name='post-create'),
    path('<int:pk>/update/', views.PostUpdateView.as_view(), name='post-update'),
    path('<int:pk>/delete/', views.PostDeleteView.as_view(), name='post-delete'),
]
```

**`reverse()` use karo views.py mein:**

```python
from django.urls import reverse

# Kisi specific post ki URL banao
url = reverse('post-detail', kwargs={'pk': 5})
# Result: '/blog/5/'

# Args se bhi ho sakta hai
url = reverse('post-detail', args=[5])
# Result: '/blog/5/'

# Namespace ke saath (agar app namespace use kar rahe ho)
url = reverse('blog:post-detail', kwargs={'pk': 5})
```

**`reverse_lazy()` — Views ke andar use karo:**

`reverse()` ko class level pe directly call nahi kar sakte (class load hone se pehle URL patterns ready nahi hote). Isliye `reverse_lazy()` use karte hain:

```python
from django.urls import reverse_lazy

class PostCreateView(CreateView):
    model = Post
    fields = ['title', 'content']
    success_url = reverse_lazy('post-list')  # ✅ Sahi tarika

# YEH GALAT HAI:
class PostCreateView(CreateView):
    success_url = reverse('post-list')  # ❌ Error aayega
```

---

### 2.2 `get_absolute_url` — Model ka Apna URL

`get_absolute_url()` ek model method hai jo us object ki canonical URL return karta hai. Yeh Django ka best practice hai.

**Model mein define karo:**

```python
# blog/models.py
from django.db import models
from django.urls import reverse

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('post-detail', kwargs={'pk': self.pk})
```

**Kahan kahan use hota hai:**

```python
# 1. Template mein seedha call kar sakte ho:
# {{ post.get_absolute_url }}
# <a href="{{ post.get_absolute_url }}">Read More</a>

# 2. CreateView / UpdateView mein success_url ki zaroorat nahi agar model mein ho:
class PostCreateView(CreateView):
    model = Post
    fields = ['title', 'content']
    # success_url ki zaroorat nahi! Django khud get_absolute_url() call karega

# 3. Views mein use:
def some_view(request):
    post = Post.objects.get(pk=1)
    return redirect(post.get_absolute_url())
    # Ya simply: return redirect(post)  # Django internally get_absolute_url() call karta hai
```

**Template mein use:**

```html
<!-- blog/post_list.html -->
{% for post in posts %}
  <div class="post">
    <h2>{{ post.title }}</h2>
    <a href="{{ post.get_absolute_url }}">Detail Dekho</a>
    <!-- Ya template tag se: -->
    <a href="{% url 'post-detail' post.pk %}">Detail Dekho</a>
  </div>
{% endfor %}
```

> **Best Practice:** Hamesha `get_absolute_url()` model mein define karo. Isse templates clean rehte hain aur URL structure change karne pe sirf ek jagah update karna padta hai.

---

## 3. Class-Based Views — Core Concepts

### 3.1 `as_view()` — CBV ko URL se Connect Karo

CBVs directly URL mein nahi diye ja sakte. `as_view()` unhe callable banata hai:

```python
# urls.py
from django.urls import path
from .views import PostListView

urlpatterns = [
    path('posts/', PostListView.as_view(), name='post-list'),
    #                    ^^^^^^^^^^^^ zarori hai
]
```

### 3.2 Built-in Generic Views ka Family Tree

```
View (Base)
│
├── TemplateView          → Sirf template render karo
├── RedirectView          → Kisi URL pe redirect karo
│
├── ListView              → Objects ki list dikhaao
├── DetailView            → Ek object ki detail
│
├── CreateView            → Naya object banao (form)
├── UpdateView            → Existing object update karo (form)
├── DeleteView            → Object delete karo
│
└── FormView              → Custom form handle karo (model ke bina)
```

### 3.3 Template Naming Convention

CBVs automatically template dhundhte hain agar `template_name` nahi diya:

```
Model: Post  (blog app mein)
ListView    → blog/post_list.html
DetailView  → blog/post_detail.html
CreateView  → blog/post_form.html
UpdateView  → blog/post_form.html
DeleteView  → blog/post_confirm_delete.html
```

### 3.4 Context Object Naming

```python
# ListView mein default context name hota hai: object_list
# Apna naam dene ke liye:
class PostListView(ListView):
    model = Post
    context_object_name = 'posts'  # Template mein {{ posts }} use karo

# DetailView mein default: object
class PostDetailView(DetailView):
    model = Post
    context_object_name = 'post'  # Template mein {{ post }} use karo
```

---

## 4. CRUD with CBVs — Full Reference

### 4.1 ListView — Records ki List

```python
from django.views.generic import ListView
from .models import Post

class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    ordering = ['-created_at']          # Latest pehle
    paginate_by = 10                     # 10 per page pagination

    # Queryset customize karna ho to:
    def get_queryset(self):
        return Post.objects.filter(is_published=True).order_by('-created_at')

    # Extra context add karna ho to:
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['total_posts'] = Post.objects.count()
        context['page_title'] = 'All Posts'
        return context
```

**Template:**

```html
<!-- blog/post_list.html -->
<h1>{{ page_title }} ({{ total_posts }})</h1>

{% for post in posts %}
  <div>
    <h2><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></h2>
    <p>{{ post.created_at }}</p>
  </div>
{% empty %}
  <p>Koi post nahi mili.</p>
{% endfor %}

<!-- Pagination -->
{% if is_paginated %}
  {% if page_obj.has_previous %}
    <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
  {% endif %}
  Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}
  {% if page_obj.has_next %}
    <a href="?page={{ page_obj.next_page_number }}">Next</a>
  {% endif %}
{% endif %}
```

---

### 4.2 DetailView — Single Record ki Detail

```python
from django.views.generic import DetailView

class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    # pk ki jagah slug use karna ho to:
    # slug_field = 'slug'
    # slug_url_kwarg = 'slug'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['related_posts'] = Post.objects.exclude(pk=self.object.pk)[:3]
        return context
```

**Template:**

```html
<!-- blog/post_detail.html -->
<h1>{{ post.title }}</h1>
<p>{{ post.content }}</p>

<h3>Related Posts</h3>
{% for rpost in related_posts %}
  <a href="{{ rpost.get_absolute_url }}">{{ rpost.title }}</a>
{% endfor %}

<a href="{% url 'post-update' post.pk %}">Edit</a>
<a href="{% url 'post-delete' post.pk %}">Delete</a>
```

---

### 4.3 CreateView — Naya Record Banao

```python
from django.views.generic import CreateView
from django.urls import reverse_lazy

class PostCreateView(CreateView):
    model = Post
    template_name = 'blog/post_form.html'
    fields = ['title', 'content', 'category']
    # Ya form_class use karo: form_class = PostForm
    success_url = reverse_lazy('post-list')

    # Form submit hone se pehle kuch karna ho (e.g., author set karna):
    def form_valid(self, form):
        form.instance.author = self.request.user  # Logged-in user ko author set karo
        return super().form_valid(form)
```

**Template:**

```html
<!-- blog/post_form.html -->
<h1>{% if object %}Post Update Karo{% else %}Naya Post{% endif %}</h1>

<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Save</button>
</form>

<a href="{% url 'post-list' %}">Cancel</a>
```

---

### 4.4 UpdateView — Record Update Karo

```python
from django.views.generic import UpdateView

class PostUpdateView(UpdateView):
    model = Post
    template_name = 'blog/post_form.html'
    fields = ['title', 'content', 'category']
    # success_url ki zaroorat nahi agar model mein get_absolute_url() hai

    def form_valid(self, form):
        # Update ke time kuch extra karna ho
        return super().form_valid(form)
```

---

### 4.5 DeleteView — Record Delete Karo

```python
from django.views.generic import DeleteView
from django.urls import reverse_lazy

class PostDeleteView(DeleteView):
    model = Post
    template_name = 'blog/post_confirm_delete.html'
    success_url = reverse_lazy('post-list')

    # Delete se pehle permission check (custom):
    def dispatch(self, request, *args, **kwargs):
        post = self.get_object()
        if post.author != request.user:
            from django.core.exceptions import PermissionDenied
            raise PermissionDenied
        return super().dispatch(request, *args, **kwargs)
```

**Template:**

```html
<!-- blog/post_confirm_delete.html -->
<h1>Post Delete Karo?</h1>
<p>"{{ post.title }}" permanently delete ho jayega.</p>

<form method="post">
  {% csrf_token %}
  <button type="submit" style="color:red;">Haan, Delete Karo</button>
  <a href="{{ post.get_absolute_url }}">Cancel</a>
</form>
```

---

### 4.6 TemplateView aur RedirectView

```python
from django.views.generic import TemplateView, RedirectView

# Sirf ek template render karna ho:
class AboutView(TemplateView):
    template_name = 'about.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['team_members'] = ['Ali', 'Sara', 'Bilal']
        return context


# Kisi URL pe redirect karna ho:
class OldPostRedirect(RedirectView):
    pattern_name = 'post-list'  # URL name se redirect
    permanent = False            # 301 ya 302?
```

---

## 5. End-to-End Use Case: Blog App

Aik complete **Blog Application** banate hain jisme posts, categories, aur authentication sab kuch hoga.

### 5.1 Project Structure

```
myblog/
├── manage.py
├── myblog/
│   ├── settings.py
│   ├── urls.py
├── blog/
│   ├── models.py
│   ├── views.py
│   ├── urls.py
│   ├── forms.py
│   └── templates/
│       └── blog/
│           ├── base.html
│           ├── post_list.html
│           ├── post_detail.html
│           ├── post_form.html
│           └── post_confirm_delete.html
```

---

### 5.2 Models

```python
# blog/models.py
from django.db import models
from django.urls import reverse
from django.contrib.auth.models import User


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True)

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse('blog:category-detail', kwargs={'slug': self.slug})

    class Meta:
        verbose_name_plural = 'Categories'


class Post(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
    ]

    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True, blank=True)
    content = models.TextField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse('blog:post-detail', kwargs={'slug': self.slug})

    class Meta:
        ordering = ['-created_at']
```

---

### 5.3 Forms

```python
# blog/forms.py
from django import forms
from .models import Post


class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'slug', 'category', 'content', 'status']
        widgets = {
            'title': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'Post title'}),
            'slug': forms.TextInput(attrs={'class': 'form-control'}),
            'content': forms.Textarea(attrs={'class': 'form-control', 'rows': 10}),
            'status': forms.Select(attrs={'class': 'form-select'}),
            'category': forms.Select(attrs={'class': 'form-select'}),
        }
```

---

### 5.4 Views — Poora CRUD

```python
# blog/views.py
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView, TemplateView
)
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.urls import reverse_lazy
from django.db.models import Q
from .models import Post, Category
from .forms import PostForm


# ─────────────────────────────────────────
# HOME / DASHBOARD
# ─────────────────────────────────────────
class HomeView(TemplateView):
    template_name = 'blog/home.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['recent_posts'] = Post.objects.filter(status='published')[:5]
        context['categories'] = Category.objects.all()
        return context


# ─────────────────────────────────────────
# POST LIST — Published posts + Search
# ─────────────────────────────────────────
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 6

    def get_queryset(self):
        queryset = Post.objects.filter(status='published')

        # Search functionality
        search_query = self.request.GET.get('q')
        if search_query:
            queryset = queryset.filter(
                Q(title__icontains=search_query) |
                Q(content__icontains=search_query)
            )

        # Category filter
        category_slug = self.kwargs.get('category_slug')
        if category_slug:
            queryset = queryset.filter(category__slug=category_slug)

        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['search_query'] = self.request.GET.get('q', '')
        context['categories'] = Category.objects.all()
        context['selected_category'] = self.kwargs.get('category_slug')
        return context


# ─────────────────────────────────────────
# POST DETAIL
# ─────────────────────────────────────────
class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'

    def get_queryset(self):
        # Only published posts visible to non-authors
        if self.request.user.is_authenticated:
            return Post.objects.filter(
                Q(status='published') | Q(author=self.request.user)
            )
        return Post.objects.filter(status='published')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        post = self.object
        context['related_posts'] = Post.objects.filter(
            category=post.category,
            status='published'
        ).exclude(pk=post.pk)[:3]
        return context


# ─────────────────────────────────────────
# CREATE — Login required
# ─────────────────────────────────────────
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'
    login_url = '/accounts/login/'

    def form_valid(self, form):
        form.instance.author = self.request.user  # Author automatically set
        return super().form_valid(form)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['form_title'] = 'Naya Post Likho'
        context['submit_label'] = 'Publish'
        return context


# ─────────────────────────────────────────
# UPDATE — Only author can edit
# ─────────────────────────────────────────
class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'

    def test_func(self):
        post = self.get_object()
        return self.request.user == post.author  # Sirf author edit kar sakta hai

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['form_title'] = f'"{self.object.title}" Update Karo'
        context['submit_label'] = 'Update'
        return context


# ─────────────────────────────────────────
# DELETE — Only author can delete
# ─────────────────────────────────────────
class PostDeleteView(LoginRequiredMixin, UserPassesTestMixin, DeleteView):
    model = Post
    template_name = 'blog/post_confirm_delete.html'
    success_url = reverse_lazy('blog:post-list')

    def test_func(self):
        post = self.get_object()
        return self.request.user == post.author


# ─────────────────────────────────────────
# MY POSTS — Logged-in user ke apne posts
# ─────────────────────────────────────────
class MyPostsView(LoginRequiredMixin, ListView):
    model = Post
    template_name = 'blog/my_posts.html'
    context_object_name = 'posts'

    def get_queryset(self):
        return Post.objects.filter(author=self.request.user).order_by('-created_at')
```

---

### 5.5 URLs

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'  # Namespace — reverse mein 'blog:post-list' likhenge

urlpatterns = [
    # Home
    path('', views.HomeView.as_view(), name='home'),

    # Post CRUD
    path('posts/', views.PostListView.as_view(), name='post-list'),
    path('posts/<slug:slug>/', views.PostDetailView.as_view(), name='post-detail'),
    path('posts/create/', views.PostCreateView.as_view(), name='post-create'),
    path('posts/<slug:slug>/update/', views.PostUpdateView.as_view(), name='post-update'),
    path('posts/<slug:slug>/delete/', views.PostDeleteView.as_view(), name='post-delete'),

    # My Posts
    path('my-posts/', views.MyPostsView.as_view(), name='my-posts'),

    # Category filter
    path('category/<slug:category_slug>/', views.PostListView.as_view(), name='category-detail'),
]
```

```python
# myblog/urls.py (main)
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls', namespace='blog')),
    path('accounts/', include('django.contrib.auth.urls')),
]
```

---

### 5.6 Templates

```html
<!-- blog/base.html -->
<!DOCTYPE html>
<html lang="ur">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}My Blog{% endblock %}</title>
</head>
<body>
  <nav>
    <a href="{% url 'blog:home' %}">Home</a>
    <a href="{% url 'blog:post-list' %}">Posts</a>
    {% if user.is_authenticated %}
      <a href="{% url 'blog:post-create' %}">New Post</a>
      <a href="{% url 'blog:my-posts' %}">My Posts</a>
      <a href="{% url 'logout' %}">Logout ({{ user.username }})</a>
    {% else %}
      <a href="{% url 'login' %}">Login</a>
    {% endif %}
  </nav>

  <main>
    {% if messages %}
      {% for message in messages %}
        <div class="alert alert-{{ message.tags }}">{{ message }}</div>
      {% endfor %}
    {% endif %}

    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

```html
<!-- blog/post_list.html -->
{% extends 'blog/base.html' %}

{% block title %}Posts - My Blog{% endblock %}

{% block content %}
<h1>All Posts</h1>

<!-- Search Form -->
<form method="get">
  <input type="text" name="q" value="{{ search_query }}" placeholder="Search posts...">
  <button type="submit">Search</button>
</form>

<!-- Categories -->
<div>
  <a href="{% url 'blog:post-list' %}">All</a>
  {% for cat in categories %}
    <a href="{% url 'blog:category-detail' cat.slug %}">{{ cat.name }}</a>
  {% endfor %}
</div>

<!-- Posts Grid -->
{% for post in posts %}
  <article>
    <h2><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></h2>
    <p>By {{ post.author.username }} | {{ post.created_at|date:"d M Y" }}</p>
    <p>{{ post.content|truncatewords:30 }}</p>
    <a href="{{ post.get_absolute_url }}">Read More →</a>
  </article>
{% empty %}
  <p>Koi post nahi mili.</p>
{% endfor %}

<!-- Pagination -->
{% if is_paginated %}
  <div class="pagination">
    {% if page_obj.has_previous %}
      <a href="?page={{ page_obj.previous_page_number }}{% if search_query %}&q={{ search_query }}{% endif %}">← Prev</a>
    {% endif %}
    <span>{{ page_obj.number }} / {{ page_obj.paginator.num_pages }}</span>
    {% if page_obj.has_next %}
      <a href="?page={{ page_obj.next_page_number }}{% if search_query %}&q={{ search_query }}{% endif %}">Next →</a>
    {% endif %}
  </div>
{% endif %}
{% endblock %}
```

```html
<!-- blog/post_detail.html -->
{% extends 'blog/base.html' %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
<article>
  <h1>{{ post.title }}</h1>
  <p>
    By <strong>{{ post.author.username }}</strong>
    | {{ post.created_at|date:"d M Y, H:i" }}
    {% if post.category %}
      | Category: <a href="{% url 'blog:category-detail' post.category.slug %}">{{ post.category.name }}</a>
    {% endif %}
  </p>

  <div>{{ post.content|linebreaks }}</div>

  <!-- Sirf author ko edit/delete dikhao -->
  {% if user == post.author %}
    <div>
      <a href="{% url 'blog:post-update' post.slug %}">✏️ Edit</a>
      <a href="{% url 'blog:post-delete' post.slug %}">🗑️ Delete</a>
    </div>
  {% endif %}
</article>

<!-- Related Posts -->
{% if related_posts %}
  <section>
    <h3>Related Posts</h3>
    {% for rpost in related_posts %}
      <a href="{{ rpost.get_absolute_url }}">{{ rpost.title }}</a>
    {% endfor %}
  </section>
{% endif %}

<a href="{% url 'blog:post-list' %}">← Back to List</a>
{% endblock %}
```

```html
<!-- blog/post_form.html -->
{% extends 'blog/base.html' %}

{% block title %}{{ form_title }}{% endblock %}

{% block content %}
<h1>{{ form_title }}</h1>

<form method="post">
  {% csrf_token %}

  {% for field in form %}
    <div>
      {{ field.label_tag }}
      {{ field }}
      {% if field.errors %}
        <ul>
          {% for error in field.errors %}
            <li style="color:red;">{{ error }}</li>
          {% endfor %}
        </ul>
      {% endif %}
    </div>
  {% endfor %}

  <button type="submit">{{ submit_label }}</button>
  <a href="{% url 'blog:post-list' %}">Cancel</a>
</form>
{% endblock %}
```

```html
<!-- blog/post_confirm_delete.html -->
{% extends 'blog/base.html' %}

{% block title %}Post Delete Karo{% endblock %}

{% block content %}
<div>
  <h1>⚠️ Post Delete Karo?</h1>
  <p>Kya aap sure hain ke <strong>"{{ post.title }}"</strong> permanently delete karna chahte hain?</p>
  <p>Yeh action undo nahi ho sakta.</p>

  <form method="post">
    {% csrf_token %}
    <button type="submit" style="color: red;">Haan, Delete Karo</button>
    <a href="{{ post.get_absolute_url }}">Nahi, Wapas Jao</a>
  </form>
</div>
{% endblock %}
```

---

## 6. Mixins aur Permissions

Mixins reusable "add-on" classes hain jo views mein extra functionality dete hain.

### 6.1 LoginRequiredMixin

```python
from django.contrib.auth.mixins import LoginRequiredMixin

class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    fields = ['title', 'content']
    login_url = '/accounts/login/'       # Default: settings.LOGIN_URL
    redirect_field_name = 'next'         # Login ke baad kahan jana hai
```

### 6.2 UserPassesTestMixin

```python
from django.contrib.auth.mixins import UserPassesTestMixin

class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    model = Post
    fields = ['title', 'content']

    def test_func(self):
        post = self.get_object()
        # True return karo agar permission hai, False pe 403 Forbidden
        return self.request.user == post.author or self.request.user.is_staff
```

### 6.3 Custom Mixin Banao

```python
# mixins.py
class AuthorRequiredMixin:
    """Sirf post ka author hi edit/delete kar sakta hai"""

    def dispatch(self, request, *args, **kwargs):
        post = self.get_object()
        if post.author != request.user and not request.user.is_staff:
            from django.core.exceptions import PermissionDenied
            raise PermissionDenied("Aap sirf apne posts edit kar sakte hain.")
        return super().dispatch(request, *args, **kwargs)


# Use karo:
class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    model = Post
    fields = ['title', 'content']
```

---

## 7. CBV Customization Tips

### 7.1 `dispatch()` — Har Request Se Pehle

```python
class PostDetailView(DetailView):
    model = Post

    def dispatch(self, request, *args, **kwargs):
        # Har request pe run hota hai (GET, POST, dono)
        # Permission check ya redirect yahan karo
        if not request.user.is_active:
            return redirect('login')
        return super().dispatch(request, *args, **kwargs)
```

### 7.2 `get_object()` — Object Fetch Customize Karo

```python
class PostDetailView(DetailView):
    model = Post

    def get_object(self):
        # Default pk ya slug se fetch karta hai
        # Custom logic add kar sakte ho
        obj = super().get_object()
        # Page views count karo
        obj.view_count = (obj.view_count or 0) + 1
        obj.save(update_fields=['view_count'])
        return obj
```

### 7.3 `form_valid()` vs `form_invalid()`

```python
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm

    def form_valid(self, form):
        # Form valid hone pe — save se pehle ya baad kuch karo
        form.instance.author = self.request.user
        response = super().form_valid(form)
        messages.success(self.request, f'Post "{self.object.title}" create ho gaya!')
        return response

    def form_invalid(self, form):
        # Form invalid hone pe
        messages.error(self.request, 'Form mein errors hain. Please check karo.')
        return super().form_invalid(form)
```

---

## 8. Quick Reference Cheat Sheet

```
┌─────────────────┬──────────────────────────┬──────────────────────────────┐
│ CBV Class       │ Use Case                 │ Key Attributes               │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ ListView        │ Objects ki list          │ model, queryset,             │
│                 │                          │ context_object_name,         │
│                 │                          │ paginate_by, ordering        │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ DetailView      │ Single object detail     │ model, slug_field,           │
│                 │                          │ slug_url_kwarg               │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ CreateView      │ New object form          │ model, fields, form_class,   │
│                 │                          │ success_url                  │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ UpdateView      │ Existing object edit     │ model, fields, form_class    │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ DeleteView      │ Object delete confirm    │ model, success_url           │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ TemplateView    │ Static/simple pages      │ template_name                │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ FormView        │ Custom form (no model)   │ form_class, success_url      │
├─────────────────┼──────────────────────────┼──────────────────────────────┤
│ RedirectView    │ URL redirect             │ url, pattern_name, permanent │
└─────────────────┴──────────────────────────┴──────────────────────────────┘
```

### Key Methods Override Order

```
Request aata hai
       ↓
dispatch()          ← Pehle yahan aao (permission check)
       ↓
get() / post()      ← HTTP method ke hisab se
       ↓
get_object()        ← Object fetch (Detail/Update/Delete)
get_queryset()      ← Queryset (List)
       ↓
get_context_data()  ← Template ko data do
       ↓
render_to_response() ← Template render karo
```

### `reverse` vs `reverse_lazy`

```python
# reverse() — Views/functions ke andar use karo
from django.urls import reverse

def my_view(request):
    url = reverse('blog:post-list')  # ✅
    return redirect(url)

# reverse_lazy() — Class attributes mein use karo
from django.urls import reverse_lazy

class MyView(CreateView):
    success_url = reverse_lazy('blog:post-list')  # ✅
    # success_url = reverse('blog:post-list')      # ❌ Error!
```

### Common Mistakes aur Solutions

```python
# ❌ GALAT: template name bhool gaye
class PostListView(ListView):
    model = Post
    # Django dhundhega: blog/post_list.html (app/model_viewtype.html)

# ✅ SAHI: explicit template name
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'  # Hamesha explicit rakho

# ❌ GALAT: success_url mein reverse()
class PostCreateView(CreateView):
    success_url = reverse('post-list')  # ImportError ya NoReverseMatch

# ✅ SAHI: reverse_lazy() ya get_absolute_url()
class PostCreateView(CreateView):
    success_url = reverse_lazy('post-list')

# ❌ GALAT: Mixin order galat
class MyView(UserPassesTestMixin, LoginRequiredMixin, View):
    pass  # LoginRequired check nahi hoga pehle

# ✅ SAHI: LoginRequired pehle
class MyView(LoginRequiredMixin, UserPassesTestMixin, View):
    pass
```

---

## Summary

| Concept | Short Summary |
|---|---|
| `reverse('name')` | URL name se actual URL string banao — views ke andar |
| `reverse_lazy('name')` | Same — lekin class attributes mein use karo |
| `get_absolute_url()` | Model method — object ki apni canonical URL return kare |
| `ListView` | `.objects.all()` style list + pagination |
| `DetailView` | Ek object pk/slug se fetch karo |
| `CreateView` | Form dikhao + save karo + redirect karo |
| `UpdateView` | Existing object form mein load karo + update karo |
| `DeleteView` | Confirm page dikhao + delete karo |
| `LoginRequiredMixin` | Login check — anonymous ko login page pe bhejo |
| `UserPassesTestMixin` | Custom permission check — `test_func()` define karo |
| `form_valid()` | Form save hone ke baad kuch extra karo |
| `get_context_data()` | Template ko extra variables do |

---

> **Pro Tip:** Django ka official documentation aur [CCBV.co.uk](https://ccbv.co.uk) — Class-Based Views ka visual reference — bookmark kar lo. CBVs ki har method ka breakdown wahan milega.
