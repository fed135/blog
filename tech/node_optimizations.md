# A list of sweet v8/Node Optimizations

- Stuff to look out for
- Setup 
- Architecture
- Best practices
- etc.

[iterative process] - collaborators welcomed!

---

## 1- Strict Mode

A quick and easy way to get a small [performance boost](http://stackoverflow.com/questions/3145966/is-strict-mode-more-performant) is to add `'use strict';` at the top of your node files. This tells the compiler to be more strict about the kind of instructions you use. The main gain not being performance as such, since only a very small number of instructions will be optimized in this mode, but rather the number of runtime warnings you will get. These can help you flag bugs, potential mistakes and unoptimized sections in your app.

---

## 2- Errors

    (new Error)

Instantiating an `Error` Object eats up a lot of resources, namely because v8 will collect the stack info on creation.
It takes around 71ms to create 10 000 instances on an average machine.


    (new Error).stack

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

In-memory cache is extremely effcicent - when well used. You can either setup a small [node-cache](https://github.com/ptarjan/node-cache) or put [ngnix](https://www.nginx.com/) or similar in front of your service.

---

## 4- Application setup

Node is extremely lightweight. There little to none overhead between I/O and disk operations - which it is bound by. Node is single-threaded and I/O instructions are queued in the system event-loop. Therefore, a good way to get more bang for your buck is - and this is my opinion only - small, non-blocking, stateless services that you can easily cluster. Avoid network communications when available with process messaging, messaging queues or [ipc](https://github.com/fed135/ipc-light).

---

## 5- Machine setup

Most machines have system configurations that take in account the most common use cases. Disks configured for file-reading, sane ammount of parallel process limit, etc. 

- Readahead:
    
Readahead is the size of the chunks that are read from storage at a time - before being processed. Mongo server configurations [strongly suggest](https://docs.mongodb.org/manual/administration/production-notes/) reducing that number to better match the size of your packets. Let's say your application sends a lot of small packets (through ipc) or reads tiny files from disk, you should drop that ammount considerably. In average, going from 4kb to 1kb will gain you considerable faster communication speed.

Continuing on that note, having faster hardware will also impact ipc perfomances. Consider buying ssd disks for your stack.

- Connection limit

The number of concurrent connections is limited in most linux distros to 1024^2. This can be [fixed](https://mrotaru.wordpress.com/2013/10/10/scaling-to-12-million-concurrent-connections-how-migratorydata-did-it/)

- File descriptors

Finally, you will need to increase the number of allowed [file descriptors](http://www.cyberciti.biz/faq/linux-increase-the-maximum-number-of-open-files/) on your system. 


