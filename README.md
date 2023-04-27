# WordPress Object Cache "info" specification

WordPress' `WP_Object_Cache` needs a spec to retrieve standardized information, such as hits, backend engine and various groups.

## Global functions

Three global public functions expose WordPress object cache information. These functions are intended for use from WP-CLI, site health, plugins, and other software.

1. `wp_cache_config()` retrieves configuration data. This function runs immediately, without incurring any IO, database lookups, or network traffic.
2. `wp_cache_stats()` retrieves statistics about the operation of the cache. This function may incur IO, database lookups, or network traffic. 
3. `wp_cache_reset_stats()` zeroes any accumulated statistics.  This function may incur IO, database lookups, or network traffic.

These functions may disclose sensitive information and so should be used only by authenticated administrative users. In multisite instances they return global information and do not depend on the current $blog_id setting (**TODO** Verify this).

### wp_cache_config()

The globally defined function `wp_cache_config()` retrieves the configuration of the WordPress cache. It is defined as follows.

```php
/**
 * Retrieve the configuration of the object cache.
 *
 * @return object The configuration.
 */
 function wp_cache_config() {
   $global $wp_object_cache;
   return $wp_object_cache->wp_cache_config();
 }
```
It is inexpensive to call. 

### wp_cache_stats()

The globally defined function `wp_cache_stats()` retrieves operational statistics from the WordPress cache. It is defined as follows.

```php
/**
 * Retrieve object cache operational statistics, such as hit rates.
 *
 * @param bool $extended [optional] Return extended statistics if true.
 *
 * @return object The configuration.
 */
function wp_cache_stats( $extended = true ) {
   $global $wp_object_cache;
   return $wp_object_cache->wp_cache_stats( $extended );
}
```
This function may incur IO, database operations, or network traffic. It need not be inexpensive to call.

### wp_cache_reset_stats()

The globally defined function `wp_cache_reset_stats()` zeros accumulated operational statistics.

```php
/**
 * Reset object cache operational statistics, such as hit rates.
 *
 * @return null
 */
function wp_cache_reset_stats() {
   $global $wp_object_cache;
   $wp_object_cache->wp_cache_reset_stats();
}
```
This function may incur IO, database operations, or network traffic. It need not be inexpensive to call. 

It must be implemented even for implementations that do not gather and store statistics.

## Public functions in WP_Object_Cache class implementations.

The global functions use corresponding functions in object class implementations, as shown in the preceding section. 

## Returned Objects

###  wp_cache_config()

This returns 

```php
(object) array (
  'plugin'             => 'redis-cache',  // The plugin slug, or null if none.
  'plugin_version'     => '2.3.0',        // The plugin's version, or null if none.
  'plugin_description' => 'text',         // A possibly localized text string.
  'extension'          => 'redis',        // The name of the php extension, or null if none.
  'extension_version'  => '5.1.1',        // The version of the php extension, or null if none.
  'serializers'        => 'php,igbinary,json', // Available serializer names, csv. The active one is first.
  'groups'  => (object) array (
             'global_groups' => [],          // Current set of global groups. 
             'non_persistent_groups' => [],  // Current set of non-persistent groups.
             'unflushable_groups'    => [],  // Current set of unflushable groups.
   'errors'  => array (                      // An array, possibly empty, of error messages.
         (object) array (
               'time'    => 1612345678.9,  // Timestamp of error.
               'code'    => 'TIMEOUT',     // Error code (example).
               'message' => 'Message',     // Error message.   
         ), ...                          
   ),
)
```

The `errors` property of this object includes only errors from the current pageview.

Callers can rely on all properties of this object being present, even if some have null values or empty array.

## wp_cache_stats()

This is complementary to the information returned by wp_cache_config(). It returns 

```php
(object) array (
   'errors'  => array (                      // An array, possibly empty, of error messages.
         (object) array (
               'time'    => 1612345678.9,  // Timestamp of error.
               'code'    => 'TIMEOUT',     // Error code (example).
               'message' => 'Message',     // Error message.   
         ), ...                          
   ),
   'stats' => (object) array (
            'start'       => 1612345600.1   // Timestamp, start of statistics recording.
            'end'         => 1612345678.9   // Timestamp, end of statistics recording.
            'stats' => (object) array (
               'overall' => (object) array (     // Overall stats.
                  'hits'        => 4,            //  Cache hits.
                  'misses'      => 1,            //  Cache misses.
                  'ratio'       => 0.25,         //  Hit ratio [0.0-1.0].
               ),
               'local'  => (object) array (      // Local cache stats, may not be present.
                  'hits'        => 4,            //  Cache hits.
                  'misses'      => 1,            //  Cache misses.
                  'ratio'       => 0.25,         //  Hit ratio [0.0-1.0].
               ),
               'store'  => (object) array (      // Persistent cache stats, may not be present.    
                  'hits'        => 4,            //  Cache hits.
                  'misses'      => 1,            //  Cache misses.
                  'ratio'       => 0.25,         //  Hit ratio [0.0-1.0].
               ),
             ),
     ),
     'measures' => array (
        (object) array (
           'op'           => 'get_multiple',              // Operation, not localized.
           'stat'         => 'n',                         // Measure, not localized ('max', 'min', etc).
           'description'  => 'Number of get_multiple ops' // Description, localized.
           'value'        => 123                          // Value.
        ), ... // Any number of items in this array.
     ),
)
```
The `errors` property of this object includes errors from the current pageview, and may include errors from previous pageviews if available.

The `stats` property must contain an `overall` property, and may or may not contain 'local' and 'store' properties depending on the implementation.

The `measures` array must be present, but it need not contain any elements. If it does contain elements, each one presents a measure gathered by the implementation.

The `start` timestamp shows the beginning time of the statistics gathering. It should reflect the time of the most recent `wp_cache_reset_stats()` call. An implementation may choose to keep statistics for a limited time period.
