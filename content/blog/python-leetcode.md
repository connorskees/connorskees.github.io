+++
title = "Using Python for Leetcode"
date = "2024-11-03"
+++

I've done a lot of interviews in a lot of different programming languages. After many years, I've decided that python is the best language for interviewing, and I would even go so far as to say that it's worth learning python even if only for interviewing.

Python is a lingua franca almost on the same tier as JavaScript, but has a much more robust standard library and set of syntactic sugar that makes it far easier to use when solving leetcode/interview problems. Some solutions can be a bit "magic," but in general are more readily understood by interviewers and lend themselves to being elegant and reading pretty close to English.

Below are some assorted tips to make python leetcode solutions nicer.

### Looping

##### Avoid `range(len(arr))`

Python uses exclusively `for .. in` loops, rather than the traditional C-style `for (int i = 0; i < arr.length; i++)` loops. If you have a loop that looks like

```py
for i in range(len(arr)):
    elem = arr[i]
    print(elem)
```

you should replace it with `for elem in arr`.


##### If you _do_ need the index, use `enumerate(..)`

`enumerate` takes a list and returns an iterator of tuples containing the elements alongside their indices. For example,

```py
for idx, elem in enumerate(arr):
    print(elem)
```

Remember the index is always the first element in the tuple. `enumerate` avoids ugly `range(len(..))` calls, and is useful in things like list comprehensions, which we'll discuss below.

##### Iterate in reverse using `reversed(..)`

In a similar vein, you shouldn't use index-based loops to iterate in reverse. Sometimes people try to do `for i in range(len(arr) - 1, -1, -1)`. This is pretty ugly and annoying to type. Instead you should do

```py
for elem in reversed(arr):
    print(elem)
```

### Comprehensions

Python has syntactic sugar for `filter` and `map` in the form of "list comprehensions". Most people writing python are familiar with comprehensions, but that are some interesting applications that some may not know about.

##### Use list comprehensions instead of `map(..)`

A simple example might be to add `1` to every element in `arr`.

```py
arr = [elem + 1 for elem in arr]
```

List comprehensions make mappings over arrays easy to read and write.

##### Use list comprehensions instead of `filter(..)`

List comprehensions also support filtering with the `if` keyword at the end. A simple example to keep only the even numbers in `arr`:

```py
arr = [elem for elem in arr if elem % 2 == 0]
```

Filtering and mapping can be combined in arbitrary ways.

##### Set and dict comprehensions

Comprehensions are not limited to constructing arrays. It's also possible to use them to easily create sets and dicts from existing collections.

For dictionary comprehensions, here's how you might create a dictionary which maps from an element to its index.

```py
elem_to_idx = { elem: idx for idx, elem in enumerate(arr) }
```

Set comprehensions work in much the same way. Here's how you would create a set of just the even numbers in an array:

```py
unique_even_elems = { elem for elem in arr if elem % 2 == 0 }
```

##### You can also create generators from comprehensions

Though generator comprehensions are less useful in leetcode problems. Generators are lazy and will only yield elements when called explicitly. This tends not to matter too much in interviews, but can be useful to 1. save characters when typing and 2. show off your knowledge of python and performance

A generator expression uses parens instead of square brackets. 

```py
elems_generator = (elem for elem in arr)
```

But the parens aren't necessary in contexts where they're already implied, for example as the only argument to a function. This can save typing a few characters for common stuff like `.join`

```py
out = '-'.join(elem for elem in arr if elem % 2 == 0)
```

##### Flattening arrays with nested comprehensions

Flattening in python can actually be pretty annoying. The best way to do this is generally with a nested comprehension. In general the syntax of nested comprehensions can be confusing and easy to typo, so it's preferable to avoid them. But in the case of flattening it's not so bad. A single level flattening would look like:

```py
flat = [elem for arr in nested_arr for elem in arr]
```

### Repeating

##### Repeat strings with multiplication

To repeat the characters of a string `n` times in python, you just multiply the string by `n`. In other languages this is a bit more explicit with methods like `.repeat(..)`.

```py
"a" * 5 => "aaaaa"
```

##### Repeat list contents with multiplication

Lists have a similar behavior, and can be multiplied to duplicate the contents.

```py
["a"] * 5 => ["a", "a", "a", "a", "a"]
```

_Use caution when multiplying lists with objects_. List multiplication does _not_ copy the objects they contain. This means that something like `[[]] * 5` will not create 5 new lists, but rather create 5 new pointers pointing to the same underlying object. This is more clear with an example:

```py
arr = [[]] * 5 # [[], [], [], [], []]
arr[0].append("a")
# arr is now [['a'], ['a'], ['a'], ['a'], ['a']]
```

This behavior only affects objects (so most commonly lists and dicts) and does not apply to things like integers or strings. 

The workaround for this behavior is to use a list comprehension,

```py
arr = [[] for _ in range(5)]
```

### Indexing

##### Slice ranges from lists and strings


This is another pretty basic python feature. You can slice the start and ends of lists and strings using the syntax `[start:end:step]`. All elements are optional. If not specified the values are `0`, `len(s)`, and `1` (no step)

```py
s = "12345"
s[1:3]  # "23"
s[:3]   # "123"
s[1:]   # "2345"
s[::2]  # "135"
s[1::2] # "24"
```

##### Negative indices for indexing the end

Python lists start at `0` and end at `-1`. This means in our string `s` of `"12345"`, `s[0]` is `1` and `s[-1]` is `5`. The negative indices count backwards from there, so `s[-2]` is `4` and so on.

This can make certain algorithms really elegant, though you have to watch out for subtle integer underflow bugs, which in other languages would throw an error for being out of bounds.

##### Negative indices in slicing

You can use negative indices combined with slicing to easily handle behavior at the end of the string.

```py
s = "12345"
s[-1]  # "5"
s[:-1] # "1234"
s[-1:] # "5"
s[-2:] # "45"
```

You can also use negative slicing to iterate in reverse. `s[::-1]` will actually reverse the string to make it `"54321"`.

##### Replacing content by slicing

Slicing in python generally creates a new array which is not related to the old one. For example,

```py
a = [1, 2, 3]
b = a[:]
b[0] = 5
# a => [1, 2, 3]
# b => [5, 2, 3]
```

However, if you slice as part of the left hand side of an assignment, you can replace that slice with a new iterable. A pretty powerful aspect of this functionality is that the iterable being used to replace part of the original array doesn't need to be of the same length.

Here's an example of how you might reverse only the inner part of an array

```py
arr = [1, 2, 3, 4, 5]
arr[1:-1] = reversed(arr[1:-1])
# arr => [1, 4, 3, 2, 5]
```

Or deleting only the inner elements,

```py
arr = [1, 2, 3, 4, 5]
arr[1:-1] = []
# arr => [1, 5]
```

### Sorting

##### Sorting with `sorted(..)`

Sorting in python is super simple, just call `sorted(..)` on the collection and it will return a list with the elements sorted.

##### Sorting by key

You can sort collections with a "key function" using the `key` parameter to `sorted(..)`. The key function just maps each element to a different value which is actually used for comparison. For example, here's how you could use the key function to sort the array such that all of the even numbers are at the start:

```py
arr = [1, 2, 3, 4]
sorted(arr, key=lambda elem: elem % 2) # [2, 4, 1, 3]
```

I also want to give a special shoutout to sorting by len, `sorted(arr, key=len)`, or by other properties which python already has builtin functions for looking up.

##### Sorting in reverse

`sorted(..)` also takes a `reversed` parameter, which can invert the sort. `sorted(arr, reversed=True)`. This can be combined with a key function. 

##### Sorting with a custom comparison function

Please don't sort using a custom comparison function. A custom comparison function is where you define a function that takes in two values and returns `-1`, `0`, or `1` depending on which value is greater. These are generally easy to typo and much more complex than sorting with a key. There is effectively no leetcode question where it would be better to sort this way than with a key.

If absolutely necessary, you can use the `cmp` argument to `sorted(..)`.

### Unpacking and Spreading

##### Iterator unpacking

Python supports list unpacking. This is also called destructuring in other languages like JavaScript. This allows you to remove repeated array lookups like `first = arr[0]`, `second = arr[1]`, etc. It actually works with all iterators, including things like tuples, sets, and generators. However, most commonly you will see this with arrays and tuples.

```py
first, second, third = [1, 2, 3]
```

This will crash if the iterator isn't exactly 3 elements in size. If your array is more than 3 elements and you just want to slice off the first few elements, you can also have a `*arg`.

```py
first, second, *rest = [1, 2, 3, 4, 5]
# first = 1
# second = 2
# rest = [3, 4, 5]
```

##### Variable swapping with unpacking

Unpacking can also be used to swap variables without a temporary variable.

Bad:
```py
tmp = a
a = b
b = tmp
```

Good:
```py
a, b = b, a
```

You can also use this for the pretty common pattern in some leetcode questions of having two variables, `a` and `b`, and needing to set `a` to `b` and `b` to a new value.

```py
def fib(n: int) -> int:
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

This is a pretty simple way to implement computing the nth fibonacci number.

##### Spreading

Python supports first class syntax for spreading iterators into function arguments and other collections. This can also be useful for flattening lists.

```py
a = [1]
[*a, *[2]] # [1, 2]
```

### Collections

Python's standard library of collections is one of the main reasons I like it so much for interviews

##### Collection construction using `set(..)`, `list(..)`, `dict(..)`

This is a bit basic, but I sometimes see solutions which manually `.append(..)` or `.add(..)` inside of a loop to convert from a list to a set or vice-versa. Converting collections in python is really simple,

Bad:
```py
arr = [1, 2, 1]
unique = set()

for elem in arr:
    unique.add(elem)
```

Good:
```py
arr = [1, 2, 1]
unique = set(arr)
```

Dicts can be created in this way, as long as the elements are lists/tuples/other iterators containing exactly two elements. So something like `dict([("key", "value"), ("a", "b")])`. This can be combined with `zip(..)` to easily combine two lists into a key: value dict

```py
keys = [0, 1, 2]
values = ["a", "b", "c"]

combined = dict(zip(keys, values)) # { 0: "a", 1: "b", 2: "c" }
```

##### Union and intersection of sets

Python sets support unioning and intersecting using `|` and `&` respectively. 


```py
set([1]) | set([2]) # { 1, 2 }
set([1]) & set([1, 2]) # { 1 }
```

It's also possible to take the union of two dictionaries this way, though you can't intersect them. If the dictionaries share keys when unioning, the resulting dictionary will have the value of the right hand dict.

##### `defaultdict`

`defaultdict` is a collection from python's `collections` module, and it's a dictionary which has a default value for missing keys. This property can be really helpful in leetcode problems when you want to avoid nesting due to checking the base case.

You set the default value for the keys using a function passed into the constructor.


Bad:
```py
elem_counts = {}

for elem in arr:
    if elem in elem_counts:
        elem_counts[elem] += 1
    else:
        elem_counts[elem] = 1
```

Good:
```py
elem_counts = defaultdict(lambda: 0) # default value of 0

for elem in arr:
    elem_counts[elem] += 1
```

##### `deque`

A less interesting collection, also from the `collections` module. This is a standard double-ended queue and can pop of the front or back with `popleft(..)` and `pop(..)` respectively, with a similar behavior for `.appendleft(..)` and `.append(..)`.

##### `Counter`

`Counter` is another utility collection from the `collections` module. It's a pretty basic structure that's more or less a dictionary which maps elements to their number of occurrences. For example, `Counter("aabc")` would give us `{ "a": 2, "b": 1, "c": 1 }`. `Counter` is a bit cooler than just a dictionary because you can add and subtract them and the counts for individual elements will combine, rather than clobbering each other.

For example, `Counter("aabc") + Counter("abd")` would give us `{ "a": 3, "b": 2, "c": 1, "d": 1 }`.

##### Hashing with tuples

In python, lists and other objects can't be used as keys to hash maps/sets. This can be pretty annoying when trying to deduplicate lists or other common operations involving hash maps.

Tuples, however, _can_ be used as keys to hash maps/sets. Converting a list to a tuple is pretty simple: `tuple(arr)`.

##### Heaps with `heapq`

The `heapq` module allows you to rearrange the elements in an array such that they are ordered like a binary heap. This doesn't change the class of the list, it only changes the order of elements.

To convert to a heap, `heapq.heapify(arr)` will modify the array in place so that it can be used as a heap.

A cool trick is that the smallest value will be at `arr[0]`, so you can peek at this value without popping.

To pop and push, `heapq.heappop(arr)` and `heapq.heappush(arr, elem)`. There's also `heapq.heappushpop(arr, elem)` that's a bit faster if you want to push and then pop at the same, though this is pretty rare.

### Utilities

##### Binary search with `bisect`

Python's standard library contains utilities for doing binary search, so you don't have to worry about implementing it yourself. Binary search can be pretty off-by-one and typo prone, so this is pretty nice.

`bisect.bisect_left(arr, elem)` will return the leftmost index of the elem in the arr if it exists, and otherwise will return the index of where the element should go. There's also `bisect.bisect_right(..)` which will return the position to the right of the rightmost element in the arr if it exists, and otherwise has the same behavior as `bisect_left`. Usually `bisect_left` is what you want.

##### Easy memoization with `@functools.cache` and `@functools.lru_cache`

Caching is a pretty common pattern in dynamic programming problems. You have a function which gets called recusively with some parameters, and a cache hashmap which you insert the parameter and return value into.

`@functools.cache` is a decorator that implements this caching for you in just a single line of code. It works like this:

```py
@functools.cache
def is_even(n: int) -> bool:
    return n % 2 == 0
```

Subsequent calls to the `is_even` function will lookup the `n` parameter in the cache to see if the result has already been computed. 

For some problems, the cache can end up getting really large and causing memory issues. This generally means that your implementation has a bug (try calling the function less, or reducing the number of parameters the function takes in), but it's also one that can often be solved without changing the implementation. Instead of a cache which never evicts keys, you can use an `@functools.lru_cache` to have keys automatically evicted when the cache goes above a certain size.

By default the max cache size is 128, but can be configured to be higher with the `max_size` parameter,

```py
@functools.lru_cache(maxsize=2 ** 9)
def is_even(n: int) -> bool:
    return n % 2 == 0
```

Note that `maxsize` doesn't have an underscore.

You also tend to get bonus points from interviewers for mentioning the potentially bad memory consequences of using `@cache`. 

### Misc Tips

##### Reminding yourself with static types

Modern versions of python support pretty robust optional static typing. These types are ignored at runtime, but can be useful if you're coming from a strongly-typed language for helping to lay out your thoughts. Writing down the function signature and noting that it returns an int or a bool or an array can be really helpful in not getting off track and returning the wrong value. For example if the function asks for the top 2 values, but you misread it and go to return only the max value, the interviewer would be able to stop you when you confirm the return type of `int` rather than `List[int]`/`tuple[int, int]`.

Using static types in python also generally tends to impress interviewers, and demonstrates a good understanding of the language.

##### Sentinel values with `math.inf`

In a lot of algorithms, you start out with the min/max value being the maximum/minimum possible representable value. In python this can be done trivially using `math.inf` and `-math.inf`.