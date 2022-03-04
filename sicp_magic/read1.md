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