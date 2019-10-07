---
title: Key Value Store
date: 2019-10-07T00:00:00-05:00
subsection: Plugins (Beta)
weight: 110
---

Most plugins require some kind of persistent storage. For example:
* The GitHub plugin [persists personal access tokens](https://github.com/mattermost/mattermost-plugin-github/blob/0992932eb1e83041a05ef250710d726f5bc94b06/server/plugin.go#L191-L195) on a per-user basis
* The Jira plugin [persists Jira instance metadata](https://github.com/mattermost/mattermost-plugin-jira/blob/d111b57064bc2e921dd6a5f50d0d4038e6b24a58/server/kv.go#L280-L303) for the plugin to access as needed.
* The upcoming [suggestions plugin](https://github.com/mattermost/mattermost-plugin-suggestions) stores previous computation results to enable incremental updates.

While all Mattermost installations have access to a database, we don't encourage directly writing to the database since we do not guarantee schema compatibility for plugins. Instead, the plugin API exposes a key value store implemented as a series of RPC calls and plugin helper methods:
* [KVGet](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVGet)
* [KVSet](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVSet)
* [KVDelete](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVDelete)
* [KVDeleteAll](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVDeleteAll)
* [KVList](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVList)
* [KVSetWithExpiry](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVSetWithExpiry)
* [KVCompareAndSet](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVCompareAndSet)
* [KVCompareAndDelete](https://developers.mattermost.com/extend/plugins/server/reference/#API.KVCompareAndDelete)
* [KVGetJSON](https://developers.mattermost.com/extend/plugins/server/reference/#Helpers.KVGetJSON)
* [KVSetJSON](https://developers.mattermost.com/extend/plugins/server/reference/#Helpers.KVSetJSON)
* [KVSetWithExpiryJSON](https://developers.mattermost.com/extend/plugins/server/reference/#Helpers.KVSetWithExpiryJSON)
* [KVCompareAndSetJSON](https://developers.mattermost.com/extend/plugins/server/reference/#Helpers.KVCompareAndSetJSON)
* [KVCompareAndDeleteJSON](https://developers.mattermost.com/extend/plugins/server/reference/#Helpers.KVCompareAndDeleteJSON)

This document outlines how and when to use these methods, and how to architect your plugin to reflect common persistent storage requirements.

# How does the key value store work?

* Written to the database (https://github.com/mattermost/mattermost-server/blob/master/store/sqlstore/plugin_store.go)
* `PluginKeyValueStore` table has columns `PluginId`, `PKey`, `PValue`, `ExpireAt`
* Writes automatically set `PluginId`, ensuring no two plugins collide.
* Key has maximum length 50 runes.
* Value has maximum length 4Mb
* ExpireAt is an optional timestamp in milliseconds after which the value should effectively be deleted. It may not be deleted immediately, but won't be returned in a `KVGet` and will be silently overwritten in a `KVSet`.

# How do I save user-specific data?

* Consider writing your own helper function to build a key that embeds the user's unique identfier:
```go
func hashUserKey(userId string) string {
    return fmt.Sprintf("user-%s", userId)
}
```
* Wrap the key value store with helper methods that expect a `userId` and automatically use compute the key:
```
func (p *Plugin) getUserData(userId string) ([]byte, error) {
    key := hashUserKey(userId)
    return p.API.KVGet(key)
}

func (p *Plugin) setUserData(userId string, data []byte) error {
    key := hashUserKey(userId)
    return p.API.KVSet(key, data)
}
```

# How do I avoid race conditions?

Always keep concurrency in mind when writing your plugin:
* The same user may make multiple requests concurrently to your HTTP API
* Plugins are started once per server. In a high-availability cluster, multiple instances of your plugin may be running at the same time.

When reading and writing to the key value store, there is a critical section in which a concurrent write may be lost:
```go
func unsafeCode() *model.AppError {
    oldData, err := p.API.KVGet("key")
    if err != nil {
        return err
    }

    updatedData := updateData(oldData)

    // This is unsafe: another request may have already written a new value with updated data.
    err := p.API.KVSet("key", updatedData)
    if err != nil {
        return err
    }

    return nil
}
```

While locking would guard against multiple goroutines running in this critical section, it cannot guard against multiple plugin processes on different servers. Instead, write this using the `KVCompare*` API methods:
```go
func safeCode() error {
    for i := 0; i < 3; i++ {
        oldData, err := p.API.KVGet("key")
        if err != nil {
            return err
        }

        updatedData := updateData(oldData)

        // This is safe: if another request has already updated the data, the API will return false.
        ok, err := p.API.KVCompareAndSet("key", oldData, updatedData)
        if err != nil {
            return err
        }
        if !ok {
            p.API.LogWarning("Concurrent update to data detected; trying again")
            continue
        }

        return nil
    }

    return errors.New("aborted after too many concurrent failures trying to update data; aborting")
}
```

Under the covers, the server writes this value to the database using the following `UPDATE` query:
```sql
UPDATE PluginKeyValueStore SET PValue = :New WHERE PluginId = :PluginId AND PKey = :Key AND PValue = :Old
```

The database guarantees that if two threads are racing to write different values, only only one such `UPDATE` will succeed, with the other being rolled back. Note that it's not sufficient to just get `KVGet` the new value and transparently write again: your plugin must be prepared to repeat the request from scratch to ensure you take into account changes from the concurrent write.

# How can I store structured data?

* Use the `*JSON*` API methods to automatically serialize to and from JSON. (example needed)

# When are expiring key values useful?

* Use expiring key values instead of manually cleaning up data you now consider invalid.
* Expiring keys are also useful to implement a cluster-wide lock (link/example needed)

# Why are keys and values limited in length?

* We used to hash keys to allow arbitrary length, but this made `KVList` less useful
    * for backwards compatibility, the server falls back to the old hashed key in case it was written prior to v5.6
* Writing values longer than 4Mb can break some MySQL installations
