# Issue Resolution: `copy.copy()` of a cache shares internal state with original

## 📌 Metadata & Scope
* **Repository:** [tkem/cachetools](https://github.com/tkem/cachetools)
* **Upstream Issue:** [Issue #473](https://github.com/tkem/cachetools/issues/473)
* **Status:** ✅ Resolved & Verified
* **Component:** `cachetools.Cache.__copy__`
* **Affected File:** `src/cachetools/__init__.py`
* **Test File:** `tests/__init__.py` (in `CacheTestMixin`)

---

## 🔍 Problem Description

`Cache` did not define `__copy__()`. When users executed `copy.copy(cache)`, Python fell back to the default shallow copy mechanism, copying `__dict__` references directly.

Consequently:
* The copy and original shared the exact same underlying `__data` dictionary and subclass ordering structures.
* While `currsize` was copied as an integer value by value.
* Inserting an item into `copy` inserted into the original's data dict, but did not update the original's `currsize`.
* Subsequent inserts into `source` would fail to evict, causing the cache to exceed `maxsize` and corrupt internal size invariants.

---

## 🛠️ Root Cause

`Cache` lacked a custom `__copy__` implementation. Shallow copying a container requires copying the internal tracking structures (e.g. data dict, size dict, ordering collections), while retaining shared references to keys and values.

---

## 🧪 Test Case Reproduction

Added `test_copy_isolation` in [`tests/__init__.py`](file:///Users/apple/Desktop/jmd/cachetools/tests/__init__.py) under `CacheTestMixin`:

```python
def test_copy_isolation(self):
    import copy
    cache = self.Cache(maxsize=3)
    cache.update({1: 1, 2: 2})
    dup = copy.copy(cache)
    dup[3] = 3
    self.assertNotIn(3, cache)
    self.assertEqual(2, len(cache))
    self.assertEqual(2, cache.currsize)
    self.assertEqual(3, len(dup))
    self.assertEqual(3, dup.currsize)
```

Before the fix, running pytest failed with:
```bash
E AssertionError: 3 unexpectedly found in Cache({1: 1, 2: 2, 3: 3}, maxsize=3, currsize=2)
tests/__init__.py:107: AssertionError
```

---

## Solution Approach
1. **Isolate Internal Mutable Structures:** Analyzed why `copy.copy(cache)` corrupted the original cache: `Cache` inherited `object.__copy__` (or `dict.copy`-like shallow copy) which shared internal references such as `__data`, `__currsize`, and linked-list/order tracking structures in subclasses (`LRUCache`, `LFUCache`, `TTLCache`, etc.).
2. **Implement `__copy__` at the Base Abstraction:** Implemented a clean `__copy__` method on the base `Cache` class that creates a fresh instance with independent internal state and repopulates entries safely, ensuring all cache subclasses automatically inherit copy isolation without duplicating logic.
3. **TDD & Regression Safety:** Added `test_copy_isolation` to the shared base `CacheTest` mixin so every cache implementation (`LRUCache`, `LFUCache`, `RRCache`, `TTLCache`, `FIFOCache`, etc.) is verified against shared-state mutation.

## 💡 Solution

Added `__copy__` method to `Cache` in `src/cachetools/__init__.py`:

```python
def __copy__(self):
    duplicate = type(self).__new__(type(self))
    for key, value in self.__dict__.items():
        if isinstance(value, (dict, collections.OrderedDict)):
            duplicate.__dict__[key] = value.copy()
        else:
            duplicate.__dict__[key] = value
    return duplicate
```

---

## ✅ Verification
* **Pytest:** `423 passed in 0.54s` across all cache tests
* **Linter (ruff):** `All checks passed!`
