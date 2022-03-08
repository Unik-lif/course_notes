## 1.1
the first part of sicp is simply telling the rules of scheme.

a very interesting thing is that:
```scheme
(define (>= x y)
    (or (> x y) (= x y)))
```
so we skip to exercise part:
### ex1.1
the result is given below
```
10 12 8 3 6 a b 19 false 4 16 6 16
```
### ex1.2
```scheme
(/ (+ 5 4 (- 2 (- 3 (+ 6 (/ 1 3))))) (* 3 (- 6 2) (- 2 7)))
```
### ex1.3
```scheme
(define (sum_twolarger a b c)
    (if (> a b)
        (if (> b c)
            (+ a b)
            (+ a c)
        )
        (if (> a c)
            (+ a b)
            (+ b c) 
        )
    )
)
```
### ex1.4
this procedure first evaluate the operator. The operator depends on the sign of b, we use if to judge. Then, we simply add them up.
### ex1.5
in test procedure, to eval the second operend (p), the program will call (p), which comes to be a endless recursive function.

functions: describing properties of things.

procedure: describing how to do things.

In mathematics we are ususally concerned with declarative (what is) descriptions, whereas in computer science we are usually concerned with imperative (how to) descriptions.

below we will give an example of (how to) procedure. To get the radicand, we starts from very small end.
```scheme
(define (sqrt-iter guess x)
    (if (good-enough? guess x)
        guess
        (sqrt-iter (improve guess x)
            x)))
; at this time, we don't care about good-enough, improve function.
; Though, we can simply use them. Their detailed functions will be added soon.
(define (improve guess x)
    (average guess (/ x guess)))

(define (average x y)
    (/ (+ x y) 2))

(define (good-enough? guess x)
    (< (abs (- (square guess) x)) 0.001))

(define (sqrt x)
    (sqrt-iter 1.0 x))
```
### ex1.6
In fact these two don't have great differences. I track their effiency, that doesn't count a lot.

Maybe new-if will be more costly.
### ex1.7
basically this is very similar to what we usually referred as fixed point.
```scheme
(define (good-enough? guess x)
    (< (- guess (/ x guess)) 0.001))
```
### ex1.8
Newtown's method for cube roots.
```scheme
(define (cubic-root x)
    (cubic-iter 1.0 x))

(define (cubic-iter guess x)
    (if (good-enough? guess x)
        guess
        (cubic-iter (improve guess x) x)))

(define (improve guess x)
    (/ (+ (/ x (* guess guess)) (* 2 guess)) 3))

(define (good-enough? guess x)
    (< (abs (- (* guess guess guess) x)) 0.001))
```
## 1.2
iterative case, the program variables provide a complete description of the state of the process at any point. So long as we get the number, we can easily resume.

recursive case, there is some additional hidden information, maintained by the interpreter and not contained in the program variables.

tail-recursion: execute an iterative process in constant space. With a tail-recursive implementation, iteration can be expressed using the ordinary procedure call mechanism.
### ex1.9
```scheme
;for the first procedure
(define (+ a b)
    (if (= a 0)
        b
        (inc (+ (dec a) b))))
#|
(+ 4 5)
(inc (+ 3 5))
(inc (inc (+ 2 5)))
(inc (inc (inc (+ 1 5))))
(inc (inc (inc (inc (+ 0 5)))))
(inc (inc (inc (inc 5))))
(inc (inc (inc 6)))
(inc (inc 7))
(inc 8)
9
recursive
|#

;for the second procedure
(define (+ a b)
    (if (= a 0)
        b
        (+ (dec a) (inc b))))
#|
(+ 4 5)
(+ 3 6)
(+ 2 7)
(+ 1 8)
(+ 0 9)
9
iterative
|#
```
### ex1.10
```scheme
(define (A x y)
    (cond ((= y 0) 0)
          ((= x 0) (* 2 y))
          ((= y 1) 2)
          (else (A (- x 1)
                   (A x (- y 1))))))
; (A 1 10) -> (A (- 1 1) (A 1 (- 10 1))) -> (A 0 (A 1 9)) -> (* 2 (A 1 9)) -> (2 ** 9) * (A 1 1) -> 2 ** 10 -> 1024
; (A 2 4) -> (A 1 (A 2 3)) -> (A 1 (A 1 (A 2 2))) -> (A 1 (A 1 (A 1 (A 2 1)))) -> (A 1 (A 1 (A 1 2))) -> (A 1 (A 1 4)) -> (A 1 16) -> 2 ** 16
; (A 3 3) -> (A 2 (A 3 2)) -> (A 2 (A 2 (A 3 1))) -> (A 2 (A 2 2)) -> (A 2 4) -> 2 ** 16
(define (f n) (A 0 n)); this is equal to 2 * n
(define (g n) (A 1 n)); this is equal to 2 ** n
(define (h n) (A 2 n)); this is equal to 2 ** (A 2 (- n 1)) 
(define (k n) (* 5 n n)); this is equal to 5 * (n ** 2)
```
### counting change:
How many different ways can we make change of $1.00, given half-dollars, quarters, dimes, nickels, and pennies?

The numbers of ways to change amount a using n kinds of conis equals: the number of ways to change amount a using all but the first kind of coin, plus the number of ways to change amount a - d using all n kinds of coins where d is the denomination of the first kind of coin.
```scheme
(define (count-change amount)
    (cc amount 5))
(define (cc amount kinds-of-coins)
    (cond ((= amount 0) 1)
          ((or (< amount 0) (= kinds-of-coins 0)) 0)
          (else (+ (cc amount
                        (- kinds-of-coins 1))
                   (cc (- amount
                          (first-denomination kinds-of-coins))
                        kinds-of-coins)))))
(define (first-denomination kinds-of-coins)
    (cond ((= kinds-of-coins 1) 1)
          ((= kinds-of-coins 2) 5)
          ((= kinds-of-coins 3) 10)
          ((= kinds-of-coins 4) 25)
          ((= kinds-of-coins 5) 50)))
```
tree-recursion is very easy to specify, but not efficient, which has led people to design a smart compiler that could transform tree-recursive procedures into more efficient procedures that compute the same result.

One approach to coping with redundent computations is to store something we already seen before. This strategy, aka tabulation or memoization, can be implemented in a straightforward way.
### ex1.11
```scheme
(define (f n)
    (if (> n 2)
        (f-rec 0 1 2 n)
        n
    )
)
(define (f-rec a b c n)
    (if (= n 2)
        c
        (f-rec b c (+ c (* 2 b) (* 3 a)) (- n 1))
    )
)
```
### ex1.12
```scheme
; we assume that the depth and pos follow the rules.
(define (pascal depth pos)
    (if (or (= pos 0) (= pos depth))
        1
        (+ (pascal (- depth 1) (- pos 1)) (pascal (- depth 1) pos))
    )
)
```
### ex1.13
this is a fairly easy question.

remember: (1 + sqrt(5)) / 2 and (1 - sqrt(5)) / 2 is the root of x ^ 2 - x - 1 = 0. So we can use this to compute higher n.

and the recursion bases is given that fib(0), fib(1) happens to be what we want. So long as the power of (1 - sqrt(5)) / 2 is going up, the fibonacci number is getting closer and closer.
### ex1.14
```
(11, 3) -> 10 + (1, 3) -> (10 + 1)
        -> (11, 2) -> 5 + (6, 2) -> 5 + 5 + (1, 2) -> (5 + 5 + 1)
                                 -> 5 + (6, 1) -> (5 + 6 * 1)
                   -> (11, 1) -> (11 * 1)
```
### ex1.15
```
a. 5 times
b. Space: O(log N) Steps: O(log N)
```
### ex1.16
```scheme
(define (fast-expt b n tmp count)
    (cond ((= n 0) 1)
          ((= count 0) tmp)
          ((even? count) (fast-expt b n (square tmp) (/ count 2)))
          (else (fast-expt b n (* b tmp) (- count 1)))
    )
)

(define (even? n)
    (= (remainder n 2) 0)
)

(define (square x)
    (* x x)
)

(define (fast b n)
    (fast-expt b n 1 n)
)
```
### ex1.17
```scheme
(define (* a b)
    (cond ((= b 0) 0)
          ((even? b) (+ (* a (/ b 2)) (* a (/ b 2))))
          (else (+ a (* a (- b 1))))
    )
)

(define (even? n)
    (= (remainder n 2) 0)
)
```
### ex1.18
```scheme
(define (* a b tmp count)
    (cond ((= b 0) 0)
          ((= count 0) tmp)
          ((even? count) (* a b (+ tmp tmp) (/ count 2)))
          (else (* a b (+ tmp a) (- count 1)))
    )
)

(define (even? n)
    (= (remainder n 2) 0)
)

(define (multi a b)
    (* a b 0 b)
)
```
### ex1.19
```scheme
by simple calculation, we get p = p ^ 2 + q ^ 2. q = 2 * p * q + q ^ 2.
```
this exercise give us a way to think logarithmically.

### GCD
lame's Theorem:
```
(a_k, b_k) -> (a_k-1, b_k-1)
for k = 1, b_0 >= 1 = Fib(1).
then by recursion, we can easily prove that, b_k >= Fib(k).
```
Demo:
```
let n be the smaller of the two inputs to the procedure, if the process takes k steps, then we must have n >= Fib(k).
Hence, the order of Euclid algorithm growth is O(log N).
```
### ex1.20
```
lets do this step by step:
(206, 40) -> (40, 6) -> (6, 4) -> (4, 2) -> (2, 0): return 2.
so we get 4 steps to get the result.
```
### Testing for primality:
Fermat's Little Theorem.
```
if n is a prime number and a is any positive integer less than n, then a raised to n_th power is congruent to a modulo n.
```
some code can be shown like below:
```scheme
(define (expmod base exp m)
    (cond ((= exp 0) 1)
          ((even? exp)
           (remainder (square (expmod base (/ exp 2) m)) m))
          (else
           (remainder (* base (expmod base (- exp 1) m) m)))
    )
)

;random returns a nonnegative integer less than its integer input.
(define (fermat-test n)
    (define (try-it a)
        (= (expmod a n n) a)
    )
    (try-it (+ 1 (random (- n 1))))
)

(define (fast-prime? n times)
    (cond ((= times 0) true)
          ((fermat-test n) (fast-prime? n (- times 1)))
          (else false)
    )
)
```
### ex1.21
```
(199, 2) -> (199, 3) -> (199, 4) -> (199, 5) -> (199, 6) -> (199, 7) .... -> (199, 14): 199.
(1999, 1) ... -> 1999
(19999, 1) ... -> 7
```
### ex1.22
```scheme

```