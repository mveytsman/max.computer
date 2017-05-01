---
title: "The perils of doing SICP on modern computers"
date: 2016-03-04
comments: true
categories: 
draft: true
---

<script src="/javascripts/deps/codemirror/lib/codemirror.js"></script>
<script src="/javascripts/deps/codemirror/mode/scheme/scheme.js"></script>
<script src="/javascripts/coding.js"></script>
<script>
  c = new CodingJS("/javascripts/");
</script>


<div id="deps">
<code>
(define (square x)
  (* x x))
  
  
(define (smallest-divisor n)
  (find-divisor n 2))

(define (find-divisor n test-divisor)
  (cond ((> (square test-divisor) n) 
         n)
        ((divides? test-divisor n) 
         test-divisor)
        (else (find-divisor 
               n 
               (+ test-divisor 1)))))

(define (divides? a b)
  (= (remainder b a) 0))
  (define (prime? n)
  (= n (smallest-divisor n)))
  


(define (time f . args)
  (newline)
  (let ((start-time (runtime)))
     (display (apply f args))
     (display " *** ")
     (display (- (runtime) start-time))))
</code>
</div>

<script>
  c.prompt("deps");
</script>





I'm going through [SICP](https://en.wikipedia.org/wiki/Structure_and_Interpretation_of_Computer_Programs) with a reading group in Toronto (if you're in Toronto you should check us out at [https://github.com/CompSciCabal/SMRTYPRTY](https://github.com/CompSciCabal/SMRTYPRTY)).

As mind expanding as reading a seminal computer science text from the 80's is, sometimes you see the advantages of studying computers back when they ran at speeds comprehendable to mere mortals. For instance, in a memorable section, you implement a couple of primality tests, and then analyze their runtime complexity by finding successively larger prime numbers and seeing how long it takes.

Here's SICP:

> Use your procedure to find the three smallest primes larger than 1000; larger than 10,000; larger than 100,000; larger than 1,000,000. Note the time needed to test each prime. Since the testing algorithm has order of growth of Θ(√n), you should expect that testing for primes around 10,000 should take about 10⎯⎯⎯⎯√10 times as long as testing for primes around 1000. 

Ha --- I'm timing in milliseconds, and it took me up to 10,000,000 to get the
"naive" prime test to take a 1 millisecond.

Things get worse when you implement a smarter (but probabilistic) primality test. The test is called Fermat's primality test. Again, SICP:

> The Θ(log⁡(n)) primality test is based on a result from number theory known as Fermat’s Little Theorem.45

> Fermat’s Little Theorem: If n is a prime number and a is any positive integer less than n, then a raised to the nth power is congruent to a modulo n.

> (Two numbers are said to be congruent modulo n if they both have the same remainder when divided by n. The remainder of a number aa when divided by n is also referred to as the remainder of a modulo n, or simply as a modulo n.)

> If n is not prime, then, in general, most of the numbers a < n will not satisfy the above relation. This leads to the following algorithm for testing primality: Given a number n, pick a random number a < n and compute the remainder of a^n modulo n. If the result is not equal to a, then nn is certainly not prime. If it is a, then chances are good that n is prime. Now pick another random number a and test it with the same method. If it also satisfies the equation, then we can be even more confident that n is prime. By trying more and more values of a, we can increase our confidence in the result. This algorithm is known as the Fermat test.

And SICP's implementation is as follows

<div id="fast-prime">
<code>
(define (expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (remainder
          (square (expmod base (/ exp 2) m))
          m))
        (else
         (remainder
          (* base (expmod base (- exp 1) m))
          m))))
          
(define (fermat-test n)
  (define (try-it a)
    (= (expmod a n n) a))
  (try-it (+ 1 (random (- n 1)))))
  
(define (fast-prime? n times)
  (cond ((= times 0) true)
        ((fermat-test n)
         (fast-prime? n (- times 1)))
        (else false)))
</code>
</div>


<script>
 c.frozen_prompt("fast-prime", ["deps"]);
</script>

We can compare the time it takes to do the naive version, vs the `fast-prime?` version.

<div id="prompt1">
<code>
(display "(prime? 100003)")
(time prime? 100003)
(newline)
(display "(fast-prime? 100003 2)")
(time fast-prime? 100003 2)
(newline)
(display "(prime? 1000037)")
(time prime? 1000037)
(newline)
(display "(fast-prime? 1000037 2)")
(time fast-prime? 1000037 2)

</code>
</div>

<script>
 c.prompt("prompt1", ["fast-prime", "deps"]);
</script>

But then something interesting happens when we get to primes above 10,000,000 **warning: this takes about 2 seconds for `prime` to run on my browser**.

<div id="prompt2">
<code>
(display "(fast-prime? 10000000019 2)")
(time fast-prime? 10000000019 2)
(newline)
(display "(prime? 10000000019)")
(time prime? 10000000019)
</code>
</div>

<script>
 c.prompt("prompt2", ["fast-prime", "deps"]);
</script>

Oh oh. Looks like our fast-prime test is *failing* on numbers around 10,000,000.
It's a probabilistic test, but it's always going to be true for primes --- that
is the fermat's test is necessary but not sufficient to be a prime.

The second argument to `fast-prime?` is how many times to run the test, and even
though I understood the above, I still ran it with absurdly high numbers of
iterations just to be sure.

You might be wondering if 10000000019 is actually prime. A quick google led me to https://primes.utm.edu/curios/page.php/10000000019.html so at least we're not alone in thinking it's prime. (The fun fact about this prime listed on that site is that "The sum of the digits of the smallest eleven-digit prime is eleven." Looks like they're digging deep for the curios here...)

At this point I gave up on the whole endeavor, until I met with the reading
group and found that [@j0ni](https://twitter.com/j0ni) had the same exact issue!
Neither of us could get the `fast-prime?` test to work on any numbers greater than 10,000,000.

As an aside, we're both using [Chicken Scheme](http://www.call-cc.org/).

The culprit is the `expmod` procedure. The Fermat test is that if a number p is prime, `(expmod a p p)` will be `a` for any `a` less than `p`.

Running `(expmod 32 10000000019 10000000019)` on my machine, I get `5293704950.0`. That decimal point is a telltale sign that we blew past the maximum integer and got into float-land.

I can modify the recursive `expmod` to print intermediate values:

<div id="print-expmod">
<code>
(define (expmod base exp m)
  (let ((r (cond ((= exp 0) 1)
                 ((even? exp)
                  (remainder 
                   (square (expmod base (/ exp 2) m))
                   m))
                 (else
                  (remainder 
                   (* base (expmod base (- exp 1) m))
                   m)))))
       (display "(expmod ")
       (display base)
       (display " ")
       (display exp)
       (display " ")
       (display m)
       (display ") ;=> ")
       (display r)
       (display "\n")
       r))

(expmod 32 10000000019 10000000019)
</code>
</div>
<script>
 c.prompt("print-expmod", ["deps"])
</script>


Things break down at `(expmod 32 18 10000000019)`

When `expmod` is called with an even exponent, it recurs with `(remainder (square (expmod base (/ exp 2) m)) m)`. This is to use squaring and halving the exponent to get logarithmic growth.

And indeed, if we perform the step directly for m = 18, we blow past the integers.

<div id="remainder">
<code>
;;(remainder (square (expmod 32 (/ 18 2) 10000000019)) 10000000019)) 
;; expands =>
;;(remainder (square 4372021990) 10000000019)
;; expands =>
(square 4372021990)
</code>
</div>
<script>
 c.prompt("remainder", ["deps"])
</script>

And there you have it. The successive squaring blows past the max int, and we get into float-land, losing precision and giving us erroneous results for expmod.

** Note in js the highest full precision number is 9007199254740991, wish there was a better way to illustrate it, like getting js to display it with an exponent **

## Fixing it

The first thing we can do is get rid of the squaring and halving part in expmod so we never get numbers that are too big:

<div id="slow-expmod">
<code>
(define (slow-expmod base exp m)
  (if (= exp 0)
      1
      (remainder 
       (* base (slow-expmod base (- exp 1) m))
       m)))
</code>
</div>
<script>
 c.prompt("slow-expmod", ["deps"])
</script>


You can try computing some values above, but unfortunately, `(slow-expmod 32 32 10000000019)` is slow enough to wear out my patience even on my "modern computer". The fact that we create 10000000019 stack frames certainly doesn't help.

A better solution exists, all you have to do is, as SICP likes to say, observe that `ab mod p = (a mod p)(b mod p) mod p`. Instead of computing the square and then finding the remainder, we can write a remainder-aware `square` procedure that computes squares while taking the remainder of intermediate results.

In an exercise, you're asked to make a fast multiply procedure that uses doubling and halving to ger O(log(n)) speed. We can use it to build a remainder-aware multiply procedure as a base of our remainder aware `sqaure`.

My fast multiply is:

<div id="fast-mult">
<code>
(define (fast-* a b)
  (cond ((= b 0)
         0)
        ((even? b)
         (fast-* (* a 2) (/ b 2)))
        (else
         (+ a (fast-* a (- b 1))))))
</code>
</div>
<script>
 c.frozen_prompt("fast-mult", ["deps"])
</script>


We can make this remainder-aware pretty easily:

<div id="fast-mod">
<code>
(define (fast-modulo-* a b m)
  (cond ((= b 0)
         0)
        ((even? b)
         (fast-modulo-* (remainder (* a 2) m) (remainder (/ b 2) m) m))
        (else
         (remainder (+ (remainder a m)  (fast-modulo-* a (- b 1) m)) m))))
         
</code>
</div>
<script>
 c.frozen_prompt("fast-mod", ["deps", "fast-mult"])
</script>

Giving us a `square-modulo`:

<div id="square-mod">
<code>
(define (square-modulo a m)
  (fast-modulo-* a a m))
</code>
</div>
<script>
 c.frozen_prompt("square-mod", ["deps", "fast-mod"])
</script>

Putting it into the `safe-expmod`, we get

<div id="safe-expmod">
<code>
(define (safe-expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (square-modulo (safe-expmod base (/ exp 2) m) m))
        (else
          (fast-modulo-* base (safe-expmod base (- exp 1) m) m))))
</code>
</div>
<script>
 c.frozen_prompt("safe-expmod", ["deps", "square-mod"])
</script>

And it works!

<div id="prompt3">
<code>
(safe-expmod 32 10000000019 10000000019)
</code>
</div>
<script>
 c.prompt("prompt3", ["deps", "safe-expmod"])
</script>

We can rewrite the Fermat test to use  `safe-expmod`:

<div id="safe-fermat">
<code>
(define (safe-fermat-test n)
  (define (try-it a)
    (= (safe-expmod a n n) a))
  (try-it (+ 1 (random (- n 1)))))

(define (safe-fast-prime? n times)
  (cond ((= times 0) #t)
        ((safe-fermat-test n) 
         (safe-fast-prime? n (- times 1)))
        (else #f)))
</code>
</div>

<script>
 c.frozen_prompt("safe-fermat", ["deps", "safe-expmod"])
</script>


And lo and behold:


<div id="prompt4">
<code>
(safe-fast-prime? 10000000019 2)
</code>
</div>
<script>
 c.prompt("prompt4", ["deps", "safe-fermat"])
</script>
