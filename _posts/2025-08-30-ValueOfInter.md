---
author: Marko Talijanac
layout: post
title: The Value of Intern
subtitle: String walks into a jvm...
description: A 1990s memory trick can solve modern performance problems. Stop creating garbage and start interning.
cover-img: /assets/img/ValueOfIntern/path_wide.jpg
thumbnail-img: /assets/img/ValueOfIntern/path_narrow.jpg
share-img: /assets/img/ValueOfIntern/path_narrow.jpg
gh-repo: mtalijanac/associations
tags: [java, jvm]
og_description: "A 1990s memory trick can solve modern performance problems. Stop creating garbage and start interning."
---

It's not an unreasonable assumption that for a class as central as `String`,every Java developer 
should be familiar with its inner workings. And yet, string interning is — at least from my experience — 
an unknown mechanism to the majority of developers.

So what is `intern()`? And why should you use it?

To start, *intern* is a [method](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/String.html#intern()) 
of the String class. When used properly, *intern* lowers memory usage by ensuring only 
one instance of a given string value exists in memory. It is a memory optimization utility.

Demonstration:

{% highlight java linenos %}
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);    // false, as these are equal but distinct objects

String s3 = s1.intern();
String s4 = s2.intern();
System.out.println(s3 == s4);    // true, after interning all references point to the same instance
{% endhighlight %}


Two equal strings are instantiated, but after interning, they reference the same instance. 
It works on string literals too as they are interned by default:

{% highlight java linenos %}
String s1 = "world";             // an example literal
String s2 = new String("world"); // and an equal string
System.out.println(s1 == s2);    // false, as s1 and s2 are different objects

s2 = s2.intern();
System.out.println(s1 == s2);    // true, as interning s2 returned the reference to s1
{% endhighlight %}

How does interning work? An internal string pool is prepopulated at program start with all 
[string literals](https://www.baeldung.com/java-literals). When `intern()` is called on 
a string, this pool is searched for an equal string. If found, the one from the pool is 
returned. Otherwise, the current string is added to the pool. Thus, interning is essentially 
a map/cache of strings, and the method name stands for _"add this string to the internal pool"_.

That's all there is to it.


### How to intern like it's 1999

Interning has a CPU and memory overhead. But used wisely, intern lowers memory usage as copies of the 
equal strings can be garbage collected. If a program deals with lots of low-cardinality data — a fancy 
term for _"repeating values"_ — string interning is a useful technique. This scenario is especially 
common when unmarshaling data.

Imagine loading users using JDBC:

{% highlight java linenos %}
ResultSet rs = stmt.executeQuery("SELECT name, gender, age FROM users");
while (rs.next()) {
    String name = rs.getString("name");
    String gender = rs.getString("gender");
    Integer age = rs.getInt("age");    
    // ... proceed to do something with this fields ...
}
{% endhighlight %}

There are three fields in the example: *name*, *gender*, and *age*. Low-cardinality data is 
a good candidate for interning. Out of those three fields *gender* and *age* are low-cardinality
values. *gender* is typically _male/female/m/f_ — it oscillates between a few possible
values across all users. Because of this, it can be interned to reduce memory usage. *age* is 
another low-cardinality field. In principal it would be great to intern age, but it can't be 
done, as Java's `Integer` lacks this functionality[^1]. There is no intern method on Integer 
to invoke. The final field, *name*, is obviously the opposite of low-cardinality. Names are 
unique or at least unique enough, that interning names would waste memory.


Imagine processing millions of rows in a loop like that. It is common for a night batch job, for example. 
In order to save memory, a reasonable change would be to intern the gender of user:

{% highlight java linenos %}
String gender = rs.getString("gender").intern();
{% endhighlight %}

And that is what programmers did in the 1990s. Interning strings was common, and every developer
knew about it. Back then, memory was a scarce resource. Time flies, and in the 2020s, few use it.
But Java being Java, the method remains.


### Why bother then?

In Java, the recommended approach to object reuse is: "Don't do it! Allocations are cheap, the GC handles
it better than you". Java programmers listen, and often it works. But now and then a Java program eats
CPU like mad and fix is "increase heap size". And now it seems that every Java program needs
gigabyte of heap just to start. Ask your opinionated-non-Java-programmer friend and he will tell you 
harsh words - Java is slow, Java is a memory hog.

A common failure when taking this advice for granted is running code under heavy allocation pressure. 
This typically looks like a loop that allocates copious amounts of memory during each iteration,
only to discard it in the next. In such programs, memory usage jumps up and down like on a trampoline. 
These heavy allocation-deallocation cycles can sometimes behave like real trampolines and throw things in 
[unwanted directions](https://youtu.be/L68zxvl2LPY?t=1850).[^2]

When software behaves this way, the dominant performance issues are:
  - The time and memory cost of unmarshaling binary data into objects
  - The time cost of collecting all that garbage

And these are two sides of the same coin. Avoid allocations, and GC improves. Improve GC, and you free up 
CPU to unmarshal more. Unmarshal more, GC kicks in sooner. Optimise one, and the other reacts. In applications 
like this, almost everything else is irrelevant. A 100,000-line system lives or dies by 1,000 lines of critical code. 
This is a classic distribution of performance sections across source.

The terms to keep track are: *high GC usage* and *allocation stalls*. Both are failure modes, which look like CPU cost.
GC has significant CPU overhead and the best advice is to not create the garbage. While newer 
algorithms like ZGC scale better and have latency boundaries - they aren't free and will eat into CPU 
cycles. Second failure mode is *allocation stall*. It is a form of resource starvation. All allocations 
_hit the memory_ and there is upper hardware limit to how fast can your computer do it. Allocation heavy 
loop will easily hit that limit. Both bottlenecks look like CPU usage and are reduced by polluting less.

Which is a full circle to an old-school wisdom. Old computers were slower, and as consequence old code 
is optimised more. Those guys had good ideas. And good ideas rarely go out of fashion. We have more 
memory now, but we also have more data. Thus, old memory tricks should still be useful. 
Efficiency is the key driver here.

The path to performance Nirvana is simple:
  - Don't unmarshal **unneeded** data
  - Reuse **low-cardinality** data

The first one is common sense - don't do what you don't need. The second one is a cache. An intern cache.
Interning lowers memory pressure. While string interning still allocates memory, it allows GC to reclaim 
newly allocated object sooner. And with generational GCs and [TLABs](https://www.baeldung.com/java-jvm-tlab)
this translates in net performance gains.

Think of intern as string-string map:

{% highlight java linenos %}
final HashMap<String, String> internMap = new HashMap<>();

String intern(String str) {
    return internMap.computeIfAbsent(str, key -> key);
}

...
// The JDBC example becomes:
String gender = rs.getString("gender"); // deserialize bytes to new object
gender = intern( gender );              // replace new object with cached one
{% endhighlight %}

To reuse data, all it takes is a map lookup for a value that is equal to its key. This trick works because
Java's Strings are essentially **value types** - they are immutable and differ only by state. So it
doesn't matter to which "female" instance your gender reference points. And given this fact, all of them 
might as much point to the same. 


### A path to Nirvana

This approach to intern has a performance flaw: we need an instance of a object to invoke intern, and thus 
we need to unmarshal objects *before* hitting the cache. We pay allocations and unmarshaling cost
only to discard object immediately.

Better way to intern would be to hit the cache before unmarshaling, and to unmarshal object only if it isn't
present in cache. This way we can avoid allocation stalls, ignore GC cycles and reuse CPU cycles spent on
unmarshaling objects.

What we want is to fetch an interned object based on its marshaled representation:

{% highlight java linenos %}
final HashMap<byte[], String> internMap = new HashMap<>();  // serialised data to instance mapping

String intern(byte[] data) {
    return internMap.computeIfAbsent(data, key -> new String(key, StandardCharsets.UTF_8));
}

...
byte[] genderData = rs.getBytes("gender"); // no deserialization, just read raw bytes
String gender = intern( genderData );      // fetch internalized value, and deserialise only on first occurrence 
{% endhighlight %}

In this variation of intern, objects are interned based on their marshaled representation (byte array). 
String instances are created once and only if they are not found in the cache. This is exactly what 
we need out better intern implementation.

Unfortunately this wont work. `HashMap` isn't usable here, because byte arrays have _identity_[^3] and can't be used as keys
in a HashMap. As far as the HashMap is concerned, each byte array is a unique instance. You could create your own wrapper around 
a byte array that would fix these identity issues — a wrapper with a proper implementation of `hashCode` and `equals`. 
However, the goal here isn't just avoiding _new String_ but avoiding **new anything**. Any wrapper around 
byte array key would require an allocation of wrapper itself, so this approach is a no-go.

To solve this issue we need a different implementation of `Map` — one that does not rely on naive `hashCode`/`equals`.
A great one is [UnifiedMapWithHashingStrategy](https://www.javaadvent.com/2023/12/hidden-treasures-of-eclipse-collections-2023-edition.html) 
from Eclipse Collections. It is a Map that allows custom `hashCode`/`equals` logic for its keys. 
You create a map, provide a `HashingStrategy` instance and from there everything works[^4].


### The final destination

But why stop here? I enjoy performance quests, and this looks like one. Can this code be done better? Faster? Stronger?

Let's outline the limitations:
  - All byte arrays keys are short (low cardinality usually implies short strings)
  - A Map lookup requires a value type with proper `hashCode` and `equals`
  - Testing the *value* of a byte array means constantly calculating `hashCode` and `equals`

That is to say — byte arrays are mutable and in order to enforce identity maps recalculate hash and equals 
on each operation. Thus we have a nontrivial compute operation right at the heart of the map lookup. 
Thus `Map<byte[],String>` solution will work, but it won't scale well. The way maps and arrays work
in Java, hashCode/equlas would be performance bottleneck.

So why not avoid all those issues in one swoop? If Java maps are a problem we should try using something
which has better semantics for our goal. And a natural structure suited for this task is a 
[Trie](https://en.wikipedia.org/wiki/Trie). If we need to cache a String with itself as a key, 
we should walk a trie instead. It is a natural fit.

And when we are at it - why do we have to walk string characters at all? Why wouldn't we walk bytes of marshaled value? 
Why not build a trie which stores marshaled version of tree but caches unmarshaled object? Thus by using 
`Trie<Byte>` we do not need to unmarshal data into String at all!!

And even that we can further optimise. By switching byte arrays into a long arrays. Long can encode
eight bytes in one fetch. Given that the majority of data arrays will be only be few bytes long 
(common consequence of low cardinality) using long will reduce trie depth, number of nodes and comparisons.
30 bytes can fit into four longs with a few bytes to spare. Thus, matching a `Trie<Long>` for a 30 byte 
object would only need to be four nodes deep.

![Variations of Trie types]({{ '/assets/img/ValueOfIntern/DogDot.svg' | relative_url }}){: .mx-auto.d-block :}

The image illustrates these ideas with tries holding words "DOG" and "DOT". The first trie encodes 
words using Strings. To walk that `Trie<String>`, we need an instance of String and we will walk 
that trie by comparing string characters to nodes of trie. This is classic representation of trie.
It is something what you would see on [Wikipedia article](https://en.wikipedia.org/wiki/Trie) as example.

The second trie, `Trie<Byte>` implementation, replaces characters for byte values, allowing us to 
reach instance of "DOG" by walking structure using the byte representation of the word. This directly 
translates to what unmarshaler does. Put in byte array, fetch a String value. 
Overhead of converting bytes to object is transfered into walking trie.

The final structure is `Trie<Long>` which merges multiple bytes into long. Thus greatly reducing 
length of walk, and improving efficiency compared to bytes version of trie. Besides switching bytes 
for longs it is the same concept as byte version.

And finally this is not a String-only trick — **it works for all value types.** We could intern Longs, 
Integers, or any other type. Any object, as long as it does not have identity, can be interned using 
a similar structure. So intern implemented using these ideas is more efficient than one provided 
by Java and universally applicable across any marshaled data.


### Walking the talk

Following these ideas, I ended up writing such structure:
[InternTrie](https://github.com/mtalijanac/associations/blob/main/src/main/java/mt/fireworks/pauseless/InternTrie.java). 
A trie-like structure for interning any **value class**.

And the results? For a small class written in an afternoon, it is undeservedly good. It is much faster 
than `intern()` itself and often comparable to the speed of allocation.[^5] It is more flexible in 
usage, as you can have as many different intern pools as you like, and you can intern any object type. 
You can even drop a pool when you are done with it. And all of that is just a bonus on top of its 
primary benefit - in my most impacted application it lowered daily count of full GC cycles by about 15%.

So back to title - What is the value of `Intern` then? As a friend of mine said: "It all goes back 
to a C64 and a need." [Good ideas](https://en.wikipedia.org/wiki/Flyweight_pattern) don't age...

— Marko Talijanac, April 2025

---

[^1]: In Java, `Integer` values in the range -128 to 127 are cached, but not using an interning mechanism.
[^2]: ZGC is a state-of-the-art GC, and its expected behavior is sub-millisecond pauses. However, it is possible to break that by running a heavy allocation load combined with constrained resources. It is also CPU heavy.
[^3]: Arrays in Java have `hashCode` and `equals` implementations based on the memory address of the array, not its content. Thus, two arrays with identical content are never considered equal. Each one is unique. This 'property' makes them unusable as Map keys or for any other operation based on data equality.
[^4]: Fundamental problem with `Map<byte[],..>` is that byte arrays can be mutated after being used as keys. So defensive copy is a must.
[^5]: It depends on the JDK version and time of day, but on my laptop, it ranges from twice as slow (OpenJDK 8) to twice as fast (OpenJDK 21) for short byte arrays (length <= 7 bytes).
