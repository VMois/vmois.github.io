---
layout: post
title: Make underlying cache methods available for timed LRU cache in Python 3
tags: [python, cache]
comments: false
---

We know that cache can greatly speed up a load of frequently used data 
(for example, S3 objects). 
Python 3 has a built-in implementation of simple unbound and LRU caches in `functools` module called `cache` and `lru_cache` respectively ([functools documentation](https://docs.python.org/3.9/library/functools.html)). Both `cache` and `lru_cache` expose some useful methods like `cache_clear` or `cache_info`. Clearing cache is especially important between your code tests because you can get strange results ([a good article that describes this problem](https://rishabhsrao.medium.com/testing-lru-cache-functions-in-python-with-pytest-33dd5757d11c)).

If you expect your data to change during the program execution, 
it makes sense to implement a timed cache. Unfortunately, Python 3 doesn't provide any built-in functionality for timed cache so we need to implement it by ourselves.

Searching on the Internet, you can find a lot of different implementations but all of them are pretty much the same. Let's take an example from [great article about cache](https://realpython.com/lru-cache-python/) by the Real Python website.

Implementation of the timed cache from the article:

```python
from functools import lru_cache, wraps
from datetime import datetime, timedelta

def timed_lru_cache(seconds: int, maxsize: int = 128):
    def wrapper_cache(func):
        func = lru_cache(maxsize=maxsize)(func)
        func.lifetime = timedelta(seconds=seconds)
        func.expiration = datetime.utcnow() + func.lifetime

        @wraps(func)
        def wrapped_func(*args, **kwargs):
            if datetime.utcnow() >= func.expiration:
                func.cache_clear()
                func.expiration = datetime.utcnow() + func.lifetime

            return func(*args, **kwargs)

        return wrapped_func

    return wrapper_cache
```

{: .box-note}
**Note:** a cache is cleared only when the `timed_lru_cache` function is called and the condition is checked. Do not expect cache to be cleared right on the expiration time.

And usage:

```python
@timed_lru_cache()
def load_key_from_s3(key: str):
    # ...

```

{: .box-warning}
**Warning:** brackets are important. `@timed_lru_cache` will not work, use `@timed_lru_cache()`

But there is an issue, the `wraps` method from `functools` is only preserving the original function's name and docstring
but not original methods ([documentation on wraps](https://docs.python.org/3.9/library/functools.html#functools.wraps)). Calling, for example, `load_key_from_s3.cache_clear()` will fail. What if we want to expose missing methods for test and statistics purposes? The simplest fix to the above implementation is the following:

```python
def timed_lru_cache(seconds: int, maxsize: int = 128):
    def wrapper_cache(func):
        func = lru_cache(maxsize=maxsize)(func)
        func.lifetime = timedelta(seconds=seconds)
        func.expiration = datetime.utcnow() + func.lifetime

        @wraps(func)
        def wrapped_func(*args, **kwargs):
            if datetime.utcnow() >= func.expiration:
                func.cache_clear()
                func.expiration = datetime.utcnow() + func.lifetime

            return func(*args, **kwargs)
        
        # add missing methods to wrapped function
        wrapped_func.cache_clear = func.cache_clear
        wrapped_func.cache_info = func.cache_info

        return wrapped_func

    return wrapper_cache
```

Now both `cache_clear` and `cache_info` methods are exposed for `load_key_from_s3` function. Thank you for reading.
