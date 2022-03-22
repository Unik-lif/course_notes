## Module
In large system, we divorce the task of building things from the task of implementing it.

Abstraction Barriers.

## Wishful thinking
image we have three procedures:
```scheme
;take two rational numbers and return a rational number.
(define (+RAT X Y)
    (Make-rat
        (+ (* (Numer X) (Denom Y)) (* (Numer Y) (Denom X)))
        (* (Denom X) (Denom Y))
    )
)
;I've already use cloud to express rational number.
;So how can we 
;(x + y) * (z + k)
;(*RAT (+RAT x y) (+RAT z k))
```
Basically it is like to bind numer with denom together with a name 'cloud', then their relationships are set to the stone.

How can we actually bind them together?
## List structure
```scheme
(cons x y);constructs a pair whose first part is x and whose second part is y
(car p);selects the first part of the pair p
(cdr p);selects the second part of the pair p
```
we can use box & pointer notation to express this.
```scheme
(define (make-rat N D)
    (cons N D)
)
(define (numer x)
    (car x)
)
(define (denom x)
    (cdr x)
)
(define (make-rat n d)
    (let ((g (gcd n d)))
        (cons (/ n g) (/ d g))
    )
)
```
Data abstraction is sort of the programming methodology of setting up data objects by postulating constructors and selectors to isolate use from representation.

Another example:
```scheme
(define (make-vector x y) (cons x y))
(define (xcor p) (car p))
(define (ycor p) (cdr p))
(define (make-seg p q) (cons p q))
(define (seg-start s) (car s))
(define (seg-end s) (cdr s))
(define (midpoint s)
    (let ((a (seg-start s))
          (b (seg-end s)))
        (make-vector
            (average (xcor a) (xcor b))
            (average (ycor a) (ycor b))
        )
    )
)
(define (length s)
    (let
        ((dx (- (xcor (seg-end s)) (xcor (seg-start s))))
         (dy (- (ycor (seg-end s)) (ycor (seg-start s))))
        )
        (sqrt (+ (square dx) (square dy)))
    )
)
```
It is like:
```
segments

-------------------------
make-seg/seg-start/seg-end
-------------------------

vectors

-------------------------
make-vector/xcor/ycor
-------------------------

Pairs

-------------------------
can we do sth here?
-------------------------
```
We can try another method:
```scheme
(define (cons a b)
    (lambda (pick)
        (cond ((= pick 1) a)
              ((= pick 2) b)
        )
    )
)
(define (car x) (x 1))
(define (cdr x) (x 2))
;very interesting design way. In fact, all the data structures can reduced to function.
;that's how we can use closure.
;even though we thinks it to be a data, it is just a procedure.
```
