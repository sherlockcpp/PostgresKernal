simplehash.h
使用规则:

simplehash中提供一些可以让用户定义的宏
simplehash内部的hash相关的函数中使用了这些未定义的宏
在用户定义了之后，相当于使用用户定制的hash规则来进行简单的hash操作
```c
#define SH_PREFIX resultcache
#define SH_ELEMENT_TYPE ResultCacheEntry
#define SH_KEY_TYPE ResultCacheKey *
#define SH_KEY key
#define SH_HASH_KEY(tb, key) ResultCacheHash_hash(tb, key)
#define SH_EQUAL(tb, a, b) ResultCacheHash_equal(tb, a, b) == 0
#define SH_SCOPE static inline
#define SH_STORE_HASH
#define SH_GET_HASH(tb, a) a->hash
#define SH_DEFINE
#include "lib/simplehash.h"
```