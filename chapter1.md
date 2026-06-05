# Chapter 1 — Django Admin Tips & ORM QuerySets

> This chapter covers two important areas for Django beginners:
> 1. **Django Admin** — how to configure the built-in admin panel to work efficiently
> 2. **Django ORM** — how to communicate with the database using Python instead of raw SQL

---

# PART 1 — Django Admin Tips

The Django Admin is a powerful built-in tool that gives you a visual interface to manage your database records — no SQL needed. It is automatically available at `/admin/` when you run your project.

> 💡 **How to enable the admin for your model:**
> ```python
> # blog/admin.py
> from django.contrib import admin
> from .models import Post
>
> admin.site.register(Post)
> ```
> Once registered, you can create, read, update, and delete `Post` records from the browser.

---

## 1. `raw_id_fields` — Speed Up Admin for Large Datasets

### The Problem

By default, Django admin renders a **dropdown (select box)** for every `ForeignKey` field.

For example, if a `Post` has an `author` field that links to `User`, the admin will load **every single user** from the database and show them in a dropdown.

This is fine when you have 10 users. But if you have **10,000+ users**, this makes the admin page extremely slow — it runs a huge query just to render the dropdown.

### The Solution — `raw_id_fields`

```python
# blog/admin.py

from django.contrib import admin
from .models import Post

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    raw_id_fields = ['author']
```

Instead of a dropdown, Django will show a **simple input field** where you type the user's ID directly. A small search icon is also shown so you can look up the correct ID.

✅ **Benefits:**
- The admin page loads instantly — no massive query to fetch all related objects
- Works well for thousands or millions of related records
- You can still click the search icon to find the right ID if you don't know it

> 🧠 **Beginner Tip:** Use `raw_id_fields` any time a ForeignKey field points to a model that has a lot of records (users, products, orders, etc.).

---

## 2. `list_display` — Control What Columns Appear in the List View

By default, the admin list page shows only the object's `__str__` representation.

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'author', 'publish', 'status']
```

Now the list page shows a table with all those columns — much more useful!

> 💡 You can also display the result of a method. For example, a short excerpt of the body:
> ```python
> def short_body(self, obj):
>     return obj.body[:50]
>
> list_display = ['title', 'author', 'short_body']
> ```

---

## 3. `list_filter` — Add Sidebar Filters

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'publish', 'status']
    list_filter  = ['status', 'created', 'publish', 'author']
```

This adds a **filter sidebar** on the right side of the admin list page.  
Clicking a filter shows only the matching records — without writing any query manually.

---

## 4. `search_fields` — Add a Search Bar

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display  = ['title', 'author', 'publish', 'status']
    search_fields = ['title', 'body']
```

This adds a **search box** at the top of the list page. Django will search across all the fields you list here.

> ⚠️ `search_fields` uses `LIKE '%query%'` internally, which can be slow on very large tables. For large-scale search, consider tools like Elasticsearch.

---

## 5. `prepopulated_fields` — Auto-fill Slugs

A **slug** is a URL-friendly version of a title (e.g. `"my-first-post"`).

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    prepopulated_fields = {'slug': ('title',)}
```

As you type the title in the admin, the slug field fills in **automatically** using JavaScript. This saves time and avoids typos.

---

## 6. `date_hierarchy` — Navigate by Date

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    date_hierarchy = 'publish'
```

This adds a **date drill-down bar** at the top of the list page.  
You can click year → month → day to filter records by publish date.

---

## 7. `ordering` — Set Default Sort Order in Admin

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    ordering = ['status', 'publish']
```

Records in the admin list will be sorted by `status` first, then by `publish` date.  
Prefix with `-` for descending order: `ordering = ['-publish']`.

---

## 8. Facet Counts (`show_facets`) — See Counts Before Clicking Filters

Facet counts show **how many objects match each filter option** before you click it — like a preview count next to each filter link.

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    show_facets = admin.ShowFacets.ALWAYS
```

| Option | Behaviour |
|---|---|
| `ShowFacets.ALLOW` | User can toggle counts on/off (this is the default) |
| `ShowFacets.ALWAYS` | Counts are always displayed |
| `ShowFacets.NEVER` | Counts are never displayed |

> ⚠️ **Why are counts off by default?**  
> Each facet count requires an extra `COUNT(*)` query to the database. If you have millions of records and many filters, this can significantly slow down the admin page. Only use `ALWAYS` if your dataset is manageable.

---

## 9. Putting It All Together — A Full `PostAdmin` Example

```python
# blog/admin.py

from django.contrib import admin
from .models import Post

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display        = ['title', 'slug', 'author', 'publish', 'status']
    list_filter         = ['status', 'created', 'publish', 'author']
    search_fields       = ['title', 'body']
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields       = ['author']
    date_hierarchy      = 'publish'
    ordering            = ['status', 'publish']
    show_facets         = admin.ShowFacets.ALWAYS
```

This gives you a fully-featured admin page for `Post` with filtering, search, auto-slug, and fast rendering even with large datasets.

---

---

# PART 2 — Django ORM & QuerySets

> The ORM lets you talk to the database using Python — no SQL needed.
> You will learn how to **create**, **read**, **filter**, **update**, and **delete** data.

---

## 10. What is the Django ORM?

The **ORM (Object-Relational Mapper)** is Django's built-in tool that lets you:

- Work with the database using **Python objects** instead of writing SQL
- Automatically generate SQL queries behind the scenes
- Switch between databases (SQLite, PostgreSQL, MySQL) without changing your Python code

```
Python code  →  Django ORM  →  SQL query  →  Database
```

> 🧠 **Think of it this way:** Instead of writing `SELECT * FROM blog_post`, you write `Post.objects.all()`. Django translates your Python into SQL for you.

---

## 11. What is a QuerySet?

A **QuerySet** is the result you get back when you ask Django to fetch records from the database.

- It behaves like a **list of model objects**
- It maps to a `SELECT` SQL statement
- Filters map to `WHERE` or `LIMIT` clauses
- QuerySets are **lazy** — they do NOT hit the database until you actually use the result

```python
# This does NOT run a database query yet — it is just a description
all_posts = Post.objects.all()

# The query runs HERE, when you start looping over the results
for post in all_posts:
    print(post.title)
```

> 💡 Being **lazy** means you can chain many filters together without extra cost — only one query runs at the end.

---

## 12. How to Use the Django Shell

Before learning queries, know that you can **test any query interactively** using the Django shell:

```bash
python manage.py shell
```

Then import your models and try any query:

```python
from blog.models import Post
from django.contrib.auth.models import User

Post.objects.all()
```

> 🧠 The shell is your best friend as a beginner. Always test your queries here first before putting them in views.

---

## 13. Creating Objects

There are three ways to create a new record in the database.

### Method 1 — Two-step: Create in memory, then save

```python
from django.contrib.auth.models import User
from blog.models import Post

# Step 1: Get the author from the database
user = User.objects.get(username='admin')

# Step 2: Create the object in memory — NOT saved to DB yet
post = Post(
    title='Another post',
    slug='another-post',
    body='Post body.',
    author=user
)

# Step 3: Save to the database — this runs an INSERT SQL statement
post.save()
```

> 🧠 This is useful when you need to do something with the object **before** saving it, like running a validation or setting a computed field.

---

### Method 2 — One-step with `create()`

```python
# Creates AND saves the object in a single line
Post.objects.create(
    title='One more post',
    slug='one-more-post',
    body='Post body.',
    author=user
)
```

> 💡 `create()` is a shortcut for Method 1. It does both the creation and the save in one step. Use this when you don't need to do anything with the object before saving.

---

### Method 3 — `get_or_create()`

Retrieves an object if it already exists, or creates it if it does not.  
It returns a **tuple**: `(object, created)` where `created` is `True` if a new record was made, or `False` if it already existed.

```python
# If a user named 'user2' already exists, it returns that user.
# If not, it creates a new user named 'user2'.
user, created = User.objects.get_or_create(username='user2')

if created:
    print("A new user was created!")
else:
    print("This user already existed.")
```

> 💡 This is very useful when importing data or setting up default records — it avoids duplicate entries.

---

### Method 4 — `update_or_create()`

Similar to `get_or_create()`, but **also updates** the object's fields if it already exists.

```python
post, created = Post.objects.update_or_create(
    slug='another-post',           # look up by this field
    defaults={'title': 'Updated Title', 'body': 'New body text.'}  # fields to set
)
```

> 🧠 The `defaults` dictionary contains the fields to update (or set when creating). The lookup fields (like `slug`) are used to find the existing record.

---

## 14. Reading / Retrieving Objects

### `get()` — Fetch exactly one object

```python
user = User.objects.get(username='admin')
post = Post.objects.get(id=1)
```

> ⚠️ **`get()` raises an exception if:**
> - No record is found → raises `Post.DoesNotExist`
> - More than one record matches → raises `Post.MultipleObjectsReturned`
>
> Always use `get()` only when you are sure exactly one record will match (like fetching by primary key or a unique field).

**Safe alternative — use `filter()` and slice:**

```python
# Returns the first matching post, or nothing (no error)
posts = Post.objects.filter(author=user)
if posts.exists():
    post = posts[0]
```

---

### `all()` — Fetch every record

```python
all_posts = Post.objects.all()
```

This is equivalent to `SELECT * FROM blog_post`. It returns all records in the table.

---

### `first()` and `last()` — Fetch one record by position

```python
first_post = Post.objects.all().first()   # first record (lowest pk)
last_post  = Post.objects.all().last()    # last record (highest pk)
```

> 💡 These return `None` if no records exist — they do NOT raise an exception like `get()` does.

---

## 15. Filtering Objects

Use `.filter()` to add conditions — like a SQL `WHERE` clause.

```python
# Get posts with an exact title match
Post.objects.filter(title='Who was Django Reinhardt?')

# Get posts by a specific author
Post.objects.filter(author=user)

# Get posts with a specific status
Post.objects.filter(status='PB')
```

### See the SQL behind any QuerySet

```python
posts = Post.objects.filter(title='Who was Django Reinhardt?')
print(posts.query)
# Output: SELECT ... FROM "blog_post" WHERE "blog_post"."title" = 'Who was Django Reinhardt?'
```

> 💡 Printing `.query` is a great debugging tool. Use it to understand exactly what SQL Django is generating.

---

## 16. Field Lookups (Double Underscore `__`)

Django uses a special `field__lookup_type` syntax for advanced filtering.  
The double underscore `__` separates the field name from the type of comparison.

```python
# Basic syntax
Model.objects.filter(field__lookup=value)

# Example
Post.objects.filter(title__icontains='django')  # titles containing 'django' (case-insensitive)
```

---

### Text Lookups

| Lookup | Example | What it does |
|---|---|---|
| `exact` | `filter(title__exact='Django')` | Exact match (case-sensitive). This is the default. |
| `iexact` | `filter(title__iexact='django')` | Exact match, ignoring uppercase/lowercase |
| `contains` | `filter(title__contains='Django')` | Title must contain 'Django' (case-sensitive) |
| `icontains` | `filter(title__icontains='django')` | Contains match, case-insensitive |
| `startswith` | `filter(title__startswith='Who')` | Title must start with 'Who' |
| `istartswith` | `filter(title__istartswith='who')` | Starts with, case-insensitive |
| `endswith` | `filter(title__endswith='?')` | Title must end with '?' |
| `iendswith` | `filter(title__iendswith='reinhardt?')` | Ends with, case-insensitive |

**Examples:**

```python
# Find all posts with 'django' anywhere in the title (regardless of case)
Post.objects.filter(title__icontains='django')

# Find all posts whose title starts with 'How'
Post.objects.filter(title__startswith='How')

# Find all posts whose title ends with a question mark
Post.objects.filter(title__endswith='?')
```

---

### Numeric / Comparison Lookups

| Lookup | Example | SQL Equivalent |
|---|---|---|
| `in` | `filter(id__in=[1, 3, 5])` | `WHERE id IN (1, 3, 5)` |
| `gt` | `filter(id__gt=3)` | `WHERE id > 3` (greater than) |
| `gte` | `filter(id__gte=3)` | `WHERE id >= 3` (greater than or equal) |
| `lt` | `filter(id__lt=3)` | `WHERE id < 3` (less than) |
| `lte` | `filter(id__lte=3)` | `WHERE id <= 3` (less than or equal) |
| `range` | `filter(id__range=(1, 5))` | `WHERE id BETWEEN 1 AND 5` |

**Examples:**

```python
# Posts with IDs 1, 2, or 3
Post.objects.filter(id__in=[1, 2, 3])

# Posts with ID greater than 5
Post.objects.filter(id__gt=5)

# Posts with ID between 3 and 10 (inclusive)
Post.objects.filter(id__range=(3, 10))
```

---

### Date Lookups

```python
from datetime import date

# Posts published on exactly January 31, 2024
Post.objects.filter(publish__date=date(2024, 1, 31))

# Posts published in 2024 (any month, any day)
Post.objects.filter(publish__year=2024)

# Posts published in January (any year)
Post.objects.filter(publish__month=1)

# Posts published on the 1st of any month
Post.objects.filter(publish__day=1)

# Posts published after January 1, 2024
Post.objects.filter(publish__date__gt=date(2024, 1, 1))

# Posts published between two dates
Post.objects.filter(
    publish__date__gte=date(2024, 1, 1),
    publish__date__lte=date(2024, 6, 30)
)
```

---

### Related Object Lookups (ForeignKey Traversal with `__`)

You can filter across relationships using `__`. Django will automatically do a SQL JOIN.

```python
# Posts where the author's username is 'admin'
Post.objects.filter(author__username='admin')

# Posts where the author's username starts with 'ad'
Post.objects.filter(author__username__startswith='ad')

# Posts published in 2024 by the author 'admin'
Post.objects.filter(publish__year=2024, author__username='admin')

# Posts where the author's email contains 'gmail'
Post.objects.filter(author__email__icontains='gmail')
```

> 🧠 You can chain `__` as many levels deep as needed:
> `Post.objects.filter(author__profile__country='Egypt')`

---

### Null Lookups

```python
# Posts that have no body text (body is NULL)
Post.objects.filter(body__isnull=True)

# Posts that DO have a body (body is not NULL)
Post.objects.filter(body__isnull=False)
```

---

## 17. Chaining Filters

Each `.filter()` returns a **new QuerySet** — you can chain multiple filters together.  
Django combines them all into a single SQL query.

```python
# These two are equivalent:

# Option 1 — Multiple conditions in one filter
Post.objects.filter(publish__year=2024, author__username='admin')

# Option 2 — Chained filters (same result)
Post.objects.filter(publish__year=2024) \
            .filter(author__username='admin')
```

**Example — chain multiple conditions:**

```python
Post.objects.filter(publish__year=2024) \
            .filter(status='PB') \
            .filter(title__icontains='python')
```

> 💡 All three `.filter()` calls above result in a **single SQL query** with all conditions in the `WHERE` clause. Django is smart enough to combine them.

---

## 18. Excluding Objects

Use `.exclude()` to **remove** matching objects from the results.  
Think of it as the opposite of `.filter()` — it removes records that match the condition.

```python
# All posts EXCEPT those with status 'DF' (draft)
Post.objects.exclude(status='DF')

# All 2024 posts whose title does NOT start with 'Why'
Post.objects.filter(publish__year=2024) \
            .exclude(title__startswith='Why')

# All posts NOT written by 'admin'
Post.objects.exclude(author__username='admin')
```

---

## 19. Ordering Objects

```python
# Ascending order (A → Z or oldest → newest)
Post.objects.order_by('title')
Post.objects.order_by('publish')

# Descending order — prefix with '-'  (Z → A or newest → oldest)
Post.objects.order_by('-title')
Post.objects.order_by('-publish')

# Order by multiple fields: first by author, then by title within the same author
Post.objects.order_by('author', 'title')

# Random order — useful for displaying random posts
Post.objects.order_by('?')
```

> 🧠 You can also set a **default ordering** on the model itself, so every query is ordered by default:
> ```python
> class Post(models.Model):
>     class Meta:
>         ordering = ['-publish']  # newest posts first
> ```

---

## 20. Limiting QuerySets (Slicing)

Use Python's list slicing syntax to limit how many results you get back.

```python
# Get the first 5 posts (SQL: LIMIT 5)
Post.objects.all()[:5]

# Skip the first 3 posts, get the next 3 (SQL: OFFSET 3 LIMIT 3)
Post.objects.all()[3:6]

# Get one random post
Post.objects.order_by('?')[0]

# Get the first 3 published posts sorted by newest
Post.objects.filter(status='PB').order_by('-publish')[:3]
```

> ⚠️ **Negative indexing is NOT supported on QuerySets.**  
> `Post.objects.all()[-1]` will raise an error. Use `.last()` instead.

---

## 21. Counting Objects

```python
# How many posts have id less than 3?
Post.objects.filter(id__lt=3).count()
# Returns an integer, e.g. 2
# SQL: SELECT COUNT(*) WHERE id < 3

# How many published posts are there?
Post.objects.filter(status='PB').count()

# How many posts did 'admin' write?
Post.objects.filter(author__username='admin').count()

# Total number of posts in the database
Post.objects.count()
```

> 💡 `.count()` is more efficient than `len(Post.objects.all())` — it runs `SELECT COUNT(*)` instead of loading all records into memory.

---

## 22. Checking If Objects Exist

```python
# Does any post start with 'Why'?
Post.objects.filter(title__startswith='Why').exists()
# Returns: True or False

# Does this specific user have any posts?
Post.objects.filter(author=user).exists()

# Is there a published post with this slug?
Post.objects.filter(slug='my-first-post', status='PB').exists()
```

> 💡 Always use `.exists()` for checking presence. It runs a much faster query than `.count() > 0` or checking `if queryset:`.

---

## 23. Updating Objects

### Update a single object

```python
# Step 1: Fetch the object
post = Post.objects.get(id=1)

# Step 2: Modify the field(s)
post.title = 'New title'
post.body  = 'Updated body text.'

# Step 3: Save — this runs an UPDATE SQL statement
post.save()
```

### Update multiple objects at once with `.update()`

```python
# Set all draft posts to published in a single query
Post.objects.filter(status='DF').update(status='PB')

# Update a specific field for all posts by one author
Post.objects.filter(author__username='admin').update(body='Content updated.')
```

> ⚠️ `.update()` runs a raw SQL `UPDATE` — it does NOT call `.save()` on each object individually, so any custom `save()` logic or signals will NOT be triggered.

---

## 24. Deleting Objects

### Delete a single object

```python
post = Post.objects.get(id=1)
post.delete()
# SQL: DELETE FROM blog_post WHERE id = 1
```

### Delete multiple objects at once

```python
# Delete all draft posts
Post.objects.filter(status='DF').delete()

# Delete all posts from 2022
Post.objects.filter(publish__year=2022).delete()
```

> ⚠️ **Cascade Delete Warning:** Deleting an object also deletes any related objects linked via `ForeignKey` with `on_delete=CASCADE`.  
> For example, if `Comment` has a `ForeignKey` to `Post` with `on_delete=CASCADE`, deleting a post also deletes all its comments.

---

## 25. Complex Lookups with Q Objects

By default, when you pass multiple conditions to `.filter()`, Django joins them with **AND**.

```python
# This returns posts where publish year is 2024 AND author is 'admin'
Post.objects.filter(publish__year=2024, author__username='admin')
```

But what if you need **OR** logic? That's where **Q objects** come in.

### What is a Q Object?

A `Q` object wraps a single lookup condition.  
You then combine them using logical operators:

| Operator | Symbol | Meaning |
|---|---|---|
| AND | `&` | Both conditions must be true |
| OR | `\|` | At least one condition must be true |
| NOT | `~` | The condition must NOT be true |
| XOR | `^` | Exactly one condition must be true |

---

### Example — Filter with OR

```python
from django.db.models import Q

# Posts where title starts with 'who' OR 'why'
Post.objects.filter(
    Q(title__istartswith='who') | Q(title__istartswith='why')
)
```

### Example — Filter with AND

```python
# Posts published in 2024 AND written by 'admin'
Post.objects.filter(
    Q(publish__year=2024) & Q(author__username='admin')
)
```

### Example — Filter with NOT

```python
# All posts that are NOT drafts
Post.objects.filter(~Q(status='DF'))
```

### Example — Mix AND and OR

```python
# Published posts where title starts with 'who' OR 'why'
Post.objects.filter(
    Q(title__istartswith='who') | Q(title__istartswith='why'),
    status='PB'    # AND this condition (added outside Q objects)
)
```

### Example — Store Q objects in variables for readability

```python
starts_who = Q(title__istartswith='who')
starts_why = Q(title__istartswith='why')
is_published = Q(status='PB')

# Combine them
Post.objects.filter((starts_who | starts_why) & is_published)
```

> 📖 Official docs: [Q objects](https://docs.djangoproject.com/en/5.0/topics/db/queries/#complex-lookups-with-q-objects)

---

## 26. When Are QuerySets Evaluated?

QuerySets are **lazy** — they don't hit the database when you write them.  
They only run the SQL query when they are actually **used**.

```python
# No database query yet — just a description of what to fetch
posts = Post.objects.filter(status='PB')

# The query runs HERE when you iterate (loop)
for post in posts:
    print(post.title)
```

### QuerySets are evaluated (database hit) when you:

| Action | Example | Notes |
|---|---|---|
| Iterate (loop) | `for post in Post.objects.all()` | Most common trigger |
| Slice | `Post.objects.all()[:3]` | Evaluates if not already cached |
| Call `list()` | `list(Post.objects.all())` | Forces full evaluation |
| Call `len()` | `len(Post.objects.all())` | Use `.count()` instead for better performance |
| Call `repr()` | `repr(Post.objects.all())` | Useful in the Django shell |
| Use in `bool()` / `if` | `if Post.objects.filter(...):` | Use `.exists()` instead for better performance |
| Pickle or cache | — | Advanced use cases |

> 💡 **Why does this matter?**  
> Because you can chain as many `.filter()` calls as you want without any performance cost — the database is only queried **once**, at the very end.
>
> ```python
> # All three lines below build up the QuerySet without hitting the DB
> posts = Post.objects.filter(publish__year=2024)
> posts = posts.filter(status='PB')
> posts = posts.order_by('-publish')
>
> # DB is hit only here
> for post in posts:
>     print(post.title)
> ```

---

## 27. Custom Model Managers

### What is a Manager?

Every model has a **manager** — it is the object that handles all database queries for that model.  
The default manager is always called `objects`:

```python
Post.objects.all()    # 'objects' is the default manager
Post.objects.filter(status='PB')
```

> 🧠 Think of the manager as the **gateway** between your model and the database.

---

### Why Create a Custom Manager?

If you find yourself repeating the same filter everywhere:

```python
# Typing this over and over in every view is tedious and error-prone
Post.objects.filter(status=Post.Status.PUBLISHED)
```

You can create a **custom manager** to make this reusable and cleaner.

---

### Method 1 — Add a Custom Method to the Existing Manager

This keeps `Post.objects` working as normal, but adds a new method to it.

```python
# models.py

class PostManager(models.Manager):
    def published(self):
        """Return only published posts."""
        return self.filter(status=Post.Status.PUBLISHED)

    def drafts(self):
        """Return only draft posts."""
        return self.filter(status=Post.Status.DRAFT)

class Post(models.Model):
    # ... your fields ...
    objects = PostManager()   # replaces the default manager with our custom one
```

**Usage:**

```python
Post.objects.published()                                          # all published posts
Post.objects.drafts()                                             # all draft posts
Post.objects.published().filter(title__icontains='django')       # chain more filters
Post.objects.published().order_by('-publish')[:5]                # latest 5 published
```

---

### Method 2 — Create a Separate Manager (Recommended for Large Projects)

This creates a brand-new manager that **always filters** published posts automatically.

```python
# models.py

class PublishedManager(models.Manager):
    def get_queryset(self):
        """Override the default QuerySet to only include published posts."""
        return super().get_queryset().filter(
            status=Post.Status.PUBLISHED
        )

class Post(models.Model):
    # ... your fields ...
    objects   = models.Manager()    # keep the default manager — important!
    published = PublishedManager()  # our custom manager
```

**Usage:**

```python
Post.published.all()                              # all published posts
Post.published.filter(title__startswith='Who')   # filter on top
Post.published.order_by('-publish')[:10]         # latest 10 published posts
Post.published.count()                           # count of published posts
```

> ⚠️ **Important Rule:** Always keep `objects = models.Manager()` when you add custom managers.  
> If you remove it, `Post.objects.all()` will stop working and may break parts of Django (like the admin).

---

### Test Custom Managers in the Django Shell

```bash
python manage.py shell
```

```python
from blog.models import Post

# Test the custom manager
Post.published.all()
Post.published.filter(title__startswith='Who')
Post.published.count()
```

---

## 28. Views — Connecting Queries to the Browser

A **view** is a Python function that:
1. Receives an HTTP request from the browser
2. Fetches data from the database using a QuerySet
3. Returns an HTTP response (usually rendered HTML)

### View 1 — List of All Published Posts

```python
# blog/views.py

from django.shortcuts import render
from .models import Post

def post_list(request):
    # Fetch all published posts using our custom manager
    posts = Post.published.all()

    # Render the template and pass 'posts' as context
    return render(
        request,
        'blog/post/list.html',
        {'posts': posts}    # 'posts' is now available in the template
    )
```

**What `render()` does:**

| Argument | What it is |
|---|---|
| `request` | The browser's HTTP request object |
| `'blog/post/list.html'` | Path to the HTML template file |
| `{'posts': posts}` | A dictionary of data passed into the template |

---

### View 2 — Single Post Detail (with manual 404)

```python
# blog/views.py

from django.http import Http404

def post_detail(request, id):
    try:
        # Try to find the published post with this id
        post = Post.published.get(id=id)
    except Post.DoesNotExist:
        # If not found, return a 404 error page
        raise Http404("No Post found.")

    return render(
        request,
        'blog/post/detail.html',
        {'post': post}
    )
```

---

### View 3 — Cleaner Version with `get_object_or_404`

Django provides a shortcut that handles the `try/except` for you automatically:

```python
# blog/views.py

from django.shortcuts import render, get_object_or_404

def post_detail(request, id):
    # Automatically raises Http404 if not found
    post = get_object_or_404(
        Post,
        id=id,
        status=Post.Status.PUBLISHED
    )

    return render(
        request,
        'blog/post/detail.html',
        {'post': post}
    )
```

> ✅ **Best practice:** Always use `get_object_or_404()` instead of a manual `try/except`.  
> It's shorter, cleaner, and follows Django convention.

---

## 29. URL Patterns

URL patterns tell Django **which view to call** when someone visits a URL.

### Step 1 — Create `blog/urls.py`

```python
# blog/urls.py

from django.urls import path
from . import views

# Namespace allows you to reference URLs as 'blog:post_list', 'blog:post_detail'
app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='post_list'),              # /blog/
    path('<int:id>/', views.post_detail, name='post_detail'), # /blog/5/
]
```

**How `path()` works:**

| Part | Example | Meaning |
|---|---|---|
| Route string | `''` | Matches the root URL of this app |
| Route with capture | `'<int:id>/'` | Captures an integer from the URL and names it `id` |
| View | `views.post_list` | The function to call when this URL is visited |
| Name | `name='post_list'` | A nickname to reference this URL in templates and code |

**Path converters:**

| Converter | Example | Matches |
|---|---|---|
| `int` | `<int:id>` | A positive integer like `5` |
| `str` | `<str:name>` | Any string (default) |
| `slug` | `<slug:post>` | Letters, numbers, hyphens, underscores |
| `uuid` | `<uuid:pk>` | A UUID string |

> 💡 For complex patterns (e.g. regex), use `re_path()` instead of `path()`.

---

### Step 2 — Include Blog URLs in the Project's `urls.py`

```python
# project/urls.py

from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),

    # All blog URLs are now accessible under /blog/
    path('blog/', include('blog.urls', namespace='blog')),
]
```

**Result:**

| URL in browser | View called |
|---|---|
| `/blog/` | `post_list` |
| `/blog/5/` | `post_detail` with `id=5` |
| `/blog/12/` | `post_detail` with `id=12` |

### Using Named URLs in Templates

Once you define URL names, **never hardcode paths** in templates. Use the `{% url %}` tag:

```html
<!-- ❌ Bad — hardcoded path breaks if you change the URL -->
<a href="/blog/">Posts</a>

<!-- ✅ Good — uses the named URL -->
<a href="{% url 'blog:post_list' %}">Posts</a>

<!-- ✅ Good — passes the post id to build the URL -->
<a href="{% url 'blog:post_detail' post.id %}">Read More</a>
```

> 💡 This way, if you ever change your URL structure in `urls.py`, you only need to update it in one place — all your templates update automatically.

---

## Quick Reference Summary

### Admin Options

| Option | What it does |
|---|---|
| `list_display` | Columns shown in the admin list view |
| `list_filter` | Sidebar filters in the admin list view |
| `search_fields` | Adds a search bar |
| `prepopulated_fields` | Auto-fills slug from title |
| `raw_id_fields` | Replaces dropdown with ID input (fast for large datasets) |
| `date_hierarchy` | Date drill-down navigation |
| `ordering` | Default sort order in admin |
| `show_facets` | Shows count next to each filter option |

### QuerySet Operations

| Operation | Method | SQL Equivalent |
|---|---|---|
| Get all | `Post.objects.all()` | `SELECT *` |
| Get one | `Post.objects.get(id=1)` | `SELECT ... WHERE id=1 LIMIT 1` |
| First | `Post.objects.first()` | `SELECT ... LIMIT 1` |
| Last | `Post.objects.last()` | `SELECT ... ORDER BY pk DESC LIMIT 1` |
| Filter | `Post.objects.filter(...)` | `WHERE ...` |
| Exclude | `Post.objects.exclude(...)` | `WHERE NOT ...` |
| OR logic | `filter(Q(...) \| Q(...))` | `WHERE ... OR ...` |
| Order | `Post.objects.order_by(...)` | `ORDER BY ...` |
| Limit | `Post.objects.all()[:5]` | `LIMIT 5` |
| Count | `Post.objects.count()` | `SELECT COUNT(*)` |
| Exists | `Post.objects.exists()` | `SELECT 1 ...` |
| Create | `Post.objects.create(...)` | `INSERT INTO ...` |
| Get or Create | `Post.objects.get_or_create(...)` | `SELECT` then `INSERT` if needed |
| Bulk Update | `Post.objects.filter(...).update(...)` | `UPDATE ... WHERE ...` |
| Single Update | `post.save()` | `UPDATE ...` |
| Delete one | `post.delete()` | `DELETE WHERE id=...` |
| Delete many | `Post.objects.filter(...).delete()` | `DELETE WHERE ...` |
