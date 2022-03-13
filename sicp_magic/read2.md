## 1.3 formulating abstractions with higher-order procedures
use (* x x x) instead of (cubic x) will cause great disadvantage, this forces us to work always at the level of the particular operations that happen to be primitives in the language.

One of the things we should demand from a powerful programming language is the ability to build abstractions by assigning names to common patterns and then to work in terms of the abstractions directly.

Procedures that manipulate procedures are called higher-order procedures.

an example is shown below:
```scheme
;instead of showing the cases explicitly, we can abstract like below:
(define (sum term a next b)
    (if (> a b)
        0
        (+ (term a) (sum term (next a) next b))
    )
)
;you can even use this to calculate integral.
(define (integral f a b dx)
    (define (add-dx x) (+ x dx))
    (* (sum f (+ a (/ dx 2.0)) add-dx b) dx)
)
(define (cube x) (* x x x))
(integral cube 0 1 0.01)
```
### ex1.29
make notes for Simpson's Rule. It will be shown in details in appendix.

first, for normal calculate ways, the result is shown below:
```
0.24998750000000042 ---> n = 100
0.249999875000001 ---> n = 1000
```
Now we use Simpson's Rule to compute the result.
```scheme
;because the simpson is based on the even and odd, we need count to represent it better.
(define (sum term a next b count)
    (if (> a b)
        0
        (+ (term a count) (sum term (next a) next b (+ count 1)))
    )
)
(define (cube x) (* x x x))
(define (simpson-rule f a b n)
    (define (term x count)
        (if (or (= 0 count) (= n count))
            (f x)
            (if (even? count)
                (* 2 (f x))
                (* 4 (f x))
            )
        )
  )
  (define (next x) (+ x (/ (- b a) n)))
  (* (/ (- b a) (* 3 n)) (sum term a next b 0))
)

(simpson-rule cube 0 1 100)
(simpson-rule cube 0 1 1000)
```
the result is shown below:
```
drRacket has round over this simply to 1/4:
1/4
1/4
```