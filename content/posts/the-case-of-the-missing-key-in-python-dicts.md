+++
title = "The case of the missing key in Python dicts"
date = "2020-06-14T14:44:49-03:00"
author = "Rocio Aramberri"
keywords = ["python dictionaries", "python dict", "python setdefault", "python defaultdict"]
tags = ["python"]
description = "Let's say you have a dictionary and you want to append values to a key that doesn't exist yet. Appending the value directly without initializing it would raise a `KeyError` exception. There are a few different solutions for this problem. On this blogpost I will go through all of them."
+++

Let's say you have a dictionary and you want to append values to a key that doesn't exist yet. Consider this dictionary:

```python
>>> books_by_author = {
    "Luciano Ramalho": [{"title": "Fluent Python"}]
}
```

Appending the value directly without initializing it would raise a `KeyError` exception:

```python
>>> books_by_author["David Beazley"].append({"title": "Python Cookbook"})
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'David Beazley'
```

There are a few different solutions that can be applied to this problem:

* Look before you leap (LBYL)
* Easier to ask for forgiveness than permission (EAFP)
* Python tools: `setdefault` or `defaultdict`

## Look before you leap

One solution could be to use `in` to find out if the key exists. If it doesn't then initialize it, and finally append the value to it.

On the following example we are grouping all books by author. Notice how we first check if the author key is already in the dictionary, and if it is not we assign it to an empty list so that we can append the book to the list later:
```python
def get_books_grouped_by_author(books):
    books_by_author = {}

    for book in books:
        for author in book["authors"]:
            if author not in books_by_author:
                books_by_author[author] = []

            books_by_author[author].append(book)

    return books_by_author
```

## Easier to ask for permission than forgiveness

Another option would be to use a `try/except` block, catch the `KeyError` exception, and if the error is raised initialize the key with just one book:
```python
def get_books_grouped_by_author(books):
    books_by_author = {}

    for book in books:
        for author in book["authors"]:
            try:
                books_by_author[author].append(book)
            except KeyError:
                books_by_author[author] = [book]

    return books_by_author
```

Using a `try/except` block will be the slowest approach for our specific example since the `KeyError` exception will be raised frequently (once per author), and the most expensive part of a `try/except` block is dealing with the exception itself. More on that on: [How fast are exceptions](https://docs.python.org/3/faq/design.html?#how-fast-are-exceptions).

## Using setdefault

`setdefault`  allows you to specify a default value for a key. If the key doesn't exist then that default value will be inserted and then returned. If the key already exists then the corresponding value will be returned.

As you can see using `setdefault` simplifies our code a lot, we can now handle the missing key with just one line:
```python
def get_books_grouped_by_author(books):
    books_by_author = {}

    for book in books:
        for author in book["authors"]:
            books_by_author.setdefault(author, [])
            books_by_author[author].append(book)

    return books_by_author
```

The one problem with this solution, though, is that the default value (in this case an empty list) will be created on every iteration, even if there is already a value for the given key. The overhead of creating a list on every iteration makes this solution slower for our use case.

## Using defaultdict

`defaultdict` is a subclass of `dict` which accepts a `default_factory` as a parameter and uses it whenever there is a missing key.

To use it, we can just update our example to initialize `books_by_author` as a `defaultdict` with `list` as the `default_factory`:
```python
from collections import defaultdict


def get_books_grouped_by_author(books):
    books_by_author = defaultdict(list)

    for book in books:
        for author in book["authors"]:
            books_by_author[author].append(book)

    return dict(books_by_author)
```

When using a `defaultdict` it's important to remember to cast the value back to a normal `dict` before returning it. This is to prevent the unexpected behavior of a key being added to the dictionary if it is accessed through the square bracket notation later.

Notice how accessing the `inexistent key` using `get` does not return any value, but using square brackets causes the key to be initialized with an empty list:
```python
>>> books_by_author = get_books_by_author(books)
>>> books_by_author.get('inexistent key')
>>> books_by_author['inexistent key']
[]
```

For this use case, `defaultdict` is the fastest option, considering the `books` list is big enough. If the books list is too small, then the overhead of creating a `defaultdict` could make it perform even slower than other solutions.

## Wrapping up

There is no right or wrong on any of these solutions, you must analyze your use case to decide which one fits better.

* If you are not expecting to have many `KeyError` exceptions since the dictionary might already be pre-populated, then using a `try/except` block might be the best option.
* If you just want to insert a default value for a single key outside of an iteration, then `setdefault` might be just right.
* If you want to use the same `default_factory` to initialize multiple keys, then `defaultdict` will probably be the fastest option.

If you are unsure about which of these approaches will be faster, you can test how each of them perform using [timeit](https://docs.python.org/3.8/library/timeit.html).