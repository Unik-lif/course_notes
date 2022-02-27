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