---
title: Depth of module
description: >-
  This article summarizes the concept of deep module described in the book “A
  Philosophy of Software Design” written by Professor John…
date: '2019-01-17T11:33:43.062Z'
tags: ["Software Design", "Philosophy"]
---

Click [here](http://nakawatch.hatenablog.com/entry/module-depth) for Japanese version

This article summarizes the concept of **deep module** described in the book “[A Philosophy of Software Design](https://www.amazon.com/t/dp/1732102201)” written by [Professor John Ousterhout](https://web.stanford.edu/~ouster/cgi-bin/home.php) at Stanford University, and is written with permission of him. However, this is mainly describing my opinion rather than giving detailed contents of this book.

{{< tweet 1084303975694757889 >}}
{{< tweet 1084606003922984960 >}}

First, I will define a good module I consider and then walk you through the concept of deep module to realize it.

### What’s a good module

What’s a good module? The term module here means classes, structures, subsystems and divided software such as Microservice etc, and language does not matter.

#### A small module?

Small is worth, as is the Unix philosophy of “Small is beautiful”. Dividing the module into small pieces makes it easier to clarify the work of each module, which makes maintenance easier as a result. Certainly it is important, but in the first place, why make a module?  
One of the main reasons is to lower the complexity of implementation by gathering elements of high relevance. See this code on the assumption that it.  
In order to open a file in Java, you need three small classes as follows.

```java
FileInputStream fileStream = new FileInputStream(fileName);
BufferedInputStream bufferedStream = new BufferedInputStream(fileStream);
ObjectInputStream objectStream = new ObjectInputStream(bufferedStream);
```

[FileInputStream](https://docs.oracle.com/javase/8/docs/api/java/io/FileInputStream.html) provides rudimentary I/O only, and [BufferedInputStream](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedInputStream.html) converts the byte input stream obtained by FileInputStream into a byte input stream with buffering read. And [ObjectInputStream](https://docs.oracle.com/javase/8/docs/api/java/io/ObjectInputStream.html) provides the ability to operate serialized objects.  
As you can see, we have to generate as many as three objects just to open a file. By making the class smaller, the work of each class became clear, but because the relevant FileInputStream and BufferedInputStream are independent, if a user forgets to create a BufferedInputStream, it will not be buffered and will be slow. The point is, the complexity of implementation is leaking to the outside. It’s not always bad that the complexity of implementation leaks out, but you should be aware of the following.

*   The fact that modules are small means that the amount of information they have is small
*   If the amount of information that it has is small, the complexity of implementation tends to leak outside
*   If the complexity of implementation is leaking to the outside, it is easier for module implementation changes to affect the surrounding modules

There is no doubt that small is beautiful, but it turned out that maintainability declined if dividing highly relevant elements. Thus, it turned out realized that you could not make a good module if you were too conscious of making it smaller.

#### High Cohesion? Loose Coupling?

It is essential that elements of high relevance gather (high cohesion), and each doesn’t depend on (loose coupling). The above code is a beautiful example of loose cohesion and tight coupling. There are many benefits to having hidden many elements of high relevance.  
Having elements of “high relevance” makes the design clearer, easier to maintain and extend, and increases reusability.  
Also, having “many” elements strengthens the functionality of modules.  
In addition, by having “hidden” it, you can promote loose coupling.  
Therefore, it turned out that high cohesion is a very important concept.

It goes without saying that the loose coupling of modules will have a positive impact on software, but in order to maintain loose coupling, it is necessary ***to make interfaces connecting the modules good***.

#### What’s a good interface?

People often say that “keep interface simple”, but the word “simple” is ambiguous, so it should be embodied.  
First of all it’s necessary to understand what is required of an interface of a module. For modules to use other modules, they sign a contract named interface. Since each module is implemented on the premise of that contract, as the contract changes, the implementation of all modules involved in the contract must be changed. However, software is quite likely to evolve. Interfaces are required to not collapse even in this contradiction, in other words, **I’m sure that a good interface is an immutable interface (it does not allow changes other than addition)**.

And it should be easy to use for users who use an interface whenever possible. But my thought on this “ease of use” is influenced by the way of thinking that Clojure author Rich Hickey mentioned in [Simplicity Matters](http://confreaks.tv/videos/railsconf2012-keynote-simplicity-matters). I consider that ease of use is a relative concept, and what is useful depends on the user.  
Users are roughly divided into two in terms of use cases, some people use in the edge case, others use in the general case. Allowing edge cases is one of the main reasons that the interface becomes complicated. For that reason, ***you should focus on general use cases and pursue ease of use***.  
Please see at the Java code above again. It’s a general usage method to use buffering when operating files. Nonetheless, it allowed the edge case that it doesn’t use buffering by allowing the BufferedInputStream class to be independent. As a result, the interface has become complicated.  
I again define a good interface as “an easy-to-use in a general case and immutable interface”.

#### A good module

Based on the above, the definition of a good module I consider is **“a module that hides many elements of high relevance and provides powerful functionalities through an easy-to-use in a general case and immutable interface”**. In the next chapter, I will walk you through the concept of deep module that realizes it.

### deep module

Implementation becomes complicated to some extent due to hide many elements, yet in order to become an ideal module, it must provide powerful functionality through interfaces that hide the complexity of implementation. Professor John Ousterhout calls such a module deep module. It is written as follows in this book.

> “they allow a lot of functionality to be accessed through a simple interface.”
>
> (John Ousterhout, A Philosophy of [Software Design](http://d.hatena.ne.jp/keyword/Software%20Design).)

{{< figure src="https://cdn-images-1.medium.com/max/800/1*lCGfGhyIOUg12Bhis-iLQA.png" width="100%" height="auto">}}

The above figure was created with reference to the figure depicted in this book. The rectangle represents a module, the length of the top edge represents the complexity of the interface, and the vertical length represents the powerfulness of the functionality (benefits the module brings). That means the longer it’s horizontally the more complicated the interface, and the longer it’s vertically the more powerful the functionality. Professor John Ousterhout uses the term “deep” to describe modules such as the portrait rectangle on the left side, and thinks it is the best.  
It indicates that the following viewpoints are necessary for module design.

*   The functionalities are powerful, but are the interfaces more complicated than that?
*   The interfaces are simple, but are the functionalities poorer than that?

***Interface complexity is cost, and powerfulness of functionality is benefit***. It’s a deep module if the benefit is above the cost, yet it’s hard to identify this, so you should see many good examples.

#### e.g.) File I/O provided by Unix OS

Specific examples of embodying deep module include system calls for I/O provided by the Unix OS. Basically, there are only five system calls if you follow the standard file I/O model.

*   open()
*   read()
*   write()
*   close()
*   lseek()

Despite the fact that the implementations of Unix I/O have evolved over the years, these five system calls have not changed. This is an ideal deep module. Since the kernel was originally designed to run the user application stably, the interface is hardly changed. Therefore, I’m convinced that seeing system calls is very helpful.

### Conclusion

Module designers are required to design with a lot of perspectives(e.g. cohesion, coupling, reversibility, orthogonality).  
However, I suppose everyone feels that it’s hard to incorporate all perspectives into software which expresses abstract world with concrete code and changes constantly.  
So, **why not try asking a single question “Is this module deep?”**.  
I would be grateful if as many people as possible could grasp Professor John Ousterhout ‘s wonderful concept deep module.  
You should purchase [this book](https://www.amazon.com/t/dp/1732102201) if you’re intrigued.
