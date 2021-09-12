# 通过实现链表学Rust

> 有问题或者想要一次性查看完整的代码？
> [GitHub上应有尽有][github]

> **注意**：本书的当前版本是基于Rust 2018的实现，
> 这个版本在2018年的12月8日发布，如果你的rust工具链足够新的话，才执行`cargo new`创建
> 项目之后，在Cargo.toml文件中会包含一行`edition = "2018"`（如果在很久之后阅读这本书，
> 这个数字可能会更大）。使用老版本的工具链也是可以的，但是可能会在编译的时候遇到一些书中没
> 有提到的编译器错误。这样听起来也挺有趣的！^_^

我经常被问到怎么用Rust实现一个链表。这个答案取决于你的需求是什么，当场回答这个问题显然是很
困难的一件事情。所以我决定写这本书来一次性的把这个问题讲清楚。

在这个系列的文章中，我会通过实现6种不同的链表来教会你Rust编程的一些基础和进阶的知识。在阅
读后面的内容之前，你需要先知道：

* 下面几种指针类型：
    * `&`
    * `&mut`
    * `Box`
    * `Rc`
    * `Arc`
    * `*const`
    * `*mut`
* 所有权，借用，继承可变性，内部可变性，复制
* 所有的关键字：struct, enum, fn, pub, impl, use, ...
* 模式匹配，泛型，析构函数
* 测试
* Unsafe Rust基础

是的，链表是个很糟糕的东西，如果想要在实际应用中使用到它，你必须处理好所有的这些概念。

虽然，在侧边栏有本书的目录（在手机上可能被折叠），但是为了方便快速索引，我们还是把目录放在下面：

1. [A Bad Singly-Linked Stack](first.md)
2. [An Ok Singly-Linked Stack](second.md)
3. [A Persistent Singly-Linked Stack](third.md)
4. [A Bad But Safe Doubly-Linked Deque](fourth.md)
5. [An Unsafe Singly-Linked Queue](fifth.md)
6. [TODO: An Ok Unsafe Doubly-Linked Deque](sixth.md)
7. [Bonus: A Bunch of Silly Lists](infinity.md)

正如你所看到的那样，我会写出所有我在终端中执行的命令。我也会使用Rust标准的打包工具（Cargo）
来开发这个项目。Cargo并不是写Rust程序必须的工具，但是用它比直接用rustc好多了。如果你只是想写几个
简单的程序试试，你也可以直接在浏览器中使用官方提供的[play.rust-lang.org][play]。

首先，我们需要创建出我们的项目：

```text
> cargo new --lib lists
> cd lists
```

我们会将每个链表的实现分别放在不同的文件中，这样我们就不用丢掉已经完成的工作。

值得一提的是，Rust编程的*真实体验*包含了写代码，被编译器怼，和搞清楚编译器是什么意思。我们会
保证这种体验会经常的发生。学会读懂Rust超级优秀的编译器给出的提示和文档对于一个Rust程序员来讲，
也是很重要的一项生存技能。

我们不会进展的特别快，我也不想特别严肃。我认为编程应该是一件有意思的事情。如果你想要看到一些特别
严肃，正式的内容，这本书不适合你。

# An Obligatory Public Service Announcement

Just so we're totally 100% clear: I hate linked lists. With
a passion. Linked lists are terrible data structures. Now of course there's
several great use cases for a linked list:

* You want to do *a lot* of splitting or merging of big lists. *A lot*.
* You're doing some awesome lock-free concurrent thing.
* You're writing a kernel/embedded thing and want to use an intrusive list.
* You're using a pure functional language and the limited semantics and absence
  of mutation makes linked lists easier to work with.
* ... and more!

But all of these cases are *super rare* for anyone writing a Rust program. 99%
of the time you should just use a Vec (array stack), and 99% of the other 1%
of the time you should be using a VecDeque (array deque). These are blatantly
superior data structures for most workloads due to less frequent allocation,
lower memory overhead, true random access, and cache locality.

Linked lists are as *niche* and *vague* of a data structure as a trie. Few would
balk at me claiming a trie is a niche structure that your average programmer
could happily never learn in an entire productive career -- and yet linked lists
have some bizarre celebrity status. We teach every undergrad how to write a
linked list. It's the only niche collection
[I couldn't kill from std::collections][rust-std-list]. It's
[*the* list in C++][cpp-std-list]!

We should all as a community say *no* to linked lists as a "standard" data
structure. It's a fine data structure with several great use cases, but those
use cases are *exceptional*, not common.

Several people apparently read the first paragraph of this PSA and then stop
reading. Like, literally they'll try to rebut my argument by listing one of the
things in my list of *great use cases*. The thing right after the first
paragraph!

Just so I can link directly to a detailed argument, here are several attempts
at counter-arguments I have seen, and my response to them. Feel free to skip
to [the first chapter](first.md) if you just want to learn some Rust!




## Performance doesn't always matter

Yes! Maybe your application is I/O-bound or the code in question is in some
cold case that just doesn't matter. But this isn't even an argument for using
a linked list. This is an argument for using *whatever at all*. Why settle for
a linked list? Use a linked hash map!

If performance doesn't matter, then it's *surely* fine to apply the natural
default of an array.





## They have O(1) split-append-insert-remove if you have a pointer there

Yep! Although as [Bjarne Stroustrup notes][bjarne] *this doesn't actually
matter* if the time it takes to get that pointer completely dwarfs the
time it would take to just copy over all the elements in an array (which is
really quite fast).

Unless you have a workload that is heavily dominated by splitting and merging
costs, the penalty *every other* operation takes due to caching effects and code
complexity will eliminate any theoretical gains.

*But yes, if you're profiling your application to spend a lot of time in
splitting and merging, you may have gains in a linked list*.





## I can't afford amortization

You've already entered a pretty niche space -- most can afford amortization.
Still, arrays are amortized *in the worst case*. Just because you're using an
array, doesn't mean you have amortized costs. If you can predict how many
elements you're going to store (or even have an upper-bound), you can
pre-reserve all the space you need. In my experience it's *very* common to be
able to predict how many elements you'll need. In Rust in particular, all
iterators provide a `size_hint` for exactly this case.

Then `push` and `pop` will be truly O(1) operations. And they're going to be
*considerably* faster than `push` and `pop` on linked list. You do a pointer
offset, write the bytes, and increment an integer. No need to go to any kind of
allocator.

How's that for low latency?

*But yes, if you can't predict your load, there are worst-case
latency savings to be had!*





## Linked lists waste less space

Well, this is complicated. A "standard" array resizing strategy is to grow
or shrink so that at most half the array is empty. This is indeed a lot of
wasted space. Especially in Rust, we don't automatically shrink collections
(it's a waste if you're just going to fill it back up again), so the wastage
can approach infinity!

But this is a worst-case scenario. In the best-case, an array stack only has
three pointers of overhead for the entire array. Basically no overhead.

Linked lists on the other hand unconditionally waste space per element.
A singly-linked lists wastes one pointer while a doubly-linked list wastes
two. Unlike an array, the relative wasteage is proportional to the size of
the element. If you have *huge* elements this approaches 0 waste. If you have
tiny elements (say, bytes), then this can be as much as 16x memory overhead
(8x on 32-bit)!

Actually, it's more like 23x (11x on 32-bit) because padding will be added
to the byte to align the whole node's size to a pointer.

This is also assuming the best-case for your allocator: that allocating and
deallocating nodes is being done densely and you're not losing memory to
fragmentation.

*But yes, if you have huge elements, can't predict your load, and have a
decent allocator, there are memory savings to be had!*





## I use linked lists all the time in &lt;functional language&gt;

Great! Linked lists are super elegant to use in functional languages
because you can manipulate them without any mutation, can describe them
recursively, and also work with infinite lists due to the magic of laziness.

Specifically, linked lists are nice because they represent an iteration without
the need for any mutable state. The next step is just visiting the next sublist.

However it should be noted that Rust can pattern match on arrays and talk
about sub-arrays [using slices][slices]! It's actually even more expressive
than a functional list in some regards because you can talk about the last
element or even "the array without the first and last two elements" or
whatever other crazy thing you want.

It is true that you can't *build* a list using slices. You can only tear
them apart.

For laziness we instead have [iterators][]. These can be infinite and you
can map, filter, reverse, and concatenate them just like a functional list,
and it will all be done just as lazily. No surprise here: slices can also be
coerced to an iterator.

*But yes, if you're limited to immutable semantics, linked lists can be very
nice*.

Note that I'm not saying that functional programming is necessarily weak or
bad. However it *is* fundamentally semantically limited: you're largely only
allowed to talk about how things *are*, and not how they should be *done*. This
is actually a *feature*, because it enables the compiler to do tons of [exotic
transformations][ghc] and potentially figure out the *best* way to do things
without you having to worry about it. However this comes at the cost of being
*able* to worry about it. There are usually escape hatches, but at some limit
you're just writing procedural code again.

Even in functional languages, you should endeavour to use the appropriate data
structure for the job when you actually need a data structure. Yes,
singly-linked lists are your primary tool for control flow, but they're a
really poor way to actually store a bunch of data and query it.


## Linked lists are great for building concurrent data structures!

Yes! Although writing a concurrent data structure is really a whole different
beast, and isn't something that should be taken lightly. Certainly not something
many people will even *consider* doing. Once one's been written, you're also not
really choosing to use a linked list. You're choosing to use an MPSC queue or
whatever. The implementation strategy is pretty far removed in this case!

*But yes, linked lists are the defacto heroes of the dark world of lock-free
concurrency.*




## Mumble mumble kernel embedded something something intrusive.

It's niche. You're talking about a situation where you're not even using
your language's *runtime*. Is that not a red flag that you're doing something
strange?

It's also wildly unsafe.

*But sure. Build your awesome zero-allocation lists on the stack.*





## Iterators don't get invalidated by unrelated insertions/removals

That's a delicate dance you're playing. Especially if you don't have
a garbage collector. I might argue that your control flow and ownership
patterns are probably a bit too tangled, depending on the details.

*But yes, you can do some really cool crazy stuff with cursors.*





## They're simple and great for teaching!

Well, yeah. You're reading a book dedicated to that premise.
Well, singly-linked lists are pretty simple. Doubly-linked lists
can get kinda gnarly, as we'll see.




# Take a Breath

Ok. That's out of the way. Let's write a bajillion linked lists.

[On to the first chapter!](first.md)


[rust-std-list]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[cpp-std-list]: http://en.cppreference.com/w/cpp/container/list
[github]: https://github.com/rust-unofficial/too-many-lists
[bjarne]: https://www.youtube.com/watch?v=YQs6IC-vgmo
[slices]: https://doc.rust-lang.org/edition-guide/rust-2018/slice-patterns.html
[iterators]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[ghc]: https://wiki.haskell.org/GHC_optimisations#Fusion
[play]: https://play.rust-lang.org/
