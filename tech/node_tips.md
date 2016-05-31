# A list of stuff to check for when using Node

## Table of content

1- [Strict Mode](#1--strict-mode)
2- [Errors](#2--errors)
3- [I/O](#3--io)
4- [Application Setup](#4--application-setup)
5- [Machine Setup](#5--machine-setup)
6- [Socket Optimization](#6--socket-optimization)

---

## 1- Strict Mode

Node is compiled Just-in-Time and unhandled exceptions cause the process to exit. Typing errors, type coercion, lost scopes and general bad practice make it hard to catch all errors during dev.

### Alternatives and solutions

This isn't a [performance boost](http://stackoverflow.com/questions/3145966/is-strict-mode-more-performant) per se, but it is always good to add `'use strict';` at the top of your node files. This tells the compiler to be more strict about the kind of instructions you use. Only a very small number of instructions will be optimized in this mode, the real gain is the number of pre-runtime warnings you will get. These can help you flag bugs, potential mistakes and unoptimized sections in your app before things break in production!

[top](#table-of-content)

---

## 2- Errors

Instantiating an `Error` Object eats up a lot of resources, namely because v8 will collect the stack info on creation.
It takes around 71ms to create 10 000 instances on an average machine.

Secondly, accessing the `'stack'` property of an `Error` Object will be painfully slow. It takes around 621ms to create and read the stack of 10 000 instances. 

This grows depending on the length of the stack as v8 will try to resolve all the entries from it's internal system and format it as a `String`.

It's worth noting that older versions of node (v0.12 and releases prior to iojs) have a *much* greater delay. In fact, accessing the `'stack'` property is known to sometimes lock the process for up to [100ms](https://groups.google.com/forum/#!searchin/nodejs/stack$20slow/nodejs/-U2hIDWcc30/5WRuCeoA8HgJ)  

### Alternatives and solutions

- Use a custom Error system with hand-made classes - if you don't need the exact stack or the features of the `Error` class.

  Ex: 
  
    ```
    function MyCustomError(msg, data) {
        this.message = mgs;
        this.data = data;
        
        // Don't extend Error
    }
    
    if (error) throw new MyCustomError("Something happened");
    ```
    

- Use the Error.stackTraceLimit property of the `Error` class to limit the number of parsed stack traces on accessing `'stack'`.
  Default value is 10.
  
  Changing it to 1 made the stack request drop from 621ms to about 310ms.

  Ex:

    `Error.stackTraceLimit = 4;  // A pretty good starting point`
    
[top](#table-of-content)

---
    
## 3- I/O

Writing or reading to disk and sending network requests are some of the most expensive types of operation.
[Latency diagram](https://gist.github.com/jboner/2841832)

### Alternatives and solutions

- Bundle I/O calls

Imagine network operations as some sort of ferry service. You need to move people and cars from one side to the other.
Round-trips are slightly slower when you put more cars in, but it's nothing compared to the time required to go travel a full back and forth.

The same goes for Database queries. It's [much faster](https://shyp.github.io/2015/07/13/speed-up-your-javascript-tests.html) to bundle DB calls.

- Cache everything

In-memory cache is extremely effcicent - when well used. You can either setup a small [node-cache](https://github.com/ptarjan/node-cache) or put [ngnix](https://www.nginx.com/) or similar in front of your service (for static content).

- Use process messaging

Communicate directly with other Node processes, when possible or use non-ip protocols like [ipc](https://github.com/fed135/ipc-light)

[top](#table-of-content)

---

## 4- Application setup

Node is extremely lightweight, but it is single-threaded and not designed for CPU-intensive or CPU-blocking tasks.

### Alternatives and solutions

- Services

Design small, non-blocking and stateless services that you can easily cluster.

- Leverage services and node asynchronicity

For complex or CPU-intensive tasks you may be better off leveraging an external application doing that specifically (written in the language of your choice) and communicate with that process with node via events, thus not blocking the event-loop.

[top](#table-of-content)

---

## 5- Machine setup

Most machines have system configurations that take in account the most common use cases. Disks configured for file-reading, sane ammount of parallel process limit, etc. 

### Alternatives and solutions

- Readahead:
    
Readahead is the size of the chunks that are read from storage at a time - before being processed. Mongo server configurations [strongly suggest](https://docs.mongodb.org/manual/administration/production-notes/) reducing that number to better match the size of your packets. Let's say your application sends a lot of small packets (through ipc) or reads tiny files from disk, you should drop that ammount considerably. In average, going from 4kb to 1kb will gain you considerable faster communication speed.

Continuing on that note, having faster hardware will also impact ipc perfomances. Consider buying ssd disks for your stack.

- Connection limit

The number of concurrent connections is limited in most linux distros to 1024^2. This can be [fixed](https://mrotaru.wordpress.com/2013/10/10/scaling-to-12-million-concurrent-connections-how-migratorydata-did-it/)

- File descriptors

Finally, you will need to increase the number of allowed [file descriptors](http://www.cyberciti.biz/faq/linux-increase-the-maximum-number-of-open-files/) on your system. 

[top](#table-of-content)

---

## 6 - Socket Optimization

Socket optimizing depends on the type of work you are accomplishing. It can be a mix of multiple - or none- of these strategies.
All of these can be leveraged through [Kalm](https://github.com/fed135/Kalm)

### Alternatives and solutions

- Naggling 

Naggling is the concept of bundling I/O calls to save on protocol overhead. It is mostly effective when you have lots of smaller packets to send at once.

- Compression 

You can "zip" payloads as they are essentially binary data in the form of a buffer. This is very effective for larger data structures that would otherwise be bigger than the MTU (Max Transfer Unit), therefore causing additional roundtrips. There are many libs to accomplish this. The Node built-in one `zlib` is slow, but gives a very good result. [snappy](https://github.com/kesla/node-snappy) is much faster for compressing and decompressing, but doesn't yield as good results.

- Binary JSON

Instead of stringifying Objects before putting them inside Buffers, wouldn't it be more efficient to somehow store the content of the Object directly? (Not the String representation) This is what the [msgpack](http://msgpack.org/index.html) protocol attempts to do. Unfortunatly today, there are still no node implementations that can come close to beating the speed of stringification. Are the gains in terms of size are good, but depending on your setup it might not really worth all the extra CPU cycles.  

- Protocol Buffers

This [Google protocol](https://developers.google.com/protocol-buffers/) is sort-of like binary JSON, but it requires a Schema specification for your data, which means that you can't just send anything and won't work with polymorphic payloads. The speed and size greatly make up for it though. It is faster than stringification and implements a byte storing mechanism that removes the need to add compression over it. This is the god of all protocols, if you are willin to Schematize all of your Socket exchanges (on both ends).

- Timeout

It is very important to manage your connection resources closely. This means adding `socket.setTimeout()` and listening for the `timeout` event to close the socket. Typically, 30 seconds is a pretty decent timeout. You may have to implement a reconnection behavior.

[top](#table-of-content)
