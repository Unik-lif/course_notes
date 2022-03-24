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