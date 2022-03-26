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