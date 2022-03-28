## Unit2 reading
chapter 1: building abstractions by combining procedures.

chapter 2: building abstractions by combining data objects.

### The reason for compound procedures:
To elevate the conceptual level.

THe use of compound data leads to a real increase in the expressive power of our programming language.
```scheme
;consider the idea of forming a linear combination: ax + by
(define (linear-combination a b x y)
    (+ (* a x) (* b y))
)
(define (linear-combination a b x y)
    (add (mul a x) (mul b y))
)
```
One key idea in dealing with compound data is the notion of closure - that the glue we use for combining data objects should allow us to combine not only primitive data objects, but compound data objects as well.

## 2.1
Data abstraction is to structure the programs that are to use compound data objects so that they operate on "abstract data".

### "Wishful thinking"
We haven't yet said how a rational number is represented, or how the procedures numer, denom, and make-rat should be implemented.

### Pair:
cons->construct
car->contents of address part of Register
cdr->contents of decrement part of Registers

```scheme
(define make-rat cons)
(define (make-rat n d) (cons n d))
(define (numer x) (car x))
(define (denom x) (cdr x))
(define (print-rat x)
    (newline)
    (display (numer x))
    (display "/")
    (display (denom x))
)
```

### ex2.1
```scheme
(define (make-rat n d)
    (if (>= (* n d) 0)
        (cons (abs n) (abs d))
        (cons (abs n) (* -1 (abs d)))
    )
)
```
### 2.1.2
abstraction barriers.
```
programs that use rational numbers

-----------------------------------
Rational numbers in problem domain
-----------------------------------

add-rat sub-rat ...................

-----------------------------------
Rational numbers as numerators and denominators
-----------------------------------

make-rat numer denom

-----------------------------------
rational numbers as pairs
-----------------------------------

cons car cdr

-----------------------------------
however pairs are implemented
```
Advantage:
make programs much easier to maintain and to modify. (module, easy to change)

The data-abstraction methodology gives us a way to defer that decision without losing the ability to make progress on the rest of the system.

### ex2.2
```scheme
;segment-point
(define (make-segment start end)
    (cons start end)
)
(define (start-segment segment)
    (car segment)
)
(define (end-segment segment)
    (cdr segment)
)
;point-numbers
(define (make-point x y)
    (cons x y)
)
(define (x-point point)
    (car point)
)
(define (y-point point)
    (cdr point)
)
;basic
(define (average a b)
    (/ (+ a b) 2)
)
;Usage part
(define (midpoint-segment segment)
    (make-point (average (x-point (start-segment segment)) (x-point (end-segment segment)))
                (average (y-point (start-segment segment)) (y-point (end-segment segment)))
    )
)
;output procedure
(define (print-point p)
    (newline)
    (display "(")
    (display (x-point p))
    (display ".")
    (display (y-point p))
    (display ")")
)
```
### ex2.3
```scheme
;tools to compute perimeter and the area.
(define (sqrt x)
    (define (close-enough? a b)
        (< (abs (- a b)) 0.0001)
    )
    (define (guess num)
        (if (close-enough? x (square num))
            num
            (guess (average (/ x num) num))
        )
    )
    (guess 1)
)
(define (square x)
    (* x x)
)
(define (segment-length segment)
    (sqrt (square (- (x-point (start-segment segment) (x-point (end-segment segment)))))
          (square (- (y-point (start-segment segment) (y-point (end-segment segment)))))
    )
)
;first way to build a rectangle
(define (rectangle a-segment b-segment)
    (cons a-segment b-segment)
)
(define (first-segment rectangle)
    (car rectangle)
)
(define (second-segment rectangle)
    (cdr rectangle)
)
(define (area rectangle)
    (* (first-segment rectangle) (second-segment rectangle))
)
(define (perimeter rectangle)
    (* 2 (+ (first-segment rectangle) (second-segment rectangle)))
)
;if you want the same perimeter and area procedures will work using either representation, this might be a little inconvient.
;for rectangles, you basically got two ways to construct this, basically 3 points or 2 segments.
;if you use 2 segments in two ways to construct your target, then It will work very well, otherwise not.
```
### 2.1.3
Abstract models: procedures plus conditions, by Hoare.

Procedures: an abstract algbraic system whose behavior is specified by axioms that correspond to our conditions.

we can use procedures to represent data structures.
```scheme
(define (cons x y)
    (define (dispatch m)
        (cond ((= m 0) x)
              ((= m 1) y)
              (else (error "Argument not 0 or 1 - CONS" m))
        )
    )
)
(define (car z) (z 0))
(define (cdr z) (z 1))
```
### ex2.4
```scheme
(define (cons x y)
    (lambda (m) 
        (m x y)
    )
)
(define (car z)
    (z (lambda (p q) p))
)
;((cons x y) (lambda (p q) p))
;((lambda (p q) p) x y)
;x
(define (cdr z)
    (z (lambda (p q) q))
)
```
### ex2.5
if $2^{a1}3^{b1} = 2^{a2}3^{b2}$, then $a_1 = a_2, b_1 = b_2$.
```scheme
(define (power a x)
    (if (= 0 x)
        1
        (* a (power a (- x 1)))
    )
)
(define (cons a b)
    (* (power 2 a) (power 3 b))
)
(define (car con)
    (define (count con num)
        (if (= (remainder con 2) 0)
            (count (/ con 2) (+ 1 num))
            num
        )
    )
    (count con 0)
)
(define (cdr con)
    (define (count con num)
        (if (= (remainder con 3) 0)
            (count (/ con 3) (+ 1 num))
            num
        )
    )
    (count con 0)
)
```
### ex2.6
```scheme
(define one (lambda (f) (lambda (x) (f x))))
(define two (lambda (f) (lambda (x) (f (f x)))))
(define (+ a b)
    (lambda (f) (lambda (x) ((b f) ((a f) x))))
)
```
### ex2.7
```scheme
(define (make-interval a b)
    (cons a b)
)
(define (upper-bound interval)
    (cdr interval)
)
(define (lower-bound interval)
    (car interval)
)
```
### ex2.8
```scheme
(define (sub-interval x y)
    (make-interval (- (lower-bound x) (upper-bound y))
                   (- (upper-bound x) (lower-bound y))
    )
)
```
### ex2.9
too easy.
### ex2.10
```scheme
;the former div-interval
(define (div-interval x y)
    (mul-interval x
        (make-interval (/ 1.0 (upper-bound y))
                       (/ 1.0 (lower-bound y))
        )
    )
)
```