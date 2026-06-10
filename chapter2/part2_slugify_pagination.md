### 🛠️ `slugify()` — Creating Slugs Automatically

`slugify()` **converts a title into a valid slug**.

```python
from django.utils.text import slugify

print(slugify("Learn Django Admin"))    # learn-django-admin
print(slugify("Hello World!"))          # hello-world
print(slugify("Django 5: Tips & Tricks")) # django-5-tips-tricks
print(slugify("مرحبا بالعالم"))         # (empty — Arabic not slug-safe by default)
```

> ⚠️ **Gotcha with Arabic/non-ASCII text:**  
> `slugify()` removes non-ASCII characters. For Arabic titles, use:
> ```python
> from django.utils.text import slugify
> slugify("مرحبا", allow_unicode=True)  # مرحبا
> ```

---

### 🔧 Auto-Generate Slug on Save

```python
# blog/models.py
from django.db import models
from django.utils.text import slugify

class Post(models.Model):
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250, unique=True)

    def save(self, *args, **kwargs):
        if not self.slug:                    # Only generate if slug is empty
            self.slug = slugify(self.title)  # "My Post" → "my-post"
        super().save(*args, **kwargs)
```

**What happens:**
```python
Post.objects.create(title="Learn Django Admin")
# slug is automatically set to: "learn-django-admin"
```

---

### 💡 SlugField vs slugify() — They Are NOT the Same

| | `slugify()` | `SlugField` |
|---|---|---|
| **What it does** | **Creates** a slug from a string | **Stores** a slug in the database |
| **Type** | Function | Model field |
| **Usage** | `slug = slugify(title)` | `slug = models.SlugField()` |

**They work together:**
```
Title "Learn Django" ──slugify()──► "learn-django" ──SlugField──► Saved in DB
```

---

### 🎯 Interview Question

> **Q: Can `SlugField` be replaced by `CharField`?**

**A:** Yes, technically. But `SlugField` is preferred because:
1. It automatically **validates** that the value is a proper slug
2. It makes the code **self-documenting** — any developer knows the purpose instantly
3. Django Admin renders a **special widget** that auto-fills the slug from the title
4. **ModelForms** get slug validation for free

---

## 5. Adding Pagination to the Post List

### 🤔 Why Pagination?

Without pagination, your view returns **all** posts in one response.  
With 1,000 posts, that's:
- 🐢 Slow page load
- 💾 Unnecessary memory use
- 😵 Terrible user experience

Pagination returns posts in **pages** (e.g., 3 per page).

---

### Step-by-Step: Adding Pagination

```python
# blog/views.py
from django.core.paginator import EmptyPage, PageNotAnInteger, Paginator
from django.shortcuts import render
from .models import Post

def post_list(request):
    all_posts = Post.published.all()         # 1. Get all published posts

    paginator = Paginator(all_posts, 3)      # 2. Show 3 posts per page

    page_number = request.GET.get('page')    # 3. Get page number from URL: ?page=2

    try:
        posts = paginator.page(page_number)  # 4. Get the Page object
    except PageNotAnInteger:
        # If ?page=abc (not a number) → show page 1
        posts = paginator.page(1)
    except EmptyPage:
        # If ?page=999 (too high) → show last page
        posts = paginator.page(paginator.num_pages)

    return render(request, 'blog/post/list.html', {'posts': posts})
```

---

### 🗺️ How Pagination Works (Visual Flow)

```
URL: /blog/?page=2
         │
         ▼
request.GET.get('page')  →  "2"
         │
         ▼
paginator.page("2")  →  Page object (posts 4, 5, 6)
         │
         ▼
Passed to template as 'posts'
```

---

### 📊 Key Paginator Attributes

```python
paginator = Paginator(queryset, 3)

paginator.num_pages       # Total number of pages → 4
paginator.count           # Total number of objects → 10
paginator.page_range      # range(1, 5) → [1, 2, 3, 4]

page = paginator.page(2)

page.number               # Current page number → 2
page.has_previous()       # True/False
page.has_next()           # True/False
page.previous_page_number() # 1
page.next_page_number()   # 3
page.object_list          # QuerySet of objects on this page
```

---

### 📄 Pagination Template: `pagination.html`

Create a reusable template snippet:

```html
<!-- templates/pagination.html -->
<div class="pagination">
    <span class="step-links">

        {% if page.has_previous %}
            <a href="?page=1">« First</a>
            <a href="?page={{ page.previous_page_number }}">Previous</a>
        {% endif %}

        <span class="current">
            Page {{ page.number }} of {{ page.paginator.num_pages }}
        </span>

        {% if page.has_next %}
            <a href="?page={{ page.next_page_number }}">Next</a>
            <a href="?page={{ page.paginator.num_pages }}">Last »</a>
        {% endif %}

    </span>
</div>
```

**Include it in your list template:**

```html
<!-- templates/blog/post/list.html -->
{% extends "blog/base.html" %}

{% block content %}
    <h1>My Blog</h1>

    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
        </h2>
        <p class="date">Published {{ post.publish }} by {{ post.author }}</p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}

    {% include "pagination.html" with page=posts %}
{% endblock %}
```

> **Note:** We pass `page=posts` because `posts` is already a `Page` object from `paginator.page()`.

---

## 6. Handling Pagination Errors

### ❌ What Errors Can Occur?

| Scenario                  | Error Raised        | Example URL            |
|--------------------------|--------------------|-----------------------|
| Page number too high     | `EmptyPage`        | `/blog/?page=999`     |
| Page number not a number | `PageNotAnInteger` | `/blog/?page=abc`     |
| No page parameter        | `PageNotAnInteger` | `/blog/`              |

### ✅ Complete Error Handling

```python
from django.core.paginator import EmptyPage, PageNotAnInteger, Paginator

try:
    posts = paginator.page(page_number)
except PageNotAnInteger:
    posts = paginator.page(1)          # Default to first page
except EmptyPage:
    posts = paginator.page(paginator.num_pages)  # Default to last page
```

---

## 7. Class-Based Views (CBVs) — ListView

### 🤔 Function-Based View vs Class-Based View

**Function-Based View (FBV)** — what we wrote first:

```python
def post_list(request):
    all_posts = Post.published.all()
    paginator = Paginator(all_posts, 3)
    # ... lots of boilerplate ...
    return render(request, 'blog/post/list.html', {'posts': posts})
```

**Class-Based View (CBV)** — the same thing, much shorter:

```python
from django.views.generic import ListView

class PostListView(ListView):
    queryset = Post.published.all()
    context_object_name = 'posts'
    paginate_by = 3
    template_name = 'blog/post/list.html'
```

> **One class = the entire FBV**, including pagination handling!

---

### 🔍 Understanding Each CBV Attribute

```python
class PostListView(ListView):

    # Which objects to list (instead of Post.objects.all())
    queryset = Post.published.all()

    # What name to use in the template (default: 'object_list')
    context_object_name = 'posts'

    # How many items per page (built-in pagination!)
    paginate_by = 3

    # Which template to use (default for Post: 'blog/post_list.html')
    template_name = 'blog/post/list.html'
```

---

### 🔗 Connecting CBV to URLs

```python
# blog/urls.py
from django.urls import path
from . import views

urlpatterns = [
    # Function-based (commented out)
    # path('', views.post_list, name='post_list'),

    # Class-based — note the .as_view() call!
    path('', views.PostListView.as_view(), name='post_list'),

    path(
        '<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'
    ),
]
```

> **Important:** Class-based views need `.as_view()` when connecting to URL patterns.  
> `as_view()` converts the class into a callable view function.

---

### 📄 Template Update for CBV Pagination

When using `ListView`, Django automatically passes the page as `page_obj` (not your custom variable).

```html
<!-- ✅ CORRECT for CBV -->
{% include "pagination.html" with page=page_obj %}

<!-- ❌ WRONG — page_obj is what ListView provides, not 'posts' -->
{% include "pagination.html" with page=posts %}
```

---

### 📊 CBV vs FBV — When to Use Which

| | FBV | CBV |
|---|---|---|
| **Simplicity** | ✅ Easy to understand | ⚠️ More to learn initially |
| **Boilerplate** | ❌ More code | ✅ Much less code |
| **Custom logic** | ✅ Very flexible | ⚠️ Need to override methods |
| **Pagination** | Manual | ✅ Built-in (`paginate_by`) |
| **Reusability** | ❌ Copy-paste | ✅ Inheritance & mixins |
| **Best for** | Complex custom logic | Standard list/detail/create/update/delete |

---

### ⚠️ CBV Error Handling Difference

With FBV, we handled `EmptyPage` / `PageNotAnInteger` manually.

With `ListView`, Django handles these automatically:
- Out-of-range page → **HTTP 404**
- Non-integer page → **HTTP 404**

```
# FBV: You control the error response
# CBV: Django returns 404 automatically
```

---

## 8. Summary Comparison Table

| Concept              | Key Point                                                        |
|---------------------|------------------------------------------------------------------|
| Canonical URL        | The "official" URL for a piece of content                       |
| `get_absolute_url()` | Model method that returns the canonical URL                     |
| `reverse()`          | Builds URL from name — survives URL pattern changes             |
| `{% url %}`          | Template tag equivalent of `reverse()` — for templates only    |
| `unique_for_date`    | Ensures slugs are unique per day (app-level, not DB-level)      |
| `SlugField`          | CharField + slug validation; use for URL-friendly strings       |
| `slugify()`          | Function that converts text to slug format                      |
| `Paginator`          | Django class that splits a queryset into pages                  |
| `paginate_by`        | ListView attribute that enables pagination automatically        |
| `page_obj`           | The page variable provided by `ListView` to templates           |
| `.as_view()`         | Converts a CBV class into a callable for URL configuration      |

---

## 9. Common Mistakes & Gotchas

### ❌ Mistake 1: Using `{% url %}` in Python Code

```python
# ❌ WRONG
return redirect("{% url 'blog:post_detail' post.id %}")

# ✅ CORRECT
from django.urls import reverse
return redirect(reverse('blog:post_detail', args=[post.id]))
```

---

### ❌ Mistake 2: Forgetting `.as_view()` for CBVs

```python
# ❌ WRONG
path('', views.PostListView, name='post_list'),

# ✅ CORRECT
path('', views.PostListView.as_view(), name='post_list'),
```

---

### ❌ Mistake 3: Wrong Pagination Variable in Template (CBV)

```html
<!-- ❌ WRONG — 'posts' is not the page object in CBV -->
{% include "pagination.html" with page=posts %}

<!-- ✅ CORRECT — ListView always provides page_obj -->
{% include "pagination.html" with page=page_obj %}
```

---

### ❌ Mistake 4: Assuming `unique_for_date` is a Database Constraint

```python
# unique_for_date='publish' is enforced by Django, NOT the database.
# The database does NOT prevent duplicates directly.
# Django validates before saving, but raw SQL inserts can bypass this!
slug = models.SlugField(max_length=250, unique_for_date='publish')
```

---

### ❌ Mistake 5: `slugify()` Drops Non-ASCII Characters

```python
from django.utils.text import slugify

# English — works great
slugify("Hello World")       # "hello-world"

# Arabic — characters are dropped!
slugify("مرحبا")             # "" ← empty string!

# Fix: use allow_unicode=True
slugify("مرحبا", allow_unicode=True)  # "مرحبا"
```

---

### ❌ Mistake 6: Not Using `get_object_or_404()`

```python
# ❌ BAD — crashes with unhandled DoesNotExist exception
post = Post.objects.get(slug=post_slug)

# ✅ GOOD — returns 404 page gracefully
from django.shortcuts import get_object_or_404
post = get_object_or_404(Post, slug=post_slug)
```

---

*Last updated: June 2026 | Django 5 By Example — Chapter 2*
