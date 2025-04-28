# Trash Talk: Understanding Python’s Garbage Collector

Check this post on my blog [here](https://hevalhazalkurt.com/blog/trash-talk-understanding-pythons-garbage-collector/).

<br>

When you write a Python program, you probably don’t spend much time thinking about what happens to all the stuff you stop using. You create variables, objects, data structures... and when you’re done with them, you just move on. 

But wait, under the hood who’s cleaning up all the mess you leave behind? The answer is Python’s garbage collector.

Let’s have a real talk about what it is, how it works, and why sometimes you might want to give it a little extra attention.

<br>

## So... What exactly is the garbage collector?

Imagine you’re cooking dinner. You chop vegetables, unwrap packages, use up ingredients, and now your kitchen is a total mess. Somebody has to clean it up, or eventually you’ll run out of counter space.

In Python, the garbage collector (GC) is like your silent kitchen helper. It quietly works in the background, cleaning up all the “garbage”, bits of memory you’re not using anymore. When your program creates objects like a list, a string, a dictionary, Python stores them in memory. When you’re done with them, when there are no more references to them, Python’s GC steps in and tosses them out. You don’t have to tell Python, “Hey, I’m done with this list, you can delete it now.” It just knows.

<br>

## How does it know what to clean?

Python mainly uses something called **reference counting**.

Here’s the idea:

- Every object keeps track of how many references or pointers are still pointing at it.
- If a variable is still using an object, the reference count stays above 0.
- When the reference count drops to 0, Python knows it can safely throw the object away.

Example:

```python
a = [1, 2, 3]   # a list is created, reference count = 1
b = a           # another reference to the same list, count = 2
del a           # remove one reference, count = 1
del b           # remove the last reference, count = 0 → Garbage collected!
```

Pretty smart, right?

<br>

## Is reference counting enough?

Nope, here’s where it gets spicy. What if two objects are referencing each other, but nothing else is using them? This is called a **circular reference**.

Example:

```python
class MyObj:
    def __init__(self):
        self.ref = None

a = MyObj()
b = MyObj()

a.ref = b
b.ref = a
```

Now `a` points to `b`, and `b` points back to `a`. Even if we delete both `a` and `b`, they’re still hanging onto each other. Their reference count never drops to zero. That’s why Python also has a full-blown garbage collector. It can look deeper, spot these "stuck together" objects, and clean them up too.

<br>


## Digging Deep → Generational Garbage Collection

Python uses a clever trick called generational garbage collection to make things fast.

Here’s the basic idea:

- Most objects die young.
- New objects are checked often.
- Older objects are checked less often, because if they survived a while, they’re probably important.

Python sorts objects into three **generations**:

1. **Generation 0**: Brand new babies (checked the most often)
2. **Generation 1**: Objects that survived a bit
3. **Generation 2**: Old timers (checked way less often)

This system saves time because Python doesn’t waste energy checking the same objects again and again.

<br>

## How You Can Actually Use GC

Can you just ignore the garbage collector? 99% of the time, yes. Normally, you don’t have to worry about any of this. Python handles it all beautifully. But sometimes, especially in big, complex programs like web apps or data science projects, you might want to peek under the hood. Here’s how:

Manually trigger garbage collection if you need to free up memory at a specific time:

```python
import gc
gc.collect()
```

<br>

Disable the garbage collector temporarily if you know it’s getting in your way. For example when you create tons of short-lived objects quickly.

```python
gc.disable()
# your code flow here
gc.enable()
```

Track down memory leaks by using tools like `gc.get_objects()` to see what’s hanging around when it shouldn’t be.

<br>

## Finally

Python’s garbage collector is like that good friend who always cleans up after the party without being asked. You don’t notice it, but life would be way harder without it. By understanding just a little about how it works, you can write smarter programs, solve weird bugs faster, and make your apps even better.