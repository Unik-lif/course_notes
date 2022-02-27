### essence of computer science.
how to formalize intuitions about computer science.

Lisp: casting a spell.
## The most important thing
### techiques for controlling complexity.
don't care about other things.

### focus
1. black box abstraction
2. conventional interfaces
3. metalinguistic abstraction

### a general idea
means of combinations and abstractions

no difference between these two below:
```scheme
(define (square x) (* x x)) ;really is naming something in essence
(define square (lambda(x) (* x x)))
```
for condition:
```scheme
(define (abs x)
    (cond ((< x 0) (-x))
          ((= x 0) 0)
          ((> x 0) x)
    )
)
```
for value and functions(procedure):
```scheme
(define A (* 5 5)); a value
(define (A) (* 5 5)); a procedure
```