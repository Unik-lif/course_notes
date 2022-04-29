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
(define (div-interval x y)
    (if (> (* (upper-bound y) (lower-bound y)) 0)
        (mul-interval x
            (make-interval (/ 1.0 (upper-bound y))
                           (/ 1.0 (lower-bound y))
            )
        )
        (error "spans 0")
    )
)
```
### ex2.11
```scheme
(define (mul-interval x y)
    ; (let ((p1 (* (lower-bound x) (lower-bound y)))
    ;      (p2 (* (lower-bound x) (upper-bound y)))
    ;      (p3 (* (upper-bound x) (lower-bound y)))
    ;      (p4 (* (upper-bound x) (upper-bound y))))
    ; (make-interval (min p1 p2 p3 p4) (max p1 p2 p3 p4)))
   
    (let ((+? positive?)
          (-? negative?)
          (lx (lower-bound x))
          (ux (upper-bound x))
          (ly (lower-bound y))
          (uy (upper-bound y)))
         (cond ((test-sign +? +? +? +? x y)
                (make-interval (* lx ly) (* ux uy)))
               ((test-sign +? +? -? +? x y)
                (make-interval (* ux ly) (* ux uy)))
               ((test-sign +? +? -? -? x y)
                (make-interval (* ux ly) (* lx uy)))
               ((test-sign -? +? +? +? x y)
                (make-interval (* lx uy) (* ux uy)))
               ((test-sign -? +? -? +? x y)
                (let ((p1 (* lx ly))
                      (p2 (* lx uy))
                      (p3 (* ux ly))
                      (p4 (* ux uy)))
                     (make-interval (min p2 p3) (max p1 p4))
                ))
               ((test-sign -? +? -? -? x y)
                (make-interval (* ux ly) (* lx ly)))
               ((test-sign -? -? +? +? x y)
                (make-interval (* lx uy) (* ux ly)))
               ((test-sign -? -? -? +? x y)
                (make-interval (* lx uy) (* lx ly)))
               ((test-sign -? -? -? -? x y)
                (make-interval (* ux uy) (* lx ly)))
         )
    )
)

  
 (define (test-sign lx-test ux-test ly-test uy-test x y) 
   (and (lx-test (lower-bound x)) 
        (ux-test (upper-bound x)) 
        (ly-test (lower-bound y)) 
        (uy-test (upper-bound y)))) 
(define a (make-interval -1.0 2.0))
(define b (make-interval 1.0 5.0))
(mul-interval a b)
```
### ex2.12
```scheme
(define (make-center-percent center tolerance)
    (make-interval (- center (* tolerance center)) (+ center (* tolerance center)))
)
(define (center x)
    (/ (+ (lower-bound x) (upper-bound x)) 2)
)
(define (percent x)
    (/ (- (upper-bound x) (lower-bound x)) (+ (lower-bound x) (upper-bound x)))
)
```
### ex2.13
```scheme
; Use leibnitz's way to compute your results.
; Assume all numbers are positive.
(define (mul-percent x y)
    (+ (percent x) (percent y))
)
```
### ex2.14
We simply do a test
```scheme
(define a (make-center-percent 10 0.002))
(define b (make-center-percent 10 0.005))
(define (par1 r1 r2)
    (div-interval (mul-interval r1 r2) (add-interval r1 r2))
)
(define (par2 r1 r2)
    (let ((one (make-interval 1 1)))
        (div-interval one (add-interval (div-interval one r1) (div-interval one r2)))
    )
)
(par1 a b)
(par2 a b)
```
The result is shown below:
```
(4.94773293472845 . 5.052734570998496)
(4.982488710486704 . 5.017488789237668)
```
### ex2.15
She is right.
### ex2.16
Every usage of uncertainty will be accumulate in the end. But you always got a way in the middle-time to remove\mitigate the uncertainty. Do it like digital circuit.

The math explanation might be easier to understand when you use differential equations to represent it.

## 2.2
Not all language have built-in general-purpose glue that makes it easy to manipulate compound data in a uniform way.

### List operations
```scheme
; find the n_th number.
(define (list-ref item n)
    (if (= n 0)
        (car item)
        (list-ref (cdr item) (- n 1))
    )
)
;find the length of the list
(define (length item)
    (if (null? item)
        0
        (+ 1 (length (cdr item)))
    )
)
(define (length_v2 item)
    (define (length-iter a count)
        (if (null? a)
            count
            (length-iter (cdr a) (+ 1 count))
        )
    )
    (length-iter item 0)
)
(define squares (list 1 4 9 16 25))
(define (append list1 list2)
    (if (null? list1)
        list2
        (cons (car list1) (append (cdr list1) list2))
    )
)
;usage of the functions
(list-ref squares 3)
(length squares)
(length_v2 squares)
```

### ex2.17
```scheme
(define (last-pair list)
    (if (nil? (cdr list))
        list
        (last-pair (cdr list))
    )
)
```
### ex2.18
```scheme
(define (reverse item)
    (if (null? (cdr item))
        item
        (append (reverse (cdr item)) (list (car item)))
    )
)
```
### ex2.19
```scheme
<!-- (define (cc amount coin-values)
    (cond ((= amount 0) 1)
          ((or (< amount 0) (no-more? coin-values)) 0)
          (else
            (+ (cc amount
                (except-first-denomination coin-values)
            )
               (cc (- amount (first-denomination coin-values)) coin-values)
            )
          )
    )
) -->
(define (first-denomination coin-values)
    (car coin-values)
)
(define (except-first-denomination coin-values)
    (cdr coin-values)
)
```
This won't change the results.
### ex2.20
```scheme
; dot definition
; (define (f x y . z) <body>)
; (f 1 2 3 4 5 6)
; x: 1
; y: 2
; z: (3 4 5 6)
(define (same-parity . w)
    (define (length list)
        (if (null? list)
            0
            (+ 1 (length (cdr list)))
        )
    )
    (define (select-one counter input output len)
        (if (= counter len)
            output
            (if (= 0 (remainder counter 2))
                (select-one (+ 1 counter) (cdr input) (append output (list (car input))) len)
                (select-one (+ 1 counter) (cdr input) output len)
            )
        )
    )
    (select-one 0 w nil (length w))
)
(same-parity 1 2 3 4 5 6 7)
```
### Mapping over the Lists
```scheme
(define (map proc items)
    (if (null? items)
        nil
        (cons (proc (car items))
            (map proc (cdr items)))
    )
)
```
scheme has a more general form of map.
### ex2.21
```scheme
; first way
(define (square-list items)
    (if (null? items)
        nil
        (cons (* (car items) (car items)) (square-list (cdr items)))
    )
)
; second way
(define (square-list items)
    (map * items items)
)
```
### ex2.22
a). This is because your 'answer' starts from nil.

b). the 'answer' is a list, not an element. use 'append' with 'list' to fix it.
```scheme
(define (square x) (* x x))
(define (square-list items)
    (define (iter things answer)
        (if (null? things)
            answer
            (iter (cdr things) (append answer (list (square (car things)))))
        )
    )
    (iter items nil)
)
```
### ex2.23
```scheme
(define (for-each proc list)
    (define (pack proc list) (proc (car list)) (for-each proc (cdr list)))
    (if (null? list)
        #t
        (pack proc list)
    )
)
```

### 2.2.2
find the total leaves.
```scheme
(define (count-leaves x)
    (cond ((null? x) 0)
          ((not (pair? x)) 1)
          (else (+ (count-leaves (car x)) (count-leaves (cdr x))))
    )
)
```
### ex2.24
(1 (2 (3 4)))
### ex2.25
```scheme
(define a (list 1 3 (list 5 7) 9))
(cdr (car (cdr (cdr a))))
(define b (list (list 7)))
(car (car b))
(define c (list 1 (list 2 (list 3 (list 4 (list 5 (list 6 7)))))))
(car (cdr (car (cdr (car (cdr (car (cdr (car (cdr (car (cdr c))))))))))))
```
### ex2.26
```scheme
(1 2 3 4 5 6)
((1 2 3) 4 5 6)
((1 2 3) (4 5 6))
```
### ex2.27
```scheme
(define (reverse item)
    (if (null? (cdr item))
        item
        (append (reverse (cdr item)) (list (car item)))
    )
)
(define (deep-reverse item)
    (if (null? item)
        nil
        (if (not (pair? item))
            item
            (append (deep-reverse (cdr item)) (list (deep-reverse (car item)))) 
        )
    )
)
```
### ex2.28
```scheme
(define (fringe item)
    (if (null? item)
        nil
        (if (not (pair? item))
            (list item)
            (append (fringe (car item)) (fringe (cdr item)))
        )
    )
)
```
### ex2.29
```scheme
(define (make-mobile left right)
    (list left right)
)
(define (make-branch length structure)
    (list length structure)
)
; a
(define (left-branch mobile)
    (car mobile)
)
(define (right-branch mobile)
    (cadr mobile)
)
(define (branch-length branch)
    (car branch)
)
(define (branch-structure branch)
    (cadr branch)
)
; b
(define (total-weight mobile)
    (define (branch-temp branch)
        (if (not (pair? (branch-structure branch)))
            (branch-structure branch)
            (+ (branch-temp (left-branch (branch-structure branch))) (branch-temp (right-branch (branch-structure branch))))
        )
    )
    (+ (branch-temp (left-branch mobile)) (branch-temp (right-branch mobile)))
)
(define a (make-branch 10 20))
(define b (make-branch 10 30))
(define c (make-mobile a b))
(define d (make-branch 10 c))
(define e (make-mobile d b))
(total-weight e) ;test passed
; c
(define (l-length mobile)
    (branch-length (left-branch mobile))
)
(define (l-structure mobile)
    (branch-structure (left-branch mobile))
)
(define (r-length mobile)
    (branch-length (right-branch mobile))
)
(define (r-structure mobile)
    (branch-structure (right-branch mobile))
)
(define (is-mobile? structure)
    (if (pair? structure)
        #t; is mobile
        #f; is weight
    )
)
(define (balanced-mobile mobile)
    (cond ((and (not (is-mobile? (l-structure mobile))) (not (is-mobile? (r-structure mobile))))
            (and (= (* (l-length mobile) (l-structure mobile)) (* (r-length mobile) (r-structure mobile)))))
          ((and (not (is-mobile? (l-structure mobile))) (is-mobile? (r-structure mobile)))
            (and (= (* (l-length mobile) (l-structure mobile)) (* (r-length mobile) (total-weight (r-structure mobile)))) (balanced-mobile (r-structure mobile))))
          ((and (is-mobile? (l-structure mobile)) (not (is-mobile? (r-structure mobile))))
            (and (= (* (l-length mobile) (total-weight (l-structure mobile))) (* (r-length mobile) (r-structure mobile))) (balanced-mobile (l-structure mobile))))
          ((and (is-mobile? (l-structure mobile)) (is-mobile? (r-structure mobile)))
            (and (= (* (l-length mobile) (total-weight (l-structure mobile))) (* (r-length mobile) (total-weight (r-structure mobile)))) (balanced-mobile (l-structure mobile)) (balanced-mobile (r-structure))))    
    )
)
; d
; if I want to change the representations of mobiles:
; I only should change cadr to cdr, nothing left.
```
### mapping over trees
```scheme
(define (scale-tree tree factor)
    (cond ((null? tree) nil)
          ((not (pair? tree) (* tree factor)))
          (else (cons (scale-tree (car tree) factor) (scale-tree (cdr tree) factor)))
    )
)
; another way to build scale-tree is to regard it as many combinations of the sub-tree.
(define (scale-tree tree factor)
    (map (lambda (sub-tree) 
         (if (pair? sub-tree)
            (scale-tree sub-tree factor)
            (* sub-tree factor)
         )) tree)
)
```
### ex2.30
```scheme
(define (square x) (* x x))
(define (square-tree tree)
    (cond ((null? tree) nil)
          ((not (pair? tree)) (square tree))
          (else (cons (square-tree (car tree)) (square-tree (cdr tree))))
    )
)
(define (square-tre tree)
    (map (lambda (sub-tree)
         (if (pair? sub-tree)
             (square-tre sub-tree)
             (square sub-tree)
         )
         ) tree)
)
(define a (list 1 (list 2 (list 3 4) 5) (list 6 7)))
(square-tree a)
(square-tre a)
```
### ex2.31
```scheme
(define (tree-map proc tree)
    (cond ((null? tree) nil)
          ((not (pair? tree)) (proc tree))
          (else (cons (tree-map proc (car tree)) (tree-map proc (cdr tree))))
    )
)
```
### ex2.32
```scheme
(define (subsets s)
    (if (null? s)
        (list nil)
        (let ((rest (subsets (cdr s))))
            (append rest (map (lambda (entry) (append entry (list (car s)))) rest))
        )
    )
)
(subsets (list 1 2 3 4))
```

### 2.2.3
concentrate on the "signals".
```scheme
(define (filter predicate sequence)
    (cond ((null? sequence) nil)
          ((predicate (car sequence)) (cons (car sequence) (filter predicate (cdr sequence))))
          (else (filter predicate (cdr sequence)))
    )
)
(define (accumulate op initial sequence)
    (if (null? sequence)
        initial
        (op (car sequence) (accumulate op initial (cdr sequence)))
    )
)
(define (enumerate-interval low high)
    (if (> low high)
        nil
        (cons low (enumerate-interval (+ low 1) high))
    )
)
;use these functions to build other complicated things.
```
1979: Richard Waters: 90% Fortran programs have similar procedure.

### ex2.33
```scheme
(define (map p sequence)
    (accumulate (lambda (x y) (cons (p x) y)) nil sequence)
)
(define (append seq1 seq2)
    (accumulate cons seq2 seq1)
)
(define (length sequence)
    (accumulate (lambda (x y) (+ 1 y)) 0 sequence)
)
```
### ex2.34
```scheme
(define (horner-eval x coefficient-sequence)
    (accumulate (lambda (this-coeff higher-terms) (+ this-coeff (* x higher-terms)))
        0
        coefficient-sequence
    )
)
```
### ex2.35
```scheme

```
### ex2.36
```scheme
(define (accumulate-n op init seqs)
    (if (null? (car seqs))
        nil
        (cons (accumulate op init (car seqs)) (accumulate-n op init (cdr seqs)))
    )
)
```
