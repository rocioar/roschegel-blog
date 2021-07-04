+++
title = "Simplified Django Tests With Pytest and Pytest FactoryBoy"
date = "2020-05-17"
author = "Rocio Aramberri"
authorTwitter = "rocioar"
tags = ["python", "django", "pytest", "testing"]
keywords = ["django tests", "pytest-factoryboy", "Pytest and Django"]
description = "Lately, I've been bothered by the amount of boilerplate on my Django application's test code. So, I decided to do some research and look for alternatives. On this blogpost I will take about pytest, factoryboy and how they can be combined."
+++

I'm always trying to find ways to make tests easier to read and extend. I hate working through a really hard feature and having to spend a big amount of time writing tests.

Lately, I've been bothered by the amount of boilerplate on my test code. So, I decided to do some research and look for alternatives.

When testing Django applications, I use a combination of [Pytest](https://docs.pytest.org/en/latest/) fixtures and  [FactoryBoy](https://factoryboy.readthedocs.io/) to write tests that need database records.

* **Pytest Fixtures** allow you to abstract the initialization of your test cases into functions (fixtures) and help you re-use code. Pytest uses dependency injection to detect when a fixture is needed on a test by matching the name provided on the parameter with the name of the fixture. Learn more about Pytest Fixtures on the [Pytest documentation](https://docs.pytest.org/en/latest/fixture.html).
* **FactoryBoy** allows you to create factories for your models with pre-defined values, which makes it easy to create model objects on the go. It handles relationships by allowing you to define subfactories and integrates with the [faker](https://faker.readthedocs.io/en/master/) library to provide the ability to use randomized values. It's a good alternative to Django [fixtures](https://docs.djangoproject.com/en/3.0/howto/initial-data/#providing-data-with-fixtures).


## The Problem

Let's look through an example to understand what we are trying to fix.

We will write a test case for the following function which returns a list of `Post`'s titles:
```python
def get_posts_titles(include_unpublished=False):
    queryset = Post.objects.all()
    if not include_unpublished:
        queryset = queryset.filter(status="published")
    return  list(queryset.values_list("title", flat=True))
```

First, in `factories.py` we define the factory that will later be used to create the model objects on the test. In this case, it's just a simple database with a `Post` model. So we just need one factory:
```python
import factory

class PostFactory(factory.django.DjangoModelFactory):
    title = "Combining Pytest and FactoryBoy for simplified tests in Django"
    status = "draft"

    class Meta:
        model = "blog.Post"
```

Then in the `test_posts.py` file, we import the factory and create 2 fixtures that use it, `post_draft` and `post_published`:
```python
import pytest

from factories import PostFactory


@pytest.fixture
def post_draft():
    return PostFactory()


@pytest.fixture
def post_published():
    return PostFactory(status="published")


@pytest.mark.django_db
def test_get_posts_titles(post_published, post_draft):
    assert get_posts_titles() == [post_published.title]
    assert get_posts_titles(include_unpublished=True) == [post_published.title, post_draft.title]
```

Notice the amount of boilerplate on the code, we had to:
* create a `PostFactory` on the `factories.py` file;
* create a fixture for a `draft` post;
* create a fixture for a `published` post.

## Avoiding boilerplate code with pytest-factoryboy

Let's see how we could re-write the code from the previous section using [pytest-factoryboy](https://pytest-factoryboy.readthedocs.io/en/latest/).

We'll still have the factories defined in the `factories.py` file:
```python
import factory


class PostFactory(factory.django.DjangoModelFactory):
    title = "Combining Pytest and FactoryBoy for simplified tests in Django"
    status = "draft"

    class Meta:
        model = "blog.Post"
```

But now, we register that factory in [conftests.py](https://docs.pytest.org/en/2.7.3/plugins.html?highlight=re) so that it is available as a fixture for all tests:
```python
from pytest_factoryboy import register

from factories import PostFactory


register(PostFactory)
register(PostFactory, "draft_post", title="Draft post", status="draft")
```

The fixture will take the name of the Factory (`post` by default) but you can optionally provide it a name. Two fixtures will be created: `post` and `draft_post`.

Then, in the tests, we can start using the registered fixtures:
```python
import pytest

from blog.utils import get_posts_titles


@pytest.mark.django_db
def test_get_posts_titles(post, draft_post):
    assert get_posts_titles() == [post.title]
    assert get_posts_titles(include_unpublished=True) == [post.title, draft_post.title]
```

Since `pytest-factoryboy` took care of registering the factories as fixtures for us, we won't need to:
* manually create the fixtures;
* manually import the `PostFactory` on the test file.

## More about pytest-factoryboy

Being able to register a factory as a fixture is just one of `pytest-factoryboy`'s features. Let's see some more.

#### Set custom attribute values for you model fixtures

There will be some test cases in which you need to customize an attribute of your model fixture. You can easily do so using `pytest.mark.parametrize`:

```python
import pytest

from blog.utils import get_posts_titles


@pytest.mark.django_db
@pytest.mark.parametrize("post__title", ["Custom title"])
@pytest.mark.parametrize("post_draft__title", ["Custom title - Draft"])
def test_get_posts_titles(post, draft_post):
    assert get_posts_titles() == ["Custom title"]
    assert get_posts_titles(include_unpublished=True) == ["Custom title", "Custom title - Draft"]
```

#### Override a fixture's value

If you are modifying the same attribute on multiple test functions within a module, you can opt to override the attribute on the fixture for the entire module. The following code will set `post.status` to `"draft"` for the entire module:

```python
import pytest

from blog.utils import get_posts_titles


@pytest.fixture
def post__status():
    """Override blog's status to draft."""
    return "draft"


@pytest.mark.django_db
def test_get_posts_titles(post):
    assert get_posts_titles() == []
    assert get_posts_titles(include_unpublished=True) == [post.title]
```

#### Factories are automatically registered as fixtures

Sometimes you will just want to interact with the factory directly, without using the model fixture (e.g. `post`). `pytest-factoryboy` automatically registers the factory as a fixture too. Notice that the fixture name is the snake_case version of the factory name:

```python
import pytest

from blog.utils import get_posts_titles


@pytest.mark.django_db
def test_get_posts_titles(post_factory):
    post = post_factory(status="draft")
    assert get_posts_titles() == []
    assert get_posts_titles(include_unpublished=True) == [post.title]
```

#### Assign other fixtures as values with parametrize

When your models have `ForeignKey` relationships you might want to assign another fixture as the value. You can do so using `parametrize` and `LazyFixture`.

Given a `Post` with a `blog` attribute as a `ForeignKey`, we could have the following factories:
```python
import factory


class BlogFactory(factory.django.DjangoModelFactory):
    name = "roschegel"

    class Meta:
        model = "blog.Blog"


class PostFactory(factory.django.DjangoModelFactory):
    title = "Combining Pytest and FactoryBoy for simplified tests in Django"
    status = "draft"
    blog = factory.SubFactory(BlogFactory)

    class Meta:
        model = "blog.Post"
```

In our test case we could then override the blog attribute using `LazyFixture`:
```python
import pytest

from pytest_factoryboy import LazyFixture


@pytest.fixture
def blog_2(blog_factory):
    return blog_factory(name="some_other_blog")


@pytest.mark.django_db
@pytest.mark.parametrize("post__blog", [LazyFixture("blog_2")])
def test_lazy_fixture(post, blog_2):
    assert post.blog.id == blog_2.id
```

## Wrapping up

Next time you find yourself writing factories and fixtures that just return instances of your models, consider using `pytest-factoryboy`. It will save you a considerable amount of boilerplate code. And that means more time to work on meaningfull stuff, which is always welcome.
