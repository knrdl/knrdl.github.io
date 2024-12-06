---
title: "LocalForage: Performant batch operations"
date: "2024-12-06"
tags: [indexeddb, localforage, javascript, typescript]
lang: "en"
---

# Problem

[LocalForage](https://github.com/localForage/localForage) is a thin wrapper around [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) (and some fallbacks). IndexedDB is a NoSQL database API exposed by modern browsers. It allows storing JSON and blobs client-side. LocalForage simplifies interacting with IndexedDB by providing a simple key-value store as API instead. The method names are [localStorage](https://developer.mozilla.org/de/docs/Web/API/Window/localStorage)-like: `getItem()`, `setItem()`, `removeItem()` etc.

However, batch operations (like a mass insert) are horribly slow when performed via LocalForage. That's because LocalForage opens for every operation a new IndexedDB transaction and this action is remarkable time-consuming compared with other DBMS. So 1000 `setItems()` open 1000 transactions which can take multiple seconds to proceed. As LocalForage doesn't provide the API to do this in a single transaction, you need to write your own thin wrapper around IndexedDB and expose transactions as entity.

# Solution

```typescript
export function kvDb(name: string = 'kvdb') {
  let openedDb: IDBDatabase | undefined

  function idbRequestToPromise<T>(request: IDBRequest<T>) {
    return new Promise<T>((resolve, reject) => {
      request.onsuccess = () => resolve(request.result)
      request.onerror = () => reject(request.error)
    })
  }

  async function openDb() {
    const dbRequest = indexedDB.open(name)
    dbRequest.onupgradeneeded = () => {
      dbRequest.result.createObjectStore('kv-pairs')
    }
    return await idbRequestToPromise(dbRequest)
  }

  return {
    /**
     * A transaction is autocommitted as soon as no new requests are made (!)
     * All requests must be added in the same cycle of the event loop.
     * The transaction cannot be used afterwards.
     */
    async transaction() {
      if (!openedDb) openedDb = await openDb()
      return openedDb.transaction('kv-pairs', 'readwrite').objectStore('kv-pairs')
    },

    async get<T>(key: string, transaction?: IDBObjectStore): Promise<T> {
      const store = transaction ?? (await this.transaction())
      const value: T | undefined = await idbRequestToPromise(store.get(key))
      if (value === undefined) throw new Error(`db "${name}" key "${key}" not found`)
      else return value
    },

    async getDefault<T>(key: string, defaultValue: T, transaction?: IDBObjectStore): Promise<T> {
      const store = transaction ?? (await this.transaction())
      const value: T | undefined = await idbRequestToPromise(store.get(key))
      if (value === undefined) return defaultValue
      else return value
    },

    async set<T>(key: string, value: T, transaction?: IDBObjectStore) {
      const store = transaction ?? (await this.transaction())
      await idbRequestToPromise(store.put(value, key))
    },

    async delete(key: string, transaction?: IDBObjectStore) {
      const store = transaction ?? (await this.transaction())
      await idbRequestToPromise(store.delete(key))
    },

    async clear(transaction?: IDBObjectStore) {
      const store = transaction ?? (await this.transaction())
      await idbRequestToPromise(store.clear())
    }
  }
}
```

As additional optimization the browser can be asked to persist the storage as long as possible:
```typescript
try {
  await navigator.storage?.persist?.()
} catch (e) {
  console.error('navigator.storage.persist() failed:', e)
}
``` 

## Usage example

```typescript
const localCache = kvDb('cache')

// Without transaction (slow on batch):

await localCache.clear()  // clear cache
await localCache.set('object1', {hello: 'world'})
await localCache.get('object1')  // = {hello: 'world'}
await localCache.getDefault('object2', 'fallback')  // = 'fallback'
await localCache.delete('object1')


// With transaction (fast on batch):

const transaction = await localCache.transaction()

await Promise.allSettled(
  Array.from(Array(1000).keys()).map(
    async i => localCache.set(`file-${i}`, await loadFile(i), transaction)
  )
)
```

**Gotchas:** A transaction can only be used once with [`Promise.all()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) or [`Promise.allSettled()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled). Using the same transaction in multiple consecutive statements is not possible (thanks to the [weird IndexedDB design](https://developer.mozilla.org/en-US/docs/Web/API/IDBTransaction#:~:text=A%20transaction%20alternates%20between%20active%20and%20inactive%20states%20between%20event%20loop%20tasks.)).