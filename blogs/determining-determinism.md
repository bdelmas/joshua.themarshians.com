<!-- Title: Determining Determinism -->
<!-- Author: Joshua Marsh -->
<!-- Tags: determinism,memoization,go,recursion -->
<!-- Languages: go,shell -->

I've always lived by the programming adage, "Get it working, then make
it either readable or fast." When you are developing games or other
high performance applications, the making it fast part can consume a
lot of your time.

I can't count the number of hours I've spent looking at
[call graphs](http://en.wikipedia.org/wiki/Call_graph). What do you do
when you've determined your bottleneck though?  Most people start
looking at common things like unraveling loops and
[recursion](http://en.wikipedia.org/wiki/Recursion) or creating
[look up tables](http://en.wikipedia.org/wiki/Lookup_table)
(LUTs). LUTs are a common practice in the gaming industry for things
like trig functions. Why?  Well, their values are static. The value of
sin(45°) will always be
[0.7071067812](http://www.wolframalpha.com/input/?i=sin(45+deg)).

There are two reasons that make a LUT useful. First, the cost of
looking up a value in a hash or array is usually orders of magnitude
smaller than performing the calculation multiple times. Second, the
math functions being called are deterministic. Translation: the
solution will always be the same for a given input.

You may be asking what this has to do with
[deterministic algorithms](http://en.wikipedia.org/wiki/Deterministic_algorithm). Well,
you can take the idea of a LUT one step further, but I see this much
less frequently even though it can be incredibly powerful. Any
algorithm that is deterministic can have it's own LUT. This is often
called [memoization](http://en.wikipedia.org/wiki/Memoization), but
most people don't recognize that it's possible because the function is
deterministic. Let me show you how it works. A normal fibonacci
algorithm in [go](http://golang.org/) might look like this:

<pre><code data-language="go">func Fibonacci(n int64) int64 {  
     var f int64  
     if n <= 1 {  
          f = 1  
     } else {  
          f = Fibonacci(n-1) + Fibonacci(n-2)  
     }  
     return f  
}  
</code></pre>


This works fine for small values of <em>n</em> like 1 or 7. But when n is 100,
this could take quite a bit of time to run. Because it's recursive,
proving that the function is deterministic has huge gains. I'll leave
the details of the proof to you, but for the sake of this post let's
just assume it is (it is).

Knowing that this function is deterministic is important! It means we
can store the previously calculated values and return them or use them
in calculating new values. Here is one possible way to do this:


<pre><code data-language="go">var fibonacciLUT map[int64]int64

func MemoizedFibonacci(n int64) int64 {  
     var f int64  
     if stored, ok := fibonacciLUT[n]; ok {  
          return stored  
     }  
     if n <= 1 {  
          f = 1  
     } else {  
          f = MemoizedFibonacci(n-1) + MemoizedFibonacci(n-2)  
     }  
     fibonacciLUT[n] = f  
     return f  
}  
</code></pre>

Not much has changed. First, we create a map (an array could also work
in this case) to store the values. Next, before we start doing the
computations, we look to see if we already have a stored result. If we
do, then we return that. If not, we perform the calculation and save
the value into our map.

How much does this improve our performance? SO MUCH I HAVE TO USE CAPS
TO SAY HOW MUCH!

<pre><code data-language="shell">$ go run fibonacci.go  
50 iterations of Non-memoized took 5m37.824026s to complete.  
50 iterations of Memoized took 106us to complete.  
</code></pre>

Yes, you are reading that right. The normal calls took more than five
minutes to run and the memoized version took 106 microseconds. You can
see the entire program in [one of my gists](https://gist.github.com/3244972).

So, why am I writing about this. Computer science is important! You
need to understand things like deterministic algorithms,
[state-space graphs](http://en.wikipedia.org/wiki/State_space_search),
and everything else that you may have missed while you were on
Facebook in your CS classes. They are what separate script kiddies
from coding ninjas.

You might say, "The Zuckster never graduated!" I don't care how or
where you learn it. Whether its [MIT](http://mit.edu/),
[Coursera](https://www.coursera.org/), or
[Wikipedia](http://www.wikipedia.org/), good programmers need to
understand these things. I've come across too many people who can't
answer fundamental computer science questions and I'll bet my bottom
dollar that this problem is directly correlated to why I have to
maintain unreadable, slow code.

Go forth and learn!
