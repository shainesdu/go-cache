---
title: doc_by_code
---
# Introduction

This document will walk you through the design and implementation of a caching system in the <SwmPath>[cache.go](/cache.go)</SwmPath> file. The caching system is designed to efficiently store and retrieve items with expiration capabilities. We will cover:

1. The architecture of the caching system.
2. The code flow for adding, retrieving, and managing cache items.
3. The subsystem responsible for cache maintenance.
4. Conclusion on the design choices and their implications.

# Architecture

<SwmSnippet path="/cache.go" line="13">

---

The caching system is built around the <SwmToken path="/cache.go" pos="35:2:2" line-data="type Cache struct {">`Cache`</SwmToken> and <SwmToken path="/cache.go" pos="13:2:2" line-data="type Item struct {">`Item`</SwmToken> types. The <SwmToken path="/cache.go" pos="35:2:2" line-data="type Cache struct {">`Cache`</SwmToken> type manages a collection of items, each represented by the <SwmToken path="/cache.go" pos="13:2:2" line-data="type Item struct {">`Item`</SwmToken> type, which includes an object and an expiration timestamp.

```
type Item struct {
	Object     interface{}
	Expiration int64
}

// Returns true if the item has expired.
func (item Item) Expired() bool {
	if item.Expiration == 0 {
		return false
	}
	return time.Now().UnixNano() > item.Expiration
}
```

---

</SwmSnippet>

<SwmSnippet path="/cache.go" line="35">

---

The <SwmToken path="/cache.go" pos="35:2:2" line-data="type Cache struct {">`Cache`</SwmToken> type uses a map to store items and a mutex for concurrent access control. This design ensures thread-safe operations on the cache.

```
type Cache struct {
	*cache
	// If this is confusing, see the comment at the bottom of New()
}

type cache struct {
	defaultExpiration time.Duration
	items             map[string]Item
	mu                sync.RWMutex
	onEvicted         func(string, interface{})
	janitor           *janitor
}
```

---

</SwmSnippet>

# Code flow

The code flow for managing cache items involves several key operations:

<SwmSnippet path="/cache.go" line="48">

---

- **Setting items**: The <SwmToken path="/cache.go" pos="51:9:9" line-data="func (c *cache) Set(k string, x interface{}, d time.Duration) {">`Set`</SwmToken> method adds or replaces items in the cache, using a specified expiration duration. If no duration is provided, the default expiration is used.

```
// Add an item to the cache, replacing any existing item. If the duration is 0
// (DefaultExpiration), the cache's default expiration time is used. If it is -1
// (NoExpiration), the item never expires.
func (c *cache) Set(k string, x interface{}, d time.Duration) {
	// "Inlining" of set
	var e int64
	if d == DefaultExpiration {
		d = c.defaultExpiration
	}
	if d > 0 {
		e = time.Now().Add(d).UnixNano()
	}
	c.mu.Lock()
	c.items[k] = Item{
		Object:     x,
		Expiration: e,
	}
	// TODO: Calls to mu.Unlock are currently not deferred because defer
	// adds ~200 ns (as of go1.)
	c.mu.Unlock()
}
```

---

</SwmSnippet>

<SwmSnippet path="/cache.go" line="118">

---

- **Retrieving items**: The <SwmToken path="/cache.go" pos="118:2:2" line-data="// Get an item from the cache. Returns the item or nil, and a bool indicating">`Get`</SwmToken> method fetches items from the cache, checking for expiration before returning the item.

```
// Get an item from the cache. Returns the item or nil, and a bool indicating
// whether the key was found.
func (c *cache) Get(k string) (interface{}, bool) {
	c.mu.RLock()
	// "Inlining" of get and Expired
	item, found := c.items[k]
	if !found {
		c.mu.RUnlock()
		return nil, false
	}
	if item.Expiration > 0 {
		if time.Now().UnixNano() > item.Expiration {
			c.mu.RUnlock()
			return nil, false
		}
	}
	c.mu.RUnlock()
	return item.Object, true
}
```

---

</SwmSnippet>

<SwmSnippet path="/cache.go" line="182">

---

- **Incrementing and decrementing values**: The cache supports incrementing and decrementing numeric values stored in items, with type-specific methods for various numeric types.

```
// Increment an item of type int, int8, int16, int32, int64, uintptr, uint,
// uint8, uint32, or uint64, float32 or float64 by n. Returns an error if the
// item's value is not an integer, if it was not found, or if it is not
// possible to increment it by n. To retrieve the incremented value, use one
// of the specialized methods, e.g. IncrementInt64.
func (c *cache) Increment(k string, n int64) error {
	c.mu.Lock()
	v, found := c.items[k]
	if !found || v.Expired() {
		c.mu.Unlock()
		return fmt.Errorf("Item %s not found", k)
	}
	switch v.Object.(type) {
	case int:
		v.Object = v.Object.(int) + int(n)
	case int8:
		v.Object = v.Object.(int8) + int8(n)
	case int16:
		v.Object = v.Object.(int16) + int16(n)
	case int32:
		v.Object = v.Object.(int32) + int32(n)
	case int64:
		v.Object = v.Object.(int64) + n
	case uint:
		v.Object = v.Object.(uint) + uint(n)
	case uintptr:
		v.Object = v.Object.(uintptr) + uintptr(n)
	case uint8:
		v.Object = v.Object.(uint8) + uint8(n)
	case uint16:
```

---

</SwmSnippet>

<SwmSnippet path="/cache.go" line="904">

---

- **Deleting items**: Items can be deleted manually, and expired items are automatically removed during cleanup operations.

```
// Delete an item from the cache. Does nothing if the key is not in the cache.
func (c *cache) Delete(k string) {
	c.mu.Lock()
	v, evicted := c.delete(k)
	c.mu.Unlock()
	if evicted {
		c.onEvicted(k, v)
	}
}
```

---

</SwmSnippet>

# Subsystem

<SwmSnippet path="/cache.go" line="1076">

---

The cache maintenance subsystem includes a janitor routine that periodically cleans up expired items. This subsystem is crucial for ensuring that the cache does not retain stale data and operates efficiently.

```
func (j *janitor) Run(c *cache) {
	ticker := time.NewTicker(j.Interval)
	for {
		select {
		case <-ticker.C:
			c.DeleteExpired()
		case <-j.stop:
			ticker.Stop()
			return
		}
	}
}
```

---

</SwmSnippet>

<SwmSnippet path="/cache.go" line="1089">

---

The janitor is managed by functions that start and stop its operation, ensuring that cleanup occurs at regular intervals.

```
func stopJanitor(c *Cache) {
	c.janitor.stop <- true
}

func runJanitor(c *cache, ci time.Duration) {
	j := &janitor{
		Interval: ci,
		stop:     make(chan bool),
	}
	c.janitor = j
	go j.Run(c)
}
```

---

</SwmSnippet>

# Conclusion

The caching system in <SwmPath>[cache.go](/cache.go)</SwmPath> is designed for efficient item management with expiration capabilities. The use of mutexes ensures thread-safe operations, while the janitor subsystem maintains cache integrity by removing expired items. This design provides a robust solution for caching needs, balancing performance with data freshness.

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBZ28tY2FjaGUlM0ElM0FzaGFpbmVzZHU=" repo-name="go-cache"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
