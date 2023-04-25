# WordPress Object Cache "info" specification

WordPress' `WP_Object_Cache` needs a spec to retrieve standadized information, such as hits, backend engine and various groups.

## Example

```php
function wp_cache_info()
{
    return (object) [
        'status' => WP_OBJECTCACHE_CONNECTED,
        'store' => 'Redis',
        'stats' => (object) [
            'hits' => 0,
            'misses' => 0,
            'ratio' => 0,
            'store_hits' => 0,
            'store_misses' => 0,
            'store_ratio' => 0,
            // ...
        ],
        'groups' => (object) [
            'global' => [],
            'non_persistent' => [],
            // ...
        ],
        'errors' => [],
        'meta' => array_filter([
            'foo' => 'custom',
            'bar' => 'yepnope',
        ]),
    ];
}
```
