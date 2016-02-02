# A list of sweet v8/Node Optimizations
**And stuff to look out for**

**And solutions, to most issues**

[iterative process] - collaborators welcomed!

## 1- Strict Mode

A quick and easy way to get a small [performance boost](http://stackoverflow.com/questions/3145966/is-strict-mode-more-performant) is to add `'use strict';` at the top of your node files. This tells the compiler to be more strict about the kind of instructions you use. The main gain not being performance as such, since only a very small number of instructions will be optimized in this mode, but rather the number of runtime warnings you will get. These can help you flag bugs, potential mistakes and unoptimized sections in your app.


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