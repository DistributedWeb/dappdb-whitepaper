# dAppDB Whitepaper
**Version:** ***1.0***
**Author:** ***Jared Rice Sr.***
**Created On:** ***09-13-2019***
**Last Modified:** ***09-13-2019***

## Abstract
dAppDB is a distributed and decentralized key/value store that utilizes the dDatabase protocol. Keys are path-like strings ```(e.g. /presidents/45/dtrump)``` and values are arbitrary binary data blobs. dAppDB was designed for decentralized applications that require better performance and user-based data control, while maintaining the decentralized, cryptographically secure and immutable features that a blockchain would typically provide. DAppDB is not a dApp's alternative to storing data on a blockchain, rather dAppDB-based databases and blockchains are actually meant to co-exist and empower each other. One dAppDB is capable of holding millions of key/value records with ease, unlike some centralized DBMSs like MongoDB, where collections are limited to 16MB. dAppDB also has an API, that creates a true DBMS-like framework, allowing developers to build an array of different applications on top of it. dAppDB can alleviate a lot of pressure from blockchains like EOS and Ethereum that can only handle so many transactions on a per-second basis, by providing an off-chain solution for the data that derives from a dApp's users, like Swarm and IPFS provide an off-chain file storage solution for a dApp's file storage.

## dAppDB Interface & Behavior
dAppDB is designed to be utilized like a hierarchical filesystem. A value in a dAppDB-based database can be written and read at locations like ```/planets/mars/info``` and the API supports querying or tracking values at subpaths like ```/planets/mars```, which would report changes to ```/planet/mars/info``` and ```/planet/mars/photos/19```. 

A dAppDB-based database can be a single dDatabase feed or multiple dDatabase feeds in a "multi-writer context". Databases created with dAppDB are named, referred to and "revelated" using the public and revelation keys of the dDatabase feed or the original feed if several dDatabase feeds are being used. When using a single-writer context, only a single node that possesses the private key can edit the database (put or delete actions).

A key is allowed to be a UTF-8 string, while a path is separated by a '/'. A key can be both a "path" and a key simultaneously (e.g. /a/b/c and /a/b can both be keys simultaneously). Values can be any binary blob of data, including an empty value (zero length). Example value types including raw unint64 integers (of either endianness), JSON encoded objects, UTF-8 encoded strings or Protobuf messages. As far as metadata, dAppDB-based databases only store the entry's "length" metadata, leaving deserialization and validation to library and application developers. 

### dAppDB Core API
A dAppDB-based database is created by creating or opening a dDatabase feed with dAppDB content. A dAppDB can be managed through four core API calls:
- ```db.put(key, value)``` - Insert value under the path key
- ```db.get(key)``` - Errors when key is non-existent.
- ```db.delete(key)``` - Removes a key from the database. Requires read/write access.
- ```db.list(prefix)``` - Returns a non-nested flattened list of all keys in the database for a specific prefix. 

An example of a pseudo-representation of managing a dAppDB-based database would look like:
```
db.put(`/usa/presidents/dtrump/news`, ``)
db.put(`/usa/presidents/hclinton/`, ``)
db.delete(`/usa/presidents/hclinton/`, ``)
db.put(`/usa/presidents/bclinton/`, ``)
db.get(`/usa/presidents/dtrump/news`, ``)
=>
db.list(`/usa/presidents/`)
=> [`/usa/presidents/dtrump/news`, /usa/presidents/bclinton/`]
```

### Protobuf-Encoded Messages
dAppDB-based dDatabase feeds are made up of two different types of Protobuf-encoded messages. These two types are known as ```Entry``` and ```Inflated Entry```. The initial entry of any dDatabase feed must be a "ddatabase-protocol-header" with ```dataStructureType string dappdb```, so the dDatabase feed uses the dAppDB structure. 

The ```Entry``` and ```Inflated Entry``` protobuf message schematics are as follows:

```
message Entry
message InflatedEntry
required string key = 1;
optional bytes value = 2;
optional bool deleted = 3;
required bytes trie = 4;
repeated uint64 clock = 5;
optional uint64 inflate = 6;
repeated Db dbs = 7;
optional bytes contentDb = 8;
```
In the above schema, there is an optional ```contentDb``` field that is sometimes associated with a primary dDatabase feed. The ```contentDb``` field is used to indicate a feed with data that ultimately needs to be stored outside the limited value size constraint. 

Below is an explanation of the common fields associated with both message types:
- ```key``` - a UTF-8 key, where leading and trailing forward slashes are stripped.
- ```value` `` - arbitrary byte array and is still valid if empty (zero length).
- ```deleted``` - recording the deletion of a key with a "bool" value.
- ```trie``` - structured array of pointers to other ```Entry``` entries  in the dDatabase feed, used for navigating a dDatabase's tree of keys.
- ```clock``` - used in multi-writer mode. In single-writer mode, it is recommended to leave this as an empty list. 
- ```inflate``` - a "pointer that correlates to the most recent ```InflateEntry``` entry in the dDatabase feed (only if dbs and contentDb fields are set. The inflate field is not set in the initial ```InflateEntry``` entry within the feed.
- ```dbs``` - for use in a multi-writer mode. The typical and recommended value in single-writer mode is the dDatabase feed's public key.
- ```contentDbs``` - for use by decentralized applications that require a parallel dDatabase feed for larger data. This field should be used to store the 32-byte public key for that feed. It is important to note that this field is not mutable and can be used in both single and multi-writer modes. 
 
###  Key Path Hashing 
Every key path in the dDatabase feed has a fixed-size hash representation. This hash representation is used by the ```trie```. The concatenation of all the key path-related hashes, outputs what is referred to as a "key path hash array" or KPHA, for the entire database key entry.

**Note:** ***There can be more than one key (string) with the exact same key-derived hash in the same dAppDB-based database, without a single issue. This is because the hash itself points to a linked-list "bucket"  of database "Entries", which are iterated over linearly to find the correct value.***

A path hash is essentially an array of bytes, where elements are 2-bit encoded (values 0,1,2,3), except for a terminating element (4), which has its own value. Each path element in a key path, consists of 32 2-bit encoded values, representing a 64-bit hash of that path element (at 2 bits per value). 

For example ```/usa/presidents``` has two key path segments (usa and presidents) and will be represented by a 65 element path hash array (two 32 element hashes plus a "terminator"). The hashing algorithm used is known as SipHash-2-4 and is implemented via the libsodium library's crypto_shorthash() function. It has an 8-byte output and a 16-byte key. The input is the UTF-8 encoded key path segment, without any slashes, separators or "terminating" bytes aka "null bytes". A 16-byte "secret" key is required where in this particular case, zeros will be used. 

When converting the 8-byte hash to an array of 2-bit bytes, the ordering is byte-by-byte, where each of the two lowest value bits are taken (hash & 0x3) as byte index 0, the next two bits (hash & 0xC) as byte index 1 and so on. 

For example, the key /a/b/c converts into a 97-byte hash array (32+32+32+1 terminating null byte):
````[1,2,0,1,2,0,2,2,3,0,1,2,1,3,0,3,0,0,2,1,0,2,0,0,2,0,0,3,2,1,1,2,0,1,2,3,2,2,2,0,3,1,1,3,0,3,1,3,0,1,0,1,3,2,0,2,2,3,2,2,3,3,2,3,0,1,1,0,1,2,3,2,2,2,0,0,3,1,2,1,3,3,3,3,3,3,0,3,3,2,3,2,3,0,1,0,4]
```

The terminator null byte (4) always has the highest bit index. "Hashing collisions" are rare in this hashing schema, although this does happen with large dAppDB-based databases that contain millions of keys within a feed. 

### Trie's With An Incremental Index
A ```prefix trie``` is stored within each node, for the purpose of looking up other keys or filtering a database for a set of keys that match a specific prefix. This is always stored in the ```trie``` filed of the ```Entry``` type, within the Protobuf message. The ```trie``` is simply a mirror of the KPHA, where each element within a ```trie``` is known as a ```bucket```. 

A few specifications as it applies to trie buckets:
- Every trie bucket that contains anything other than a zero-length value (non-empty), points to the newest ```Entries``` that posses a path that is considered to be identical up to that precise prefix location. Because the trie has 4 "values" at each node, pointers can exist for up to 3 other "values" at any element in the ```trie array```. 
- Empty trie buckets are technically possible, only if there aren't any "branches" at this node in the trie.
- Only non-null elements are stored on disk. 
    
A trie's data structure is essentially made up of a thin array of pointers to other ```Entry``` entries.

## dAppDB Protocol
Below, I will explain the protocol behind dAppDB and how it works step-by-step, from key lookups, to key writes and key deletes.

### Looking Up Keys In A Database
To lookup a key in a dAppDB-based database, you have to:
1. Calculate the KPHA for the key you are looking up.
2. Perform a comparison of the KPHAs. If the paths are an exact match, then compare keys. If keys are an exact match, your lookup was successful. 
3. Check if the deleted flag was set, if so, this ```Entry``` represents that the key was deleted from the databases.
4. If the paths match, but the keys don't, search the last trie array index for a pointer and iterate from the third step with the new ```Entry```.
5.  If the paths do not have an exact match, look for the 1st index where there are two differing arrays and attempt to find a corresponding element in this ```Entry's``` trie array.
6. If this element is empty and there isn't a pointer matching the 2-bit value, then the key we're looking up doesn't exist. 
7. If the trie element has more than a zero length, then follow that pointer to select the next ```Entry```. Continue this process from the third step. Throughout this process, you descend the trie in a lookup and will either have an unsuccessful lookup for the particular ```Entry``` being search for, or it will be determined that the key being looked up simply doesn't exist in the database.

### Writing A Key To a Database
The process of writing a key to a dAppDB-based database is similar to the process of looking up keys in a database:
1. Calculate  the key's KPHA, starting with an empty ```trie``` of the same length. You will then write to the trie array from the current index (0).
2. Select the "latest" dDatabase feed ```Entry```. 
3. Perform a comparison of the KPHA's. If paths and keys are an exact match, then a dDatabase feed ```Entry``` already exists and you are performing an overwrite of the "current" ```Entry``` and CAN duplicate the "remainder" of its trie, up to the "current" index ```trie```.
4. If a path is an exact match but the keys are not an exact match, you are adding a new key to a bucket that is already existent. Duplicate the trie and extend the trie to its full length and at the final array index, add a pointer to the ```Entry```, that has the same hash.
5. If there isn't an exact match with the path(s), try to locate the first index where there are two differing arrays. Duplicate all the entries with the ```trie```, in to a new trie for indices  between the current index and the differing index.
6.  Search for the matching element within a dDatabase's ```Entry``` trie array, at the differing index. If the element is empty, you have found the most similar ```Entry```. A pointer is written to this node, of the trie at the differing index and the write process is done. Any ```trie``` elements that remain can be omitted due to them being empty. 

### Deleting A Key 
To delete a key value, we follow the same exact process as adding a key to a database and write an ```Entry``` setting the delete flag. Deletion nodes persist in a dAppDB-based database for the lifetime of the database.

## Binary Trie Encoding
dAppDB uses what is known as "Binary Trie Encoding", which is a scheme for encoding trie data structures. 

This uses the following scheme:
````
- trie index (varint)
- bucket bitfield (packed in a varint)
- feed index (varint, with an extra low bit)
- entry index (varint)

## Directed Acyclic Graph
When all of the dAppsDB's members, combine all the performed operations, it creates a directed acyclic graph (DAG). Each write to the database (setting a key to a value), which includes information that points backward to all the known "heads" in the graph. 

A pseudo-representation of this, if Alice was starting a new dAppDB by adding 2 values to it, would look like this:
//dDatabase Feed
0 (/usa/presidents = `dtrump`)
1 (/usa/states = `texas`)
//Graph
Alice: 0 ` + nodes[0].value)
})
})

## JavaScript API (Reference Implementation)
Below is a reference implementation created in JavaScript, that includes a low-level API for creating a dAppDB-based database within Node.Js-based decentralized applications. 

### Creating a new dAppDB-based database
```var db = dappdb(storage, [key], [options])```
```storage``` - can be a string or a function. If a string is used, like in the above example, dWeb's core ```dwraf``` module is used (random-access-file) storage module is used. The resulting folder with the data will be whatever storage is set to. If ```storage``` is a function it will be called with every filename the database needs to operate on. 

#### Random-Access Storage
There are many providers for the random-access interface:

```
var ram = require(`@dwcore/dwram`)
var feed = dappdb(function (filename) )
```

```key``` - a Buffer containing the local dDatabase feed's public key. If you do not set this to the public key, it will be loaded from storage. If no public key exists, a new keypair will be generated.

#### API Reference
```db.key``` - Buffer containing the public key identifying this dAppDB-based database. Populate after ```ready``` has been emitted. May be null before the event.

```db.revelationKey``` - Buffer containing a key derived from the db.key. In contract to db.key, this key does not allow you to verify the data but can be used to announce or look for dWeb peers that are sharing the identical dAppDB-based database, without leaking the database's key. Populated after ```ready``` has been emitted. 

```db.on(`ready`)``` - Emitted only once, when the database is fully ready and all static properties have been set. You do not need to wait for this when calling any async functions.

```db.version(callback)``` - Get the current version of the identifier, as a buffer for the database.

```db.checkout(version)``` - Checkout the database at an older version. The checkout is a dAppDB-based database instance as well. Version should be a version identifier returned by the ```db.version``` API call or an array of nodes returned from ```db.heads```.

```db.put(key, value, [callback])``` - Insert a new value. Will merge any previous values seen for this key.
```db.get(key, callback)``` - Lookup a string key. Returns a node's array with the current values for this key. I there is no current conflicts for this key, the array will only contain a single node.
```db.del(key, callback)``` - Deletes the string key. 
```db.batch(batch, [callback])``` - Efficiently insert a batch of values, in a single atomic transaction. A batch should be an array of objects.
```db.local``` - Your local writable dDatabase feed. You have to get an owner of the dAppDB to authorize you to have your writes replicated. The creator of a dAppDB-based database is the first owner. 
```db.authorize(key, [callback])``` - The authorization of a dWeb peer to write to a dAppDB-based database.
```db.authorized(key, [callback])```- A method for checking if a key has the necessary authorization to write to the database. 

#### Example of db.authorized
```myDb.authorized(otherDb.local.key, function (err, auth) )```

#### Watcher
Easily watch a folder and get a notification anytime a key inside this folder has changed. For example:
```
db.watch(`usa/presidents`, function() )
...
db.put(`usa/presidents/dtrump`, `45th`) // triggers the above
```
You can destroy the watcher by calling ```watcher.destroy()```. The watcher will emit watching when it starts watching and change when a change has been detected. If a critical error occurs, an error will be emitted on the watcher.

#### Readable Stream Of Nodes
```var stream = db.createReadStream(prefix[options])``` - Create a readable stream of nodes stored in the database. Set prefix to only iterate nodes prefixed with that folder.

```var stream = db.createWriteStream()``` - Creates a writable stream. Where stream.write(data) accepts data as an object or an array of objects with the same form as db.batch().

```db.list(prefix[, options], callback)``` - Same as ```createReadStream``` but buffers the result to a list that is passed to the callback. 

```var stream = db.createDiffStream(prefix[, checkout)``` - Find out about changes in key/value pairs between the version checkout and the current version preference. 

## Conclusion 
dAppDB allows for developers of decentralized applications to create dApps that operate at global-scale, by placing the data that derives from the user's of their applications off-chain, rather than creating millions of event-based transactions on a blockchain, that can only handle so many transactions per-second. It is clear that with blockchains like EOS, that currently have the capacity for 5,000 transactions per-second, that blockchains are far from giving an entire world of dApps, the ability to all operate at global-scale. dWeb's peer-to-peer network, is able to operate at global scale, because peers only consume and distribute the data they want to. dApps built on blockchains like Ethereum and EOS, suffer the effects caused by other dApps that use the same network, while a dApp on dWeb does not suffer the effects from other dApps on the dWeb. dApps built entirely around a blockchain and services like IPFS, may be entirely decentralized but have growth bottlenecks when it comes to serving billions of users. dAppDB allows decentralized application developers to bring an infinite amount of off-chain data-based events and high-performance data querying into their applications, while maintaining their decentralization.
