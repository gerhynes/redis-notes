# Redis Notes

Sources:
- [Redis Docs](https://redis.io/docs/)
- [Redis Crash Course](https://youtu.be/OqCK95AS-YE)

Redis (from Remote Dictionary Server) is an open-source in-memory data structure store, used as a database, cache, message broker, and streaming engine. 

Redis provides data structures such as strings, hashes, lists, sets, sorted sets with range queries, bitmaps, hyperloglogs, geospatial indexes, and streams.

To achieve top performance, Redis works with an in-memory dataset. Depending on your use case, Redis can persist data either by periodically dumping the dataset to disk or by appending each command to a disk-based log.

Redis supports asynchronous replication, with fast non-blocking synchronization and auto-reconnection with partial resynchronization on net split.

Redis is often used as a cache on top of other databases to improve performance. However, Redis is a fully-fledged primary database that can be used to store and persist multiple data formats for complex applications.

## Redis Data Types
### Keys
Redis keys are binary safe, so you can use any binary sequence as a key, from a string to the content of a JPEG. An empty string is also a valid key.

Very long keys are not a good idea. The lookup of the key in the dataset may require several costly key-comparisons. Hashing a large value (such as with SHA1) is a better idea from the perspective of memory and bandwidth.

Very short keys are often also not a good idea. There is little benefit to writing "u1000flw" instead of "user:1000:followers". Readability makes up for the extra memory consumed.

Try to stick with a schema. Dots or dashes are often used for multi-word fields, for example: "comment:4321:reply.to" or "comment:4321:reply-to".

The maximum allowed key size is 512 MB.

### Strings
The Redis string type is the simplest type of value you can associate with a Redis key. It's also the only data type in memcached, and so natural for newcomers to Redis.

Since Redis keys are strings, when you use a string type as a value too, you are mapping a string to a string. The string data type is useful for a number of use cases, such as caching HTML fragments or pages.

```bash
set mykey somevalue 
# OK

get mykey 
# "somevalue"
```

The `SET` and `GET` commands are the way to set and retrieve a string value. `SET` will replace any existing value already stored in the key, even if the key is associated with a non-string value.

`SET` can take arguments, for example to fail if the key already exists or only succeed if the key already exists.

```bash
set mykey newval nx 
# (nil)

set mykey newval xx 
# OK
```

Strings can be atomically incremented or decremented using the `INCR`, `INCRBY`, `DERC`, and `DECRBY` commands. These parse the string as an integer, increment/decrement it, and set the obtained value as the new value.

Since these commands are atomic, multiple clients issuign `INCR` commands against the same key will never enter a race condition.  The read-increment-set operation is performed while all the other clients are not executing a command.

```bash
set counter 100

incr counter 
# (integer) 101

incrby counter 50 
# (integer) 151
```

The `GETSET` command sets a key to a new value, returning the old value as the result.

To reduce latency, you can set or retrieve the values of multiple keys in a single command: `MSET` and `MGET`. When `MGET` is used, Redis returns an array of values.

```bash
mset a 10 b 20 c 30 
# OK

mget a b c
# 1) "10"
# 2) "20"
# 3) "30"
```

### Altering and Querying the Key Space
Some commands are not defined on particular types and can be useful when interacting with keys of any type.

`EXISTS` returns 1 or 0 to signal if a given key exists in the database.

`DEL` deletes a key and associated value, whatever the value is, returning 1 if the key existed and was removed or 0 if no such key was found.

`TYPE` returns the kind of value stored at the specified key.

```bash
set mykey hello 
# OK

type mykey 
# string

exists mykey 
# (integer) 1

del mykey 
# (integer) 1

type mykey 
# none

exists mykey 
# (integer) 0
```

### Key Expiration
Key expiration lets you set a timeout/"time to live" for a key. When the time to live expires, the key is automatically destroyed.

Key expiration can be set using either second or millisecond precision. The expire time resolution is always 1 millisecond.

Information about expires are replicated and persisted on disk. The time virtually passes when your Redis server remains stopped, meaning Redis saves the darte at which a key will expire.

Use `EXPIRE` to set a key's expiration. Use `PERSIST` to remove the expire and make the key persist forever.

```bash
set mykey somevalue 
# OK

expire key 5 
# (integer) 1

# after some time
get key 
# (nil)
```

You can create keys with expires using other commands.

```bash
set key 100 ex 10
# OK

ttl key
# (integer) 9
```

### Lists
Redis lists are implemented via linked lists. This means that even if you have millions of elements inside a list, the operation of adding a new element to the head or tail of the list is performed in constant time. The speed of using `LPUSH` to add an element to the head of a list with ten elements is the the same as adding to the head of a list with 10 million elements.

The downside is that accessing an element by index is slower than in lists implemented with an array. The operation requires an amount of work proportional to the index of the accessed element. For fast access to the middle of a large collection, Redis uses sorted sets.

The `LPUSH` command adds a new element to the head of a list, while `RPUSH` adds a new element to the tail of a list.

`LRANGE` extracts ranges of elements from a list.

```bash
rpush mylist A
# (integer) 1

rpush mylist B
# (integer) 1

lpush mylist first
# (integer) 3

lrange mylist 0 -1
# 1) "first"
# 2) "A"
# 3) "B"
```

`LRANGE` takes two indexes, the first and last element of the range to return. Both indexes can be negative, telling Redis to start counting from the end: -1 is the last element, -2 the penultimate.

Both `LPUSH` and `RPUSH` can push multiple elements into a list in a single call.

```bash
rpush mylist 1 2 3 4 5 "and a string"
# (integer) 9

lrange mylist 0 -1
# 1) "first"
# 2) "A"
# 3) "B"
# 4) "1"
# 5) "2"
# 6) "3"
# 7) "4"
# 8) "5"
# 9) "and a string"
```

You can also pop elements from the left or right of a list. Popping an element both retrieves it from the list and eliminates it from the list at the same time.

Redis will return a null value if there are no elements in the list.

```bash
rpush mylist 1 2 3
# (integer) 3

rpop mylist
# "c"

rpop mylist
# "b"

rpop mylist
# "a"

rpop mylist
# (nil)
```

Representative use cases for lists include:
- Remembering the latest updated posts by users
- Communicating between processes, using a consumer-producer pattern where the producer pushes items into a list and a consumer (usually a worker) consumes those items and executes actions. Redis has special list commands to make this use case both more reliable and efficient. 

For example, image your home page shows your latest photos published and you want to speed up access.
- Every time a user posts a new photo, its ID is added into a list with `LPUSH`
- When users visit the homepage, `LRANGE 0 9` is used to get the latest 10 posted items.

### Capped Lists
In many use cases you just want a list to store the latest items, whether they are logs, social media updates, etc.

Redis lets you use lists as a capped collection, only remembering the latest n items and discarding all the oldest items using the `LTRIM` command.

`LTRIM` is similar to `LRANGE` but instead of displaying the specified range of elements it sets this range as the new list value. All elements outside the given range are removed.

```bash
rpush mylist 1 2 3 4 5
# (integer) 5

ltrim mylist 0 2
# OK

lrange mylist 0 -1
# 1) "1"
# 2) "2"
# 3) "3"
```

A simple but useful pattern is doing a list push operation and a list trim operation together to add a new element and discard elements exceeding a limit. For example, adding a new element and taking only the 1000 newest elements as the list.

```bash
lpush mylist some_element
ltrim mylist 0 999
```

#### Blocking Operations on Lists
Lists have a special feature that makes them suitable to implement queues, and in general as a building block for inter-process communication systems: blocking operations.

Imagine you want to push items into a list with one process, and use a different process to do work with those items. This producer/consumer setup can be implemented as follows:
- To push items into the list, producers call `LPUSH`
- To extract/process items from the list, consumers call `RPOP`

However, it's possible that sometimes the list is empty and there is nothign to process, so `RPOP` just returns null. In this case, a consumer is forced to wait some time and retry `RPOP`. This polling is not a good idea in this context because:

- It forces Redis and clients to process useless commands (all the requests when the list is empty will get no work done and just return null)
- It adds a delay to the processing of items, since after a worker receives null, it waits some time. To make the delay smaller, you could wait less time between calls to `RPOP` but this means more useless calls to Redis

Redis' `BRPOP` and `BLPOP` commands are versions of `RPOP` and `LPOP` that are able to block if the list is empty. They will return to the caller only when a new element is added to the list, or when a user-specified timeout is reached.

```bash
# wait for elements in tasks, but return after 5 seconds if no element is available
brpop tasks 5
# 1) "tasks"
# 2) "do_something_else"
```

You can use 0 as a timeout to wait for elements forever, and you can also specify multiple lists in order to wait on multiple lists at the same time, and get notified when the first list receives an element.

Regarding `BRPOP`:
1. Clients are served in an ordered way. The first client that blocked waiting for a list is served first when an element is pushed by some other client
2. The return value is different compared to `RPOP`. It's a two-element array since it also includes the name of the key, because `BRPOP` and `BLPOP` are able to block while waiting for elements from multiple lists
3. If the timeout is reached, null is returned

It's possible to build safer queues, or rotating queues, using `LMOVE`. There is also a blocking variant of this command, called `BLMOVE`

### Automatic Creation and Removal of Keys
With Redis data types composed of multiple elements (Lists, Streams, Sets, Sorted Sets, Hashes), Redis has the following behaviour:

1. When you add an element to an aggregate data type, if the target key does not exist, an empty aggregate data type is created before adding the element
2. When you remove elements from an aggregate data type, if the value remians empty, the key is automatically destroyed (with the exception of Stream)
3. Calling a read-only command, such as `LLEN` (which returns the length of a list), or a write command removing elements, with an empty key, always produces the same result as if the key is holding an empty aggregate type of the type the command expects to find.

### Hashes
Redis hashes consist of field-value pairs.

```bash
hset user:1000 username antirez birthyear 1977 verified 1
# (integer) 3

hget user:1000 username
# "antirez"

hget user:1000 birthyear
# "1977"

hgetall user:1000
# 1) "username"
# 2) "antirez"
# 3) "birthyear"
# 4) "1977"
# 5) "verified"
# 6) "1"
```

While hashes are useful for representing objects, the number of fields you can put inside a hash has no practical limits other than available memory, so you can use hashes in serveral different ways inside an application.

`HSET` sets multiple fields of the hash, while `HGET` retrieves a single field. `HMGET` is similar to `HGET` but returns an array of values:

```bash
hmget user:1000 username birthyear no-such-field
# 1) "antirez"
# 2) "1977"
# 3) (nil)
```

Individual fields can be operated on, such as with `HINCRBY`.

```bash
hincrby user:1000 birthyear 10
# (integer) 1987

hincrby user:1000 birthyear 10
# (integer) 1997
```

Small hashes are encoded differently to make them very memory efficient.

### Sets
Redis sets are unordered collections of strings.

`SADD` adds new elements to a set. There are also commands for testing if a given element already exists, performing the intersection, union or difference between multiple sets, and so on.

```bash
sadd myset 1 2 3
# (integer) 1

smembers myset
# 1. 3
# 2. 1
# 3. 2

sismember myset 3
# (integer) 1

sismember myset 30
# (integer) 0
```

Sets are good for expressing relations between objects. For instance, to implement tags. A simple way to model the problem is to have a set for every object you want to tag. The set contains the IDs of the tags associated with the object. 

For example, if new article ID 1000 is tagged with tags 1, 2, 5, and 77, a set can associate these tag IDs with the news item.

```bash
sadd news:1000:tags 1 2 5 77
# (integer) 4
```

You might also want to have the inverse relation as well, a list of all the news articles tagged with a given tag.

```bash
sadd tag:1:news 1000
# (integer) 1

sadd tag:2:news 1000
# (integer) 1

sadd tag:5:news 1000
# (integer) 1

sadd tag:77:news 1000
# (integer) 1
```

Then to get all tags for a given object:

```bash
smembers news:1000:tags
# 1. 5
# 2. 1
# 3. 77
# 4. 2
```

All this assumes you have another data strcuture (such as a Redis hash) which maps tag IDs to tag names.

If you want a list of all the objects with the tags 1, 2, 10, and 27, you can use the `SINTER` command, which performs the intersection between dffereent sets.

```bash
sinter tag:1:news tag:2:news tag:10:news tag:27:news
```

In addition to intersections, you can perform operations on sets such as unions, difference, extracting a random element.

You can extract a random element from a set and return it to the client using `SPOP`. 

`SUNIONSTORE` performs the union between multiple sets and stores the result in another set. Since the union of a single set is itself, you can use `SUNIONSTORE` to copy a set.

```bash
sunionstore game:1:deck deck
# (integer) 52
```

`SCARD` provides the number of elements inside a set, also known as the cardinality of the set.

```bash
scard game:1:deck
# (integer) 52
```

When you need to get random elements from a set without removing them from the set, use the `SRANDMEMBER` command. It also features the ability to return both repeating and non-repeating elements.

### Sorted Sets
Sorted Sets are a data type similar to a mix between a Set and a Hash. Like sets, sorted sets are composed of unique, non-repeating string elements, so in a sense a sorted set is a set as well.

While elements inside sets are not ordered, every element in a sorted set is associated with a floating point value, called the **score**. This is similar to a hash since every element is mapped to a value.

Elements in a sorted set are taken in order. They are not ordered on request, order is a peculiarity of the data structure used to represent sorted sets.

They are ordered according to the following rule:
- If B and A are two elements with different scores, then A > B if A.score > B.score
- If B and A have exactly the same score, then A > B if the A string is lexicographically greater than the B string. B and A strings can't be equal since sorted sets only have unique elements.

```bash
zadd hackers 1957 "Sophie Wilson"
zadd hackers 1965 "Yukihiro Matsumoto"
zadd hackers 1912 "Alan Turing"
```

`ZADD` is similar to `SADD` but takes one additional argument (placed before the element to be added) which is the score. `ZADD` is variadic, so you are free to specify multiple score-value pairs.

Sorted sets make it easy to return sorted elements becuase they are already sorted by the score. You use `ZRANGE` to access a range between two indexes. You can access elements in reverse using `ZREVRANGE`.

```bash
zrange hackers 0 -1
# 1) "Alan Turing"
# 2) "Sophie Wilson"
# 3) "Yukihiro Matsumoto"
```

Using the `WITHSCORES` argument will return scores as well.

```bash
zrange hackers 0 -1 withscores
# 1) "Alan Turing"
# 2) "1912"
# 3) "Sophie Wilson"
# 4) "1957"
# 5) "Yukihiro Matsumoto"
# 6) "1965"
```

#### Operating on Ranges
Sorted Sets can operate on ranges. The `ZRANGEBYSCORE` command will return all the elements with a score within a specified range (both extremes included).

```bash
zrangebyscore hackers -inf 1950
# 1) "Alan Turing"
```

It's also possible to remove ranges of elements using `ZREMRANGEBYSCORE`. This returns the number of removed elements.

```bash
zremrangebyscore hackers 1940 1960
# (integer) 1
```

The `ZRANK` command lets you perform a get-rank operation, accessing the position of an element in a set of ordered elements. `ZREVRANK` will get the rank, considering the elements sorted in descending order.

```bash
zrank hacker "Alan Turing"
# (integer) 0
```

#### Lexicographical Scores
From Redis 2.8 it's possible to get ranges lexicographically, assuming elements in a sorted set are all inserted with the same identical score.

The main commands to operate with lexicographical ranges are:
- `ZRANGEBYLEX`
- `ZREVRANGEBYLEX`
- `ZREMRANGEBYLEX`
- `ZLEXCOUNT`

```bash
zadd hackers 0 "Sophie Wilson" 0 "Yukihiro Matsumoto" 0 "Alan Turing"

zrange hackers 0 -1
# 1) "Alan Turing"
# 2) "Sophie Wilson"
# 3) "Yukihiro Matsumoto"
```

Using `ZRANGEBYLEX` you can ask for lexicographical ranges. Ranges can be inclusive or exclusive, depending on the first character.

```bash
zrangebylex hackers [B [W
# 1) "Sophie Wilson"
```

This feature allows you to use sorted sets as a generic index

#### Updating Scores
Sorted sets' scores can be updated at any time. Calling `ZADD` against an element already included in the sorted set will update its score (and position) with O(log(N)) time complexity. As such, sorted sets are suitable when there are tons of updates. 

Their characteristics make sorted sets suitable for leader boards. You combine the ability to take users sorted by their high score, plus the get-rank operation, in order to show the top-N users, and the user rank in the leader board.

### Bitmaps
Bitmaps are not an actual data type, but a set of bit-oriented operations defined on the String type. Since strings are binary-safe blobs and their maximum length is 512MB, they are suitable to set up to 2^32 different bits.

Bit operations are divided into two groups: constant-time single-bit operations, like setting a bit to 1 or 0, or getting its value, and operations on groups of bits, for example counting the number of set bits in a given range of bits, such as population counting.

One of the biggest advantages of bitmaps is that they often provide extreme space savings when storing information. For example, in a system where different users are represented by incremental user IDs, it's possible to remember a single bit information (for example, whether a user wants to receive a newsletter) of 4 billion users using just 512MB of memory.

Bits are set and retrieved using the `SETBIT` and `GETBIT` commands. 

`SETBIT` takes as its first argument the bit number, and as its second argument the value to set the bit to, which is 1 or 0. The command automatically enlarges the string if the addressed bit is outside the current string length.

`GETBIT` returns the value of the bit at the specified index. Out of range bits (addressing a bit that is outside the length of the string stored into the target key) are always considered to be zero.

```bash
setbit key 10 1
# (integer) 1
getbit key 10
# (integer) 1
getbit key 11
# (integer) 0
```

There are three commands operating on groups of bits:
1. `BITOP` performs bit-wise operations between different strings. The provided operations  are AND, OR, XOR, and NOT
2. `BITCOUNT` performs population counting, reporting the number of bits set to 1
3. `BITPOS` finds the first bit having the specified value of 0 or 1

Both `BITPOS` and `BITCOUNT` are able to operate with byte ranges of the string, instead of running for the whole length of the strings.

```bash
setbit key 0 1
# (integer) 1
setbit key 100 1
# (integer) 0
bitcount key
# (integer) 2
```

Common uses for bitmaps are:
- Real-time analytics of all kinds
- Storing space-efficient but high performance boolean information associated with object IDs

Bitmaps can be split into multiple keys, for example for the sake of sharding the data set and because in general it is better to avoid working with huge keys. To split a bitmap across different keys instead of setting all the bits into a key, store M bits per key and obtain the key name with bit-number/M, and the Nth bit to address inside the key with bit-number MOD M. 

### HyperLogLogs
A HyperLogLog is a probabilistic data structure used in order to count things (technically estimating the cardinality of a set). 

Usually counting unique items requires using an amount of memory proportional to the number of items you want to count, because you need to remember the elements you have already seen in the past in order to avoid counting them multiple times. However there is a set of algorithms that trade memory for precision: you end with an estimated measure with a standard error (in the Redis implementation, less than 1%). You no longer need to use an amount of memory proportional to the number of items counted, and instead can use a constant amount of memory, 12K bytes in the worst case, and a lot less if the HyperLogLog has seen very few elements.

HyperLogLogs in Redis, while technically a different data structure, are encoded as a Redis string, so you can call `GET` to serialize a HyperLogLog and `SET` to deserialize it back to the server.

Conceptually, the HyperLogLog API is like using Sets to do the same task. You would `SADD` every observed element into a set, and use `SCARD` to check the number of elements inside the set, which are unique since `SADD` will not re-add an existing element.

While you don't really add iterms to a HyperLogLog, because the data structure only contains a state that does not include actual elements, the API is the same:

- Every time you see a new element, you add it to the count with `PFADD`
- Every time you want to retrieve the current approximation of the unique elements added with `PFADD` so far, you use `PFCOUNT`

```bash
pfadd hll a b c d
# (integer) 1
pfcount hll
# (integer) 4
```

An example use case could be counting unique queries performed by users in a search form every day.

## redis-cli
External programs talk to Redis using a TCP socket and a Redis-specific protocol, which is implemented in the Redis client libraries for each supported programming language. Redis also provides a command-line utility to send commands to Redis.

redis-cli has two main modes: an interactive REPL mode where the user types Redis commands and receives replies, and a command mode, where the CLI is executed with additional arguments and the reply is printed to the standard output.

You can check if Redis is working by sending a `ping` command from redis-cli.

Runing `redis-cli` followed by a command and its arguments will send this command to the Redis instance running on localhost port 6379. You can change the host and port using redis-cli.

If you run redis-cli without arguments, the program will start in interactive mode.

```bash
redis-cli

ping 
# PONG

set mykey somevalue 
# OK

get mykey 
# "somevalue"
```

Redis replies are typed and you will see the type of the reply (string, array, integer, nil, error) between parentheses. redis-cli only shows this additional information for human readability when it detectds the standard output is a tty or terminal. For all other outputs it wil auto-enable the raw output mode.

```bash
redis-cli INCR mycounter > /tmp/output.txt
cat /tmp/output.txt
# 8
```

You can enforce raw output on the terminal with the `--raw` option and enforce human-readable output when writing to files or in a pipe to other commands by using `-no-raw`.

### Command Line Usage
#### String Quoting and Escaping
When redis-cli parses a command, whitespace characters automatically delimit the arguments. You can use quoted and escaped strings to input string values that contain whitespaces or non-printable characters.

Double quoted strings support the following escape sequences:
-   `\"` - double-quote
-   `\n` - newline
-   `\r` - carriage return
-   `\t` - horizontal tab
-   `\b` - backspace
-   `\a` - alert
-   `\\` - backslash
-   `\xhh` - any ASCII character represented by a hexadecimal number

Single quotes assume the string is literal, and allow only the following escape sequences:
-   `\'` - single quote
-   `\\` - backslash

```bash
AUTH some_admin_user ">^8T>6Na{u|jp>+v\"55\@_;OU(OR]7mbAYGqsfyu48(j'%hQH7;v*f1H${*gD(Se'"
```

#### Host, Port, Password, and Database
By default, redis-cli connects to the server at 127.0.0.1 on port 6379. To specify a different  hostname or IP adress, use the `-h` option. To set a different port, use the `-p` option.

If your instance is password-protected, the `-a <password>` option will perform authentication saving the need to use the `AUTH` command. 

For security, you should provide the password to redis-cli automatically via the `REDISCLI_AUTH` environment variable.

```bash
redis-cli -a myUnguessablePazzzzzword123 PING
PONG
```

It's possible to send a command that operates on a database number other than the default number zero by using the `-n <dbnum>` option:

``` bash
redis-cli FLUSHALL
# OK
redis-cli -n 1 INCR a
# (integer) 1
redis-cli -n 1 INCR a
# (integer) 2
redis-cli -n 2 INCR a
# (integer) 1
```

Some, or all, of this information can be provided by using the `-u <uri>` option and the URI pattern `redis://user:password@host:port/dbnum`.

```bash
redis-cli -u redis://LJenkins:p%40ssw0rd@redis-16379.hosted.com:16379/0 PING
# PONG
```

#### SSL/TLS
By default, redis-cli uses a plain TCP connection to connect to Redis. You can enable SSL/TLS using the `--tls` option, along with `--cacert` or `--cacertdir` to configure a trusted root certificate bundle or directory.

If the target server requires authentication using a client-side certificate, you can specify a certificate and a corresponding private key using `--cert` and `--key`.

#### Getting Input from Other Programs
There are two ways you can use redis-cli to receive input from other commandsvia the standard input. 

One is to use the target payload as the last argument from from stdin. For example, to set the Redis key `net_services` to the contents of the file `/etc/services` from a local filesystem, use the `-x` option.

```bash
redis-cli -x SET net_services < /etc/services
# OK
redis-cli GETRANGE net_services 0 50
# "#\n# Network services, Internet style\n#\n# Note that "
```

A different approach is to feed redis-cli a sequence of commands written in a text file. The commands in the fle will be executed sequentially by redis-cli as if they were typed by the user in interactive mode. 

Strings can be quoted inside the file so it's possible to have single arguments with spaces, newlines, or other special characters: `SET arg_example "This is a single argument"`

```bash
cat /tmp/commands.txt
# SET item:3374 100
# INCR item:3374
# APPEND item:3374 xxx
# GET item:3374

cat /tmp/commands.txt | redis-cli
# OK
# (integer) 101
# (integer) 6
# "101xxx"
```

#### Continuously Run the Same Command
It's possible to execute a single command a specified number of times with a user-selected pause between executions. This could be useful if you want to monitor some key content of `INFO` field output, or if you want to simulate some recurring write event. 

This is controlled by the `-r` option (how many times to run a command) and the `-i` option (the interval in seconds between commands). By default the interval is set to 0, so commands are executed as soon as possible. The interval can also be specified in milliseconds, such as 0.1 to represent 100 milliseconds.

```bash
redis-cli -r 5 INCR counter_value
# (integer) 1
# (integer) 2
# (integer) 3
# (integer) 4
# (integer) 5
```

To run the same command indefinitely, use -1 as the count value.

#### Mass Insertion of Data Using redis-cli
TODO

#### CSV Output
redis-cli can export data from Redis to an external program using a CSV output feature.

The `--csv` flag will only work on a single command, not the entirety of a DB as an export.

```bash
redis-cli LPUSH mylist a b c d
# (integer) 4
redis-cli --csv LRANGE mylist 0 -1
# "d", "c", "b", "a"
```

#### Running Lua Scripts
From Redis 3.2 onwards, redis-cli has extensive support for using the degbugging facility of Lua scripting.

Even without using the debugger, redis-cli can be used to run scripts from a file as an argument.

```bash
cat /tmp/script.lua
# return redis.call('SET', KEYS[1],ARGV[1])
redis-cli --eval /tmp/script.lua location:hastings:temp , 23
# OK
```

The Redis `EVAL` command takes the list of keys the script uses, and the other non-key arguments, as different arrays. When calling `EVAL` you provide the number of keys as a number.

When calling redis-cli with the `--eval` option, there is no need to specify the number of keys explicitly. Instead, it uses the convention of seperating keys and arguments with a comma.

The `--eval` option is useful when writing simple scripts. For more complex work, the Lua debugger is recommended. The two can be mixed since the debugger can also execute scripts from an external file.

### Interactive Mode





## Securing Redis
By default, Redis binds to all interfaces and has no authentication at all.

The following steps will increase security:
1. make sure the port Redis uses to listen for connections (default 6379, cluster modde 16379, Sentinel 26379) is firewalled, so that it's not possible to contact Redis from the outside world.
2. Use a configuration file where the `bind` directive is set in order to guarantee that Redis listens on only the network interfaces you are using. For example, only the loopback interface, 127.0.0.1, if you are accessing Redis locally from the same computer.
3. Use the `requirepass` option to add an additional layer of security so that clients will be required to authenticate using the `AUTH` command.
4. Use spiped or another SSL tunnelling software to encrypt traffic between Redis servers and Redis clients if your environment requires encryption.

A Redis instance exposed to the internet without any security is very simple to exploit, so make sure to at least apply a firewall layer.





## Using Redis as a Primary Database
### Redis as a Multi-Model DB
A complex application serving millions of users through multiple microservices might need a relational database, full text search, graph database, document database, and caching service.

The challenges of multiple data services include:
- All these data services need to be deployed and maintained
- Your team needs know-how for each data service
- There are different scaling and infrastructure requirements for each data service

You could use the managed data services from Cloud Service Providers, but this could be very expensive as you pay for each service seperately.

Your application code also becomes more complex when having to interact with with different databases, each requiring a seperate connector and logic.

There is also the issue of higher latency because of more network hopping.

As a multi-model database, Redis can resolve most of these challenges:
- You run and maintain just one data service
- This requires only one programmatic interface
- Latency will be reduced by using a single data service endpoint

Redis allows you to store different data structures, while also acting as a cache.

### Redis Modules
Redis Core is a key-value store that supports storing multiple types of data, such as:
- Strings
- Sets
- Bitmaps
- Sorted Sets
- Bit Fields
- Geospatial
- Hashes
- Hyperloglog
- Lists
- Streams

Redis Core can be extended with modules for different data types:
- RediSearch for search functionality
- RedisGraph for graph data storage
- RedisJSON for document data storage
- RedisTimeSeries for time series data

This is modular and you can choose which functionality you want to add.

Since you don't need to implement a caching layer, there is less complexity in your application.

As an in-memory database, Redis is fast and performant making both the application and its test suites faster.

Redis doesn't need a schema and so it doesn't need time to initialize the database and build a schema before running the tests.

## Data Persistence and Durability
Since Redis is an in-memory database, if the server or Redis process fails all the stored data will be lost.

The simplest way to have data backups is to replicate Redis, so if the primary instance goes down, the replicas will still be running.

Redis has multiple methods for persisting data:
1. Snapshots (RDB)
	- Produces single-file point-in-time snapshots of your dataset
	- Can be configured based on time or number of writes passed
	- Data is stored on disk. Great for backup and disatser recovery
	- You may lose the latest minutes of data since the last snapshot