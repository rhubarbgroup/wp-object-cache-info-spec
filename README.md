# WordPress Object Cache "info" specification

WordPress's [`WP_Object_Cache`](https://developer.wordpress.org/reference/classes/wp_object_cache/) needs a spec to retrieve standardized information, such as hits, backend engine and various groups.

## Background

A persistent WordPress object cache is a two-level cache. Level 1 caches items in RAM, and Level 2 caches them in a persistent backing store. Various WordPress plugins offer access to various types of backing store. The purpose of this specification is unifying the way the various plugins report their efficacy and performance.  

The core object cache (as of WordPress 6.2, April 2023) is a single-level RAM-only cache. As the handling for each request begins, the RAM cache is empty. Operations (database retrievals and computations) during request handling place items into the cache, and when they are needed again they can be read from the cache. Each cache reading operation returns either the cached item or a false `$found`. Thus, the single-level cache's efficacy is measured by these numbers:

* Found items in the cache (hits).
* Requested items from the cache (total hits and misses).
* Stored items.

The hit rate is defined as Found items / Requested items. 

When a plugin introduces a second level persistent cache, the Level 1 RAM cache is empty as each request begins. Each cache reading operation can have three outcomes, as follows.

1. The item is found in the RAM cache and returned to the caller: a Level 1 hit.
2. The item is not found in the RAM cache, but is found in the backing store. It is loaded into the RAM cache and returned to the caller: a Level 2 hit.
3. Ihe item is not found in either cache: a Level 2 miss.

When a caller stores an item in a persistent cache, the caching code stores it in the Level 1 RAM cache and writes it through to the Level 2 persistent cache. The write-through operation can be optimized away if the cached value of the stored item is already the same as the value being provided. Thus, the efficacy of the two-level cache can be measured with these numbers.

* Found items in the Level 1 cache (cheap hits).
* Found items in the Level 2 cache (expensive hits).
* Requested items from the cache, totaled. (Level 1 hits, Level 2 hits, and misses)
* Items stored by callers into the cache.
* Items written through from the Level 1 cache to the Level 2 cache.

## Scope

This specification is intended to be implemented by each object cache implementation, including core's WP_Object_Cache and various plugins. 

It is intended for use by diagnostic software such as Site Health and debugging plugins such as Query Monitor.

Terminology: It uses the phrases MUST / SHOULD / MAY / SHOULD NOT / MUST NOT as defined in [RFC2119](https://www.rfc-editor.org/rfc/rfc2119). 

## Global functions

These global public functions expose WordPress object cache information.

1. `wp_cache_get_config()` retrieves configuration data. 
2. `wp_cache_get_errors()` retrieves a list of accumulated errors. 
3. `wp_cache_get_stats()` retrieves statistics about the operation of the cache. 
4. `wp_cache_reset_stats()` zeroes any accumulated statistics.  This function may incur IO, database lookups, or network traffic.
5. `wp_cache_get_global_groups()` returns the names of the global cache groups.
6. `wp_cache_get_non_persistent_groups()` returns the names of the global cache groups.

These functions disclose sensitive information. The information they produce MUST NOT be visible to unauthenticated users or users without administrative privilege. In multisite instances they return global information and do not depend on the current $blog_id setting.

### wp_cache_get_config()

This function answers the question "what kind of cache is in use?"  A cache implementation SHOULD implement this function. If `true === function_exists( 'wp_cache_get_config' )` then the implementation MUST comply with this specification. 

The globally defined function `wp_cache_get_config()` retrieves the configuration of the WordPress cache. It is defined as follows.

```php
/**
 * Retrieve the configuration of the object cache.
 *
 * @return array The configuration array.
 */
 function wp_cache_get_config() {
   global $wp_object_cache;
   return $wp_object_cache->wp_cache_get_config();
 }
```
This function SHOULD be inexpensive to call. It SHOULD require O(1) time in the number of cached items, and SHOULD NOT require establishing any sort of network connection. 

It returns the following array.

```php
array (
  'status'             => 'active',       // A short nonlocalized text string showing the status.
  'status_description' => 'Connected',    // A localized text string showing the status.
  'plugin'             => 'redis-cache',  // The plugin's slug, or null if none.
  'plugin_version'     => '2.3.0',        // The plugin's version, or null if none.
  'plugin_description' => 'text',         // A localized text string, or null if none
  'extension'          => 'redis',        // The name of the php extension, or null if none.
  'extension_version'  => '5.1.1',        // The version of the php extension, or null if none.
  'serializer'         => 'igbinary',     // Object serializer in use ('php', 'json') or null if unknown.
)
```

The `status` element contains strings such as 'connected', 'disconnected'. A null value or the string 'none' means no Level 2 cache is available. 

### wp_cache_get_errors()

A cache implementation MAY implement this function. If `true === function_exists( 'wp_cache_get_errors' )` then the implementation MUST comply with this specification. 

The globally defined function `wp_cache_get_errors()` retrieves accumulated errors, if any.

```php
/**
 * Retrieve object cache errors.
 *
 * @param bool $clear Clear the accumulated errors. Default true.
 *
 * @return [WP_Error] An array of WP_Errors.
 */
 function wp_cache_get_errors( $clear = true ) {
   global $wp_object_cache;
   return $wp_object_cache->wp_cache_get_errors( $clear );
 }
```

This function returns a (possibly empty) array containing errors accumulated by the cache implementation.

If the `$clear` parameter is false, the implementation MAY retain accumulated errors. If the `$clear` paraemter is true (the default), the implementation MUST NOT retain errors after returning them.

### wp_cache_get_stats()


The globally defined function `wp_cache_get_stats()` retrieves operational statistics from the WordPress cache. It is defined as follows.

```php
/**
 * Retrieve object cache operational statistics, such as hit rates.
 *
 * @return array The statistics.
 */
function wp_cache_get_stats() {
   global $wp_object_cache;
   return $wp_object_cache->wp_cache_get_stats();
}
```

The function returns an array with elements as follows.

```php
array (
   '1' => array (
       'hit'  => 95,   // Total number of hits returned to caller.
       'miss' => 5,   // Total number of misses returned to caller.
    ), 
   '2' => array (
       'hit'  => 5,   // Total number of hits returned to caller.
       'miss' => 5,   // Total number of misses returned to caller.
    ), 
)
```

The array has one element for each cache level. When no persistent backing store is available, the array has only one element.  

An example serves to explain these numbers.  Consider the scenario shown above. From the cache caller's perspective, the cache hit rate is 95%. That's shown by the first array element.

But, looking at the second array element, we see that the Level 1 RAM cache passed along 10 cache requests to the Level 2 persistent cache. 5 were hits and 5 were misses, passed back to the Level 1 cache.

There's a subtlety in this multi-level cache handling. The Level 1 cache MAY keep a bit of metadata for each cache item saying it's known to be missing from the Level 2 cache. In that case the Level 1 cache does not pass on the request to Level 2 but rather immediately returns a cache miss.

It's conceivable (if unlikely) that a cache implementation could have more than two levels. This returned array contains an element for each cache level.

This function MAY incur IO, database operations, or network traffic. It MAY be expensive to call.

 A caller looking for a single cache-efficiency fractional number can use this expression. 
 
 ```php
 $stats[1]['hit'] / ($stats[1]['hit'] + $stats[1]['miss'])
 ```
 
 The same expression can be used to report the efficiency of the Level 2 cache.
 
 To determine the Level 1 cache's ability to fulfill requests without resorting to the Level 2 cache this expression serves.
 
 ```php
 ($stats[1]['hit'] - ( $stats[2]['hit'] - $stats[2]['miss'] ) ) / ($stats[1]['hit'] + $stats[1]['miss'])
 ```
 

### wp_cache_reset_stats()

The globally defined `wp_cache_reset_stats()` function zeros the counters used to generate statistics. Its purpose is allowing other software to take time-bounded samples of cache statistics.

An implementation MAY define this function. If it is defined, when invoked  the implementation MAY set to zero its counters for the statistics returned by `wp_cache_stats()`.  (That is, an implementation may choose not to implement this function, or, if implementing it may choose to have it do nothing.)

```php
/**
 * Reset object cache operational statistics, such as hit rates.
 *
 * @return null
 */
function wp_cache_reset_stats() {
    global $wp_object_cache;
    $wp_object_cache->wp_cache_reset_stats();
}
```

### wp_cache_get_global_groups()

We also need APIs to query global and non-persistent groups.
The globally defined `wp_cache_get_global_groups()` returns the current set of global cache groups. It is defined as follows.

```php
/**
 * Gets the list of global groups.
 *
 * @return [string] An array of the names of global groups.
 */
 function wp_cache_get_global_groups() {
     global $wp_object_cache;
     $wp_object_cache->wp_cache_get_global_groups();
 } 
```


### wp_cache_get_non_persistent_groups()

The globally defined `wp_cache_get_non_persistent_groups()` returns the current set of non-persistent cache groups. It is defined as follows.

```php
/**
 * Gets the list of non-persistent groups, those not saved to a Level 2 cache.
 *
 * @return [string] An array of the names of non-persistent groups.
 */
 function wp_cache_get_non_persistent_groups() {
     global $wp_object_cache;
     $wp_object_cache->wp_cache_get_non_persistent_groups();
 } 
```


## Public functions in WP_Object_Cache class implementations.

The global functions use corresponding functions in object class implementations, as shown in the preceding sections. 

