+++
title = "Making your Django tests faster"
date = "2020-05-09"
author = "Rocio Aramberri"
authorTwitter = "rocioar" #do not include @
tags = ["python", "django", "pytest", "testing"]
keywords = ["Fast Django tests", "faster tests", "faster django tests", "django tests"]
description = "Tests need to be fast. If tests are slow, our development process is affected and we end up spending a considerable amount of time waiting for the results. On this blogpost I will go through some of the techniques that I’ve applied to speed up tests on Django applications."
+++

Tests need to be fast. If tests are slow, our development process is affected and we end up spending a considerable amount of time waiting for the results.

I will go through some of the techniques that I’ve applied to speed up tests on Django applications.

## Tips for speeding up test execution

#### 1. Run your tests in parallel

If you are running your tests on multi-core hardware, running your tests in parallel is probably the best optimization you can make if you aren’t doing it yet.

If you are using pytest, install the [pytest-xdist](https://docs.pytest.org/en/3.0.1/xdist.html) plugin:

```bash
$ pip install pytest-xdist
```

And then run your tests with the `-n` parameter:

```bash
$ pytest -n auto
```

The `-n` parameter accepts either a number, indicating the number of cores in which you would like to run the tests, or the value `auto` in which case the plugin will automatically detect the number of cores that are available on your machine and use them all.

If you are using the Django test runner, simply run the tests using the `--parallel` parameter:

```bash
$ ./manage.py test --parallel
```

You can also indicate the specific number of cores in which you would like your tests to run by providing a number:

```bash
$ ./manage.py test --parallel 4
```

#### 2. Use fixtures for modularized setup (Pytest only)

In the typical xUnit based testing frameworks (such as [unittest](https://docs.python.org/3/library/unittest.html)) you create a setup method within your test class in which you set up everything you need for your tests.

Pytest fixtures are a replacement for the conventional setup method. To declare a fixture, you create a function and decorate it with the `@pytest.fixture` decorator.

To use a fixture on a test case you can:
* Make it run automatically for each test case with the `autouse` parameter: `@pytest.fixture(autouse=True)`
* Add it to your test case parameter list. This will trigger the fixture to be executed.
* Add the `usefixtures` mark decorator to your test class: `@pytest.mark.usefixtures("your_fixture")`

One of the greatest advantages of using fixtures is that you can modularize your setup into smaller and less generic functions.

By splitting your setup method into multiple methods, you will be able to select only the ones you need for each test case. This will ensure that you execute exactly what is needed for each test case and nothing else.

#### 3. Use the database only when it is needed

It is common to see lots of database records being created on the setup method of a test class. Usually, not all the records are used by all the test cases, but they still get created each time.

Make sure you **only** create the records that will be used by all the tests on your setup method. If there is a database record that is not being used by all the test cases, either:
* Move it to a fixture (if you are using pytest)
* Move it to the specific test where you need it

#### 4. Search for the slowest tests and look for possible optimizations (Pytest only)

Pytest provides a way to find which are the slowest tests by executing them using the `--durations` parameter.

The following example command will execute all the tests and print a summary at the end with a list of the 5 tests that took the longest to run:

```bash
$ pytest --durations=5
```

#### 5. Squash your migrations

Consider squashing your migrations if you have too many. Squashing refers to reducing the number of migrations you have by merging them into one.

Migrations are run each time you execute your test suite. [Squashing migrations](https://docs.djangoproject.com/en/3.0/topics/migrations/#migration-squashing) would not only speed up your tests but also your development setup.

## Tips for speeding up your tests execution locally

When running tests **locally** while developing a new feature, fixing a bug or refactoring, there are few more things you can do to speed up your tests:

#### 1. Re-use the database among test runs

Every time you run your Django tests, a new test database is created, this takes a considerable amount of time. When you are not making changes to your models or migration files, you can skip this step by telling the runner that you want to preserve the database among test executions.

If you are using Pytest, you can use [pytest-django’s](https://pytest-django.readthedocs.io/en/latest/)  `--reuse-db`  parameter:

```bash
$ pytest --reuse-db
```

If you are using the Django test runner you can accomplish the same with the `--keep-db` parameter:

```bash
$ ./manage.py test --keep-db
```

####  2. Disable Migrations with —no-migrations

When the test runner starts, the database is created and migrations run one by one to reach the current state of your models. If you choose to run your tests without executing the migrations, Django will create the database using the current state of your models.

Skipping migrations should be safe unless you are making changes to your migrations, in which case you would also want to ensure they are working as expected.

Depending on the number of migrations you have, they could take a lot of time. But most of the time, while testing locally, we don’t need them to run.

To disable migrations, include the `--no-migrations` parameter:

```bash
$ pytest --no-migrations
```

Or with the Django test runner:

```bash
$ ./manage.py test --no-migrations
```

#### 3. Run only the tests that failed on the last run (Pytest only)

If you just need to know if the tests that were previously failing are still failing, use pytest’s `--lf` parameter:

```bash
$ pytest --lf
```


## Wrapping up

Ensuring tests run fast is key to having a healthy project. Applying these techniques should help you take your first steps into faster and more efficient tests with Django.

If your application is big enough you might want to consider applying the [Repository Pattern](https://www.cosmicpython.com/book/chapter_02_repository.html) and making your tests completely independent of the database.