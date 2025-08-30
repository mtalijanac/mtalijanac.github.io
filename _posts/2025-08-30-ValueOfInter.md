---
layout: post
title: Value of Intern
subtitle: There's lots to learn!
cover-img: /assets/img/path.jpg
gh-repo: mtalijanac/associations
tags: [java jmv]
author: Marko Talijanac
---

It isn't an unreasonable assumption that for a class as popular as String, 
every Java developer should be deeply familiar with its inner workings. 
And yet, string interning is — at least from my experience — an unknown mechanism 
to the majority of developers.

So what Intern is? And why should you use it?

To start _intern_ is a [method](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/String.html#intern())
of the String. When used properly, intern lowers memory usage by keeping only one instance of string in memory.
It is a memory utility.

Demonstration:

{% highlight java linenos %}
String s1 = "hello";
String s2 = new String("hello");
System.out.println(s1 == s2); // false, as s1 and s2 are not the same string

s2 = s2.intern();
System.out.println(s1 == s2); // true, as interning s2 returned s1 ref, an first interned value
{% endhighlight %}

Two equal strings are instantiated, but after interning one stays. s1 string is created
as string literal, and thus it is interned by default. But it works for any string, not
only literals:

{% highlight java linenos %}
String s4 = new String("world");
String s5 = new String("world");
System.out.println(s4 == s5); // false, again those two are equal but are not the same

String s6 = s4.intern();
String s7 = s5.intern();
System.out.println(s6 == s7); // true, as their interned values point to same instance
{% endhighlight %}

How does interning work? An internal string pool is prepopulated at program start with all
[string literals](https://www.baeldung.com/java-literals). When _intern()_ is called on a string,
this pool is searched for an equal string. If found, the one from pool is returned. Otherwise,
the current string is added to the pool. Thus, interning is nothing but a cache of strings, 
and method name stands for _"add this string to internal pool"_. 

That's all there is to it.


## How to Intern Like It's 1999

Interning is expensive but, it lowers memory usage as copies of same string can be disposed of.
If a program deals with lots of low-cardinality — a fancy wording for _"repeating values"_ —
string interning is the way to go. This scenario is especially common when unmarshalling data.

Imagine loading users using JDBC:

{% highlight java linenos %}
ResultSet rs = stmt.executeQuery("SELECT name, gender, age FROM users");
while (rs.next()) {
    String name = rs.getString("name");
    String gender = rs.getString("gender");
    Integer age = rs.getInt("age");
    
    System.out.printf("Name: %s, Gender: %s, Age: %d%n", name, gender, age);
}
{% endhighlight %}

There are three fields in the example: *name*, *gender* and *age*. Low-cardinality data can interned. *Name* is obviuosly
opiste of low-cardinality. It is quite unique (there was and never will another "Dudley Pound"), so interning name would
be a memory wasted - or even cause leaks. However _gender_ - that is "male"/"female" or "m"/"f". This field oscilates between
few possible values across all users. Thus, gender can be safley interned to reduce memory usage. Final field - *age* - can't
be interned because Integer lacks this functionality[^1].

Thus the reasonable change to the program would thus be:

{% highlight java linenos %}
String gender = rs.getString("gender").intern();
{% endhighlight %}

In the 1990s, interning strings was common, and every developer knew of it. Back then memory was a scarce
resource. In the 2020s few use it. But Java being Java the method remains.


## Why Bother Then?

If interning is old school why bother with it? Well good ideas rarely go out of fashion.
New technologies often repeate the same old limitations but on a different scale. Yes we have
more memory now, but we also have more data. Thus old memory tricks should still be usefull.
Efficency is a mover here.

One of the common failures of modern software is running code under heavy allocation pressure.
This typically looks like a loop which allocates copius amounts of memory during each iteration,
only to discard it in the next. In such programs, memory usage jumps up and down, up and down like 
on a trampoline. These heavy allocation-deallocation cyles (gc pauses and allocations stalls)
sometimes behave like real trampolines and throw things in [unwanted directions](https://youtu.be/L68zxvl2LPY?t=1850).[^2]

When software expresses this behaviour dominant performance issues usually are:
  - Unmarshalling binary data into Java objects (it takes time and memory, and allocation stalls)
  - Garbage collecting all those objects (it takes time, cpus usage and interuptions)

And these are two sides of the same coin. Remove allocation than gc improves. Gc improves, more cpu 
to allocate faster. Optimize one, and the other improves. In applications like this everything
else almost doesn't matter. A 100,000-line system lives or dies on 1,000 lines of code.
This is a classic performance distribution of source.

Here the path to performance Nirvana is simplicity:
  - Don't unmarshall **unneeded** data
  - Reuse **low-cardinality** data

The first one is common sense. And second one is a cache. The cache of values. Intern cache of values.
Think of it as:

{% highlight java linenos %}
final HashMap<String, String> internMap = new HashMap<>();

String intern(String str) {
    return internMap.computeIfAbsent(str, key -> key);
}

...
// JDBC example becomes:
String gender = rs.getString("gender"); // deserilazie
gender = intern( gender );              // internalize
{% endhighlight %}

To reuse data all it takes is a map lookup for a value equal to its key. And this is not a String only trick 
- it works for all **value types**. We could intern Longs or Integers or any other type. Any objects, as long
- as it doesn't have [_identity_](https://en.wikipedia.org/wiki/Identity_(object-oriented_programming)),
- can be interned by similar map.

This approach has a flaw: we still allocate objects before checking the cache.
To avoid allocations stalls, to truly save memory, we should avoid unmarshalling entirely.
We should reuse data before it is converted to an object!


##  A Better Way to Intern

What we want is to fetch an interned object based on its marshaled representation:

{% highlight java linenos %}
final HashMap<byte[], String> internMap = new HashMap<>();

String intern(byte[] data) {
    return internMap.computeIfAbsent(data, key -> new String(key, StandardCharsets.UTF_8));
}

...
byte[] genderData = rs.getBytes("gender"); // no deserilazion, just read raw byte
String gender = intern( genderData )       // fetch internlized value
{% endhighlight %}

In this variation objects are interned based on their marshalled representation.
And instances are allocated once and only if not found in cache. This is exactly what is needed.

Unfortunately, this code won't work becuse byte arrays have _identity_ [^3] and can't be used as keys in Map.
As far Map is concerned each byte array is a unique instance. You could roll own wrapper around
byte array which fixes identity issues - a wrapper with a proper hashCode and equals. 
However the goal here isn't avoiding ***new String*** but ***new anything***. Any wrapper would thus
require allocation of wrapper, so that is no go.

To trully repair this issue we need a different kind of Map — the one which does not use naive hashCode/equals.
And the great one to use is [UnifiedMapWithHashigStrategy](https://www.javaadvent.com/2023/12/hidden-treasures-of-eclipse-collections-2023-edition.html)
from Eclipse Collections. It is a Map, but the one which allows custom hashCode/equals logic for it keys.
You create a map. You provide HashingStrategy. It works. Use it.


## An Even Better Way

But why stop here? I do like performance quests and this looks like a one. 
Can this be done better? Faster? Stronger?

Let's write out limitations:
  - in this problem byte arrays are short (low cardinality doesn't like LONG names)
  - Map lookup requires a value type using hashCode and equals
  - To test _value_ of byte array means constantly calculating hashCode and equals

That is to say — we have nontrivial compute operation right in the heart of map lookup. 
Map<byte[],String> solution will work — but it won't scale.

So why not avoid all those issues and use a structure suited for this task — a [Trie](https://en.wikipedia.org/wiki/Trie)? 
Trie<Byte> sounds much closer to our problem. If we need to walk byte array (for equality, for hashCode)
why not walk a Trie instead?

And why do we have to walk bytes at all? One long can encode eight bytes. Majority of data arrays 
will be ony a few bytes long. 30 bytes can fit in a four longs with a few bytes to spare. Thus matching 
Trie<Long> would only need to be be four nodes deep.

![Variations of Trie types](ValueOfIntern.DogDot.svg)

Image illustrates these ideas. The first Trie encodes "DOG" by using Strings. To walk that Trie we
need an instance of String. Second one replaces strings at nodes with byte values. Thus we can find
cached instance of "DOG" by walking second Trie using byte representation of word DOG.
The final trie, merges multiple bytes into longs thus not only avoiding string allocation,
but shortening length of walk.

Following these ideas I ended up writing one: [InternTrie](https://github.com/mtalijanac/associations/blob/main/src/main/java/mt/fireworks/pauseless/InternTrie.java). A trie-like structure for interning any **value class**. 

And results? For a small class written in an afternoon — it is undeservingly good.
It is much fastern than _intern()_ itself, and often comparable to the speed of allocation. [^4] It is more
flexibile in useage as you can have as many different intern pools as you would like. And you 
can intern any object type. And all of that is a bonus on top of having it removed about 15% of GC cycle
on a loads which my apps have. 

So what is value of Intern then? As friend of mine said: "It all goes back to a C64 and a need."
[Good ideas](https://en.wikipedia.org/wiki/Flyweight_pattern) don't age...

— Marko Talijanac, April 2025

---

[^1]: In Java Integers in range -128 and 127 are cached, but not using intered mechanism.

[^2]: ZGC is state of the art GC, and expected behavior should be a sub millisecond pause.
      However it is possible to break that behaivor by running heavy allocation load combined with 
      constrained resources.
      While I have not tried ZGC yet, I have tested similar algorithms: Metronome and Shenandoah, 
      and both faced simillar issues. They do work as advertised, but to stay under breaking point
      under the loads seen by my applications, both required significantly higher CPU and memory resources.

[^3]: Arrays in Java do have hashCode and equals, but implementation is based on memory address of array,
      and not on content. Thus two arrays can never be equal. Each one is unique. This 'property' makes them
      unusable for Map keys or any other operation based on equality of data.

[^4]: It dependes on JDK version and time of day, but on my lapotop it is between twice as slow (OpenJDK 8) 
      and twice as fast (OpenJDK 21) for short byte arrays (len <= 7 bytes).
