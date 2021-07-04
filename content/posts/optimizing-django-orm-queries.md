+++
title = "Optimizing Django ORM Queries"
date = "2020-05-03"
author = "Rocio Aramberri"
keywords = ["Django", "Django ORM Optimization", "Django ORM"]
tags = ["python", "django"]
description = "The Django ORM (Object Relational Mapping) is one of the most powerful features of Django. It enables us to interact with the database using Python code instead of SQL. On this blogpost you will learn how to efficiently make queries using the ORM."
+++

The Django ORM (Object Relational Mapping) is one of the most powerful features of Django. It enables us to interact with the database using Python code instead of SQL.

It has multiple advantages:

* The database engine is abstracted from us, so it is possible to switch to another database system with ease.
* It supports [migrations](https://docs.djangoproject.com/en/3.0/topics/migrations/): we can easily change our tables by updating our models and Django will automatically generate the migration scripts needed to update the database tables.
* It supports [transactions](https://docs.djangoproject.com/en/3.0/topics/db/transactions/): you can make multiple updates to the database within a transaction and, if something fails, roll it back to the way it was when you started.

But it also comes with some disadvantages:
* Since it is an abstraction on top of SQL, it is obscure, and we don’t know exactly which SQL queries will be generated from our Python code.
* Django has no way to guess when we will need to use a related table, so it won’t do JOINs for us when we need them.
* The ORM gives us the wrong sensation that what we are doing is not expensive. We have no easy way to know that accessing an attribute in an object might trigger a query to the database that could have been prevented with a JOIN.

To overcome the disadvantages we need to become more acquainted with it and understand what is happening under the hood.

## Find out what’s happening under the hood

First, we need to understand what is happening in our system, which SQL queries are being run, and what’s costing us the most.

Here are some different mechanisms to inspect SQL queries as they are executed:

#### 1. connection.queries

When `debug=True`, it is possible to access the queries that have been executed by printing `connection.queries`:

```python
>>> from django.db import connection
>>> Post.objects.all()
>>> connection.queries
[
   {
      'sql': 'SELECT "blogposts_post"."id", "blogposts_post"."title", '
             '"blogposts_post"."content", "blogposts_post"."blog_id", '
             '"blogposts_post"."published" FROM "blogposts_post" LIMIT 21',
      'time': '0.000'
   }
]
```

`connection.queries` holds a list of SQL queries in the form of dictionaries containing the SQL code and the time it took to run.

The queries list could get convoluted very easily. To fix that, Django provides a way to clean them up:
```python
>>> from django.db import reset_queries
>>> reset_queries()
```

#### 2. shell_plus --print-sql

The [django-extensions](https://github.com/django-extensions/) project is great and comes with a handful of useful features.

`shell_plus` is one of them. It’s a Django shell with extra additions. If you call it with the `--print-sql` parameter, it will print the SQL queries as they are executed when you run your code.

I will use `shell_plus` throughout this post so that you can see the SQL queries that are being executed as the code is ran.

Here is a quick example of how the output would look like:

```python
$ ./manage.py shell_plus --print-sql
>>> post = Post.objects.get(id=1)
SELECT "blogposts_post"."id",
     	"blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
ORDER BY "blogposts_post"."id" ASC
LIMIT 1
```

#### 3. django-silk

[django-silk](https://github.com/jazzband/django-silk) is a profiling tool. It intercepts requests, records the SQL queries that were performed, and provides a way to visualize them.

You will be able to browse through the requests, see a list of SQL queries that were performed, and look at the details about a specific query including which line of code caused a certain query to run.

#### 4. django-debug-toolbar

[django-debug-toolbar](https://github.com/jazzband/django-debug-toolbar) adds a toolbar on your browser that will show you lots of debugging information while you browse your Django project. Using it, you can see the number of SQL queries that were performed on a request. It is also possible to inspect these queries further, check the SQL code and see in which order they were performed and how much time each one took.

## Optimize your queries

### Introducing an Example Database Model

We will use the following database models as an example for the upcoming sections:

```python
class Blog(models.Model):
   name = models.CharField(max_length=250)
   url = models.URLField()

   def __str__(self):
       return self.name


class Author(models.Model):
   name = models.CharField(max_length=250)
   email = models.EmailField()

   def __str__(self):
       return self.name


class Post(models.Model):
   title = models.CharField(max_length=250)
   content = models.TextField()
   published = models.BooleanField(default=False)

   blog = models.ForeignKey(Blog, on_delete=models.CASCADE)
   authors = models.ManyToManyField(Author, related_name="posts")

   def __str__(self):
       return self.title
```

### Use cached Foreign Key ids

If we just need to access the `id` of a `ForeignKey` field, we can use the cached id that Django already has cached for us via `<field_name>_id`.

Let see it through an example:

```python
>>> Post.objects.first().blog.id
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
ORDER BY "blogposts_post"."id" ASC
LIMIT 1

Execution time: 0.001668s [Database: default]
SELECT "blogposts_blog"."id",
      "blogposts_blog"."name",
      "blogposts_blog"."url"
 FROM "blogposts_blog"
WHERE "blogposts_blog"."id" = 1
LIMIT 21

Execution time: 0.000197s [Database: default]
```

Accessing the blog's `id` through the nested object `blog` generated a new SQL query to obtain the entire blog object. But since we won’t need to access any other attribute from the blog object, we could completely avoid the above query from being executed by doing:

```python
>>> Post.objects.first().blog_id
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
ORDER BY "blogposts_post"."id" ASC
LIMIT 1

Execution time: 0.000165s [Database: default]
```

### Let Django know what you will need in advance

#### Use select_related for Foreign Keys

Django has no way of anticipating when we will need to access a `ForeignKey` relationship from within the model we are querying. The [select_related](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#select-related) utility allows us to tell Django exactly which related models we want, so that it can perform JOINs.

In our example, we have a `Post` model. A `Post` belongs to a specific `Blog`. This relationship is represented on the database through a `ForeignKey` from the `Post` to the `Blog`.

To access a specific `Post` object we could do:

```python
>>> post = Post.objects.get(id=1)
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
ORDER BY "blogposts_post"."id" ASC
LIMIT 1
```

If we wanted to access the `Blog` object from within the `Post`, we could do:

```python
>>> post.blog
SELECT "blogposts_blog"."id",
      "blogposts_blog"."name",
      "blogposts_blog"."url"
 FROM "blogposts_blog"
WHERE "blogposts_blog"."id" = 1
LIMIT 21

Execution time: 0.000602s [Database: default]
<Blog: Rocio's Blog>
```

However, this statement generated a new query to grab the information from the blog. We want to avoid that. This is when `select_related` comes to our rescue. To use it we can update our original query to be:

```python
>>> post = Post.objects.select_related("blog").get(id=1)
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published",
      "blogposts_blog"."id",
      "blogposts_blog"."name",
      "blogposts_blog"."url"
 FROM "blogposts_post"
INNER JOIN "blogposts_blog"
   ON ("blogposts_post"."blog_id" = "blogposts_blog"."id")
WHERE "blogposts_post"."id" = 1
LIMIT 21

Execution time: 0.000150s [Database: default]
```

Notice now how Django used a `JOIN` on the SQL query above to also grab the attributes from the blog table for us. Now, when accessing the `Blog` object from within the `Post`, it will not require an extra query since it will already be cached:

```python
>>> post.blog
<Blog: Rocio's Blog>
```

`select_related` also works for querysets. We could pre-select the blog object for an entire queryset. If there were 50 Posts and we didn’t use `select_related` to pre-select the blog object, it would take Django 50 queries to run the following code. With `select_related` it just takes one:

```python
>>> posts = Post.objects.select_related("blog").all()
>>> for post in posts:
       post.blog

SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published",
      "blogposts_blog"."id",
      "blogposts_blog"."name",
      "blogposts_blog"."url"
 FROM "blogposts_post"
INNER JOIN "blogposts_blog"
   ON ("blogposts_post"."blog_id" = "blogposts_blog"."id")

Execution time: 0.000224s [Database: default]
```

#### Use prefetch_related for ManyToMany fields

[prefetch_related](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#prefetch-related) is similar to `select_related`, but it is used for pre-selecting `ManyToMany` fields. `prefetch_related` works differently, let’s see it through an example.

Let’s say we wanted to grab all the Posts and then print the Authors for each of the Posts. We could do the following:

```python
>>> for post in Post.objects.all():
       post.authors.all()

SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"

Execution time: 0.000158s [Database: default]
<QuerySet []>
SELECT "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_author"."id" = "blogposts_post_authors"."author_id")
WHERE "blogposts_post_authors"."post_id" = 1
LIMIT 21

Execution time: 0.000101s [Database: default]
<QuerySet []>
SELECT "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_author"."id" = "blogposts_post_authors"."author_id")
WHERE "blogposts_post_authors"."post_id" = 2
LIMIT 21
Execution time: 0.001043s [Database: default]

Execution time: 0.000101s [Database: default]
<QuerySet []>
SELECT "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_author"."id" = "blogposts_post_authors"."author_id")
WHERE "blogposts_post_authors"."post_id" = 3
LIMIT 21
Execution time: 0.001043s [Database: default]
```

Notice that the code above generated 4 queries, one to grab the posts, and then one query for each of the posts to grab the authors (there were 3 posts in total).

This is the famous N + 1 problem. Given N posts, N + 1 queries will be performed. In this scenario we have 3 posts, which translate to 4 queries. It isn’t that much, but this could very easily escalate as we create new Posts. With 50 Posts, this code would generate 51 queries.

To avoid that, we could pre-select the Authors by using `prefetch_related`:

```python
>>> for post in Post.objects.prefetch_related("authors").all():
       post.authors.all()

SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"

Execution time: 0.000158s [Database: default]
SELECT ("blogposts_post_authors"."post_id") AS "_prefetch_related_val_post_id",
      "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_author"."id" = "blogposts_post_authors"."author_id")
WHERE "blogposts_post_authors"."post_id" IN (1, 2, 3)

Execution time: 0.001043s [Database: default]
```

With our updated code only 2 queries were performed. When `prefetch_related` is used, Django first grabs all the posts and then runs another SQL query that retrieves all the authors for all the posts.

#### Customizing Prefetch

In some scenarios `prefetch_related` basic syntax is not enough to prevent Django from doing extra queries. To further control the prefetch you can use the `Prefetch` object.

In our example database, there is a `Post` model and an `Author` model. The `Post` model is related to the `Author` model through a `ManyToMany` field. Let’s say we wanted to go author by author and grab all the posts that were published by that author:

```python
>>> authors = Author.objects.all()
>>> for author in authors:
       print(author.posts.filter(published=True))

SELECT "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"

Execution time: 0.000251s [Database: default]
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_post"."id" = "blogposts_post_authors"."post_id")
WHERE ("blogposts_post_authors"."author_id" = 1 AND "blogposts_post"."published" = 1)
LIMIT 21

Execution time: 0.000178s [Database: default]
<QuerySet [<Post: Optimizing Django ORM Queries>, <Post: Placeholder Post>, <Post: Placeholder Post 2>, <Post: Placeholder Post 3>, <Post: Placeholder Post 4>, <Post: Placeholder Post 6>, <Post: Placeholder Post 7>, <Post: Placeholder Post 8>, <Post: Placeholder Post 9>]>
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_post"."id" = "blogposts_post_authors"."post_id")
WHERE ("blogposts_post_authors"."author_id" = 2 AND "blogposts_post"."published" = 1)
LIMIT 21

Execution time: 0.000081s [Database: default]
<QuerySet [<Post: Optimizing Django ORM Queries>]>
```

As you can see, the above code generated 3 queries, 1 to grab the author, and then 2 queries to grab the posts for each of the authors.

What if we used `prefetch_related`? It seems to be the right thing to do:

```python
>>> authors = Author.objects.prefetch_related("posts").all()
>>> for author in authors:
       print(author.posts.filter(published=True))

SELECT "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"

Execution time: 0.000097s [Database: default]
SELECT ("blogposts_post_authors"."author_id") AS "_prefetch_related_val_author_id",
      "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_post"."id" = "blogposts_post_authors"."post_id")
WHERE "blogposts_post_authors"."author_id" IN (1, 2)

Execution time: 0.000190s [Database: default]
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_post"."id" = "blogposts_post_authors"."post_id")
WHERE ("blogposts_post_authors"."author_id" = 1 AND "blogposts_post"."published" = 1)
LIMIT 21

Execution time: 0.000074s [Database: default]
<QuerySet [<Post: Optimizing Django ORM Queries>, <Post: Placeholder Post>,
<Post: Placeholder Post 2>, <Post: Placeholder Post 3>,
<Post: Placeholder Post 4>, <Post: Placeholder Post 6>,
<Post: Placeholder Post 7>, <Post: Placeholder Post 8>,
<Post: Placeholder Post 9>]>
SELECT "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_post"."id" = "blogposts_post_authors"."post_id")
WHERE ("blogposts_post_authors"."author_id" = 2 AND "blogposts_post"."published" = 1)
LIMIT 21

Execution time: 0.000070s [Database: default]
<QuerySet [<Post: Optimizing Django ORM Queries>]>
```

What did just happen? We used `prefetch_related` to reduce the number of queries, and we actually increased it by 1.

This is happening because we are filtering the posts with `published=True`. Django can’t use our cached posts since they were not filtered when they were queried. To avoid this from happening, we can customize the queryset using the `Prefetch` object:

```python
>>> authors = Author.objects.prefetch_related(
       Prefetch(
          "posts",
          queryset=Post.objects.filter(published=True),
          to_attr="published_posts",
       )
    )
>>> for author in authors:
      print(author.published_posts)

SELECT "blogposts_author"."id",
      "blogposts_author"."name",
      "blogposts_author"."email"
 FROM "blogposts_author"

Execution time: 0.000129s [Database: default]
SELECT ("blogposts_post_authors"."author_id") AS "_prefetch_related_val_author_id",
      "blogposts_post"."id",
      "blogposts_post"."title",
      "blogposts_post"."content",
      "blogposts_post"."blog_id",
      "blogposts_post"."published"
 FROM "blogposts_post"
INNER JOIN "blogposts_post_authors"
   ON ("blogposts_post"."id" = "blogposts_post_authors"."post_id")
WHERE ("blogposts_post"."published" = 1 AND "blogposts_post_authors"."author_id" IN (1, 2))

Execution time: 0.000089s [Database: default]
[<Post: Optimizing Django ORM Queries>, <Post: Placeholder Post>,
<Post: Placeholder Post 2>, <Post: Placeholder Post 3>,
<Post: Placeholder Post 4>, <Post: Placeholder Post 6>,
<Post: Placeholder Post 7>, <Post: Placeholder Post 8>,
<Post: Placeholder Post 9>]
[<Post: Optimizing Django ORM Queries>]
```

We used the `Prefetch` object to tell Django to:

* Use a specific queryset to retrieve the posts - through the `queryset` parameter.
* Store the filtered posts in a new attribute (`published_posts`) - through the `to_attr` parameter.

When `author.published_posts` is executed, no queries will be run since everything will already be cached. No matter the number of authors on our system, the operation will always take 2 SQL queries.

## Wrapping up

While working with the Django ORM, it is extremely important that we think about what’s happening under the hood.

The concepts that you learned on this blog post will help you write more optimized queries and be in the lookout for possible optimizations while reviewing code. Beware though, that you should always measure the time a query is taking before and after the optimization to ensure that the optimization worked. Sometimes less queries doesn't necessairly mean less time, JOINs could be expensive too.
