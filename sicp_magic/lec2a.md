### High-order Procedure
how to write a more general sum_up program?
```scheme
;Add from 0 to b.
(define (sum-int a b)
    (if (> a b)
        0
        (+ a (sum-int (1 + a) b))
    )
)

;Add from 0 to b, square.
(define (sum-sq a b)
    (if (> a b)
        0
        (+ (square a) (sum-sq (1 + a) b))
    )
)

;these things have something in common, so how to simplify it?
;at least the recursion structure is similar.
;we condense a general model here.
(define (sum term a next b)
    (if (> a b)
        0
        (+ (term a) (sum term (next a) next b))
    )
)
;we use term to get the a-element, "next" method to get the next a.
(define (sum-int a b)
    (define (identity a) a)
    (sum identity a 1+ b)
)

;we now talk about fixed point.
(define (fixed-point f start)
    (define (iter old new)
        (if (close-enough? old new)
            new
            (iter new (f new))
        )
    )
    (iter start (f start))
)

;procedure can be seen as operator in math.
;lambda will give us anoymous benefits.
(define average-damp
    (lambda (f)
        (lambda (x) (average (f x) x))
    )
)

;Newton method
;to find a y such that f(y) = 0, start with a guess y_{0}, iterate y_{n + 1} & y_{n}
;use wishful thinking as a habbit.

```
