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
sequence your function, do it like module, therefore you can use it multiple times.

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
;(define (count-leaves x)
;    (cond ((null? x) 0)
;          ((not (pair? x)) 1)
;          (else (+ (count-leaves (car x)) (count-leaves (cdr x))))
;    )
;)

;very interesting problem.
(define (count-leaves t)
    (accumulate + 0 (map (lambda (x) 
                            (cond ((not (pair? x)) 1)
                                  ((null? x) 0)
                                  (else (count-leaves x))
                            )) 
    t))
)
```
### ex2.36
```scheme
; 确实没有想到用map，不过这个可能和我忘记先做了2.35有关系。。
(define (accumulate-n op init seqs)
    (if (null? (car seqs))
        nil
        (cons (accumulate op init (map car seqs)) (accumulate-n op init (map cdr seqs)))
    )
)
(define a (list (list 1 2 3) (list 4 5 6) (list 7 8 9) (list 10 11 12)))
(accumulate-n + 0 a)
```
### ex2.37
```scheme
;matrix representation: ((1 2 3 4) (4 5 6 6) (6 7 8 9))
(define (dot-product v w)
    (accumulate + 0 (map * v w))
)
(define (matrix-*-vector m v)
    (map (lambda (line) (accumulate + 0 (map * line v))) m)
)
(define (matrix-*-matrix m n)
    (let ((cols (transpose n)))
        (map (lambda (line) (accumulate cons nil (matrix-*-vector cols line))) m)
    )
)
(define (transpose mat)
    (accumulate-n cons nil mat)
)
; test part!

(define v (list (1 2 3 4))
(define w (list (1 2 3 4))
(define m (list (list 1 2 3 4) (list 4 5 6 6) (list 6 7 8 9)))
(dot-product v w)
(matrix-*-vector m v)

(define n (list (list 1 5 1) (list 2 6 1) (list 3 7 1) (list 4 8 1)))
(matrix-*-matrix m n)
;Interesting!
```
### ex2.38
```scheme
(define (fold-left op initial sequence)
    (define (iter result rest)
        (if (null? rest)
            result
            (iter (op result (car test)) (cdr test))
        )
    )
    (iter initial sequence)
)
(define (accumulate op initial sequence)
    (if (null? sequence)
        initial
        (op (car sequence) (accumulate op initial (cdr sequence)))
    )
)
(accumulate / 1 (list 1 2 3))
(fold-left / 1 (list 1 2 3))
(accumulate list nil (list 1 2 3))
(fold-left list nil (list 1 2 3))
;The key is your 'operaion' is commutative. eg. + *. Then these two will be the same.
```
### ex2.39
```scheme
;assume 'fold-right' is same as accumulate
(define (reverse sequence)
    (accumulate (lambda (x y) (append y (list x))) nil sequence)
)
(define (reverse-v2 sequence)
    (fold-left (lambda (x y) (append (list y) x)) nil sequence)
)
(reverse (list 1 2 3 4 5))
(reverse-v2 (list 1 2 3 4 5))
;test done!
```
Nested Mappings:

```scheme
(accumulate append nil (map (lambda (i) (map (lambda (j) (list i j)))) (enumerate-interval 1 n)))
(define (prime-sum? pair)
    (prime? (+ (car pair) (cadr pair)))
)
```

### ex2.40
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
(define (unique-pairs n)
    (accumulate append nil (map (lambda (i) (map (lambda (j) (list i j)) (enumerate-interval 1 (- i 1)))) (enumerate-interval 1 n)))
)
(define (prime-sum-pairs n)
    (map (lambda (x) (list (car x) (cadr x) (+ (car x) (cadr x)))) (filter (lambda (x) (prime? (+ (car x) (cadr x)))) (unique-pairs n)))
)
```
### ex2.41
```scheme
(define (order-triple n s)
    (filter (lambda (triple) (= s (+ (car triple) (cadr triple) (car (cdr (cdr triple)))))) (accumulate append nil (accumulate append nil (map (lambda (i) (map (lambda (j) (map (lambda (k) (list i j k)) (enumerate-interval 1 n))) (enumerate-interval 1 n))) (enumerate-interval 1 n)))))
)
```
### ex2.42
eight-queens puzzle:
```scheme
(define (flatmap proc seq)
    (accumulate append nil (map proc seq))
)
(define (length list)
    (if (null? list)
        0
        (+ 1 (length (cdr list)))
    )
)
(define (adjoin-position new-row k rest-of-queens)
    (if (= (- k 1) (length rest-of-queens))
        (append rest-of-queens (list new-row))
        #f
    )
)
(define (find k positions)
    (if (= 1 k)
        (car positions)
        (find (- k 1) (cdr positions))
    )
)
(define (safe? k positions)
    (define new-row (find k positions))
    (define (iter positions)
        (if (null? (cdr positions))
            #t
            (if (= (car positions) new-row)
                #f
                (iter (cdr positions))
            )
        )
    )
    (iter positions)
)
(define empty-board nil)
(define (queens board-size)
    (define (queen-cols k)
        (if (= k 0)
            (list empty-board)
            (filter (lambda (positions) (safe? k positions)) (flatmap (lambda (rest-of-queens) (map (lambda (new-row) (adjoin-position new-row k rest-of-queens)) (enumerate-interval 1 board-size))) (queen-cols (- k 1))))
        )
    )
    (queen-cols board-size)
)
(define a (list 1 2 3 4 6))
(find 5 a)
(safe? 5 a)
(adjoin-position 7 6 a)
(queens 4);test passed
```
### ex2.43
Too many recursion! Of course slow!

### ex2.44
```scheme
(define (up-split painter n)
    (if (= n 0)
        painter
        (let ((smaller (up-split painter (- n 1))))
            (below painter (besides smaller smaller))
        )
    )
)
```
### ex2.45
```scheme
(define (split procedure1 procedure2)
    (lambda (painter n)
        (procedure1 painter (procedure2 ((split procedure1 procedure2) painter (- n 1)) ((split procedure1 procedure2) painter (- n 1))))
    )
)
```
### Frames
```scheme
;assume the vector is on a unit circle.
(define (frame-coord-map frame)
    (lambda (v)
        (add-vect (origin-frame frame) (add-vect (scale-vect (xcor-vect v) (edge1-frame frame)) (scale-vect (ycor-vect v) (edge2-frame frame))))
    )
)
```
### ex2.46
```scheme
(define (make-vect x y)
    (cons x y)
)
(define (xcor-vect vect)
    (make-vect (car vect) 0)
)
(define (ycor-vect vect)
    (make-vect 0 (cadr vect))
)
(define (add-vect vect1 vect2)
    (make-vect (+ (car vect1) (car vect2)) (+ (cadr vect1) (cadr vect2)))
)
(define (sub-vect vect1 vect2)
    (make-vect (- (car vect1) (car vect2)) (- (cadr vect1) (cadr vect2)))
)
(define (scale-vect vect scale)
    (make-vect (* (car vect) scale) (* (cadr vect) scale))
)
```
### ex2.47
```scheme
;for the first case
(define (make-frame origin edge1 edge2)
    (list origin edge1 edge2)
)
(define (origin-frame frame)
    (car frame)
)
(define (edge1-frame frame)
    (cadr frame)
)
(define (edge2-frame frame)
    (cadr (cdr frame))
)
;for the second case
(define (make-frame origin edge1 edge2)
    (cons origin (cons edge1 edge2))
)
(define (origin-frame frame)
    (car frame)
)
(define (edge1-frame frame)
    (cadr frame)
)
(define (edge2-frame frame)
    (cadr (cdr frame))
)
;Remember, edge1 and edge2 are vectors, so when using frame, we can simply call them.
```
### painters
if we have a procedure draw-line that draws a line on the screen between two specified points, then we can create painters for line drawings.
```scheme
;segment is the side of a graph.
(define (segments->painter segment-list)
    (lambda (frame)
        (for-each (lambda (segment) (draw-line ((frame-coord-map frame) (start-segment segment)) ((frame-coord-map frame) (end-segment segment)))) segment-list)
    )
)
```
### ex2.48
```scheme
(define (make-segment vect1 vect2)
    (cons vect1 vect2)
)
(define (start-segment segment)
    (car segment)
)
(define (end-segment segment)
    (cadr segment)
)
```
### ex2.49
```scheme
; All the procedure will use the same frame given as myframe.
; a: the painter that draws the outline of the designated frame
; recall that edge1, edge2 are all vectors in definition.
; frame origin, edge1, edge2. The offset of the origin will be taken into consideration in frame-coord-map procedure.
; we assume we get edge1 edge2 origin beforehand.
(define myframe (make-frame origin edge1 edge2))
(define segment1 (make-segment (make-vector 0 0) edge1))
(define segment2 (make-segment (make-vector 0 0) edge2))
(define segment3 (make-segment edge1 (add-vector edge1 edge2)))
(define segment4 (make-segment edge2 (add-vector edge1 edge2)))
; The drawing process was like that:
;        ^ - - - >
;        |       ^
;        |       |
;        | _ _ _ >
;     origin point of the frame
(define outline
    (segments->painter (list segment1 segment2 segment3 segment4))
)

; b: the painter that draws an "X" by connecting opposite corners of the frame
; We've already get the frame, so we just need to lock the points for injection bars into the frame.
(define segment5 (make-segment (make-vector 0 0) (add-vector edge1 edge2)))
(define segment6 (make-segment edge1 edge2))
(define X
    (segments->painter (list segment5 segment6))
)

; c: the painter that draws a diamond shape by connecting the mid-points of the sides of the frame.
(define segment7 (make-segment (scale-vect edge1 0.5) (scale-vect edge2 0.5)))
(define segment8 (make-segment (scale-vect edge1 0.5) (add-vect edge1 (scale-vect edge2 0.5))))
(define segment9 (make-segment (scale-vect edge2 0.5) (add-vect edge2 (scale-vect edge1 0.5))))
(define segment10 (make-segement (add-vect edge1 (scale-vect edge2 0.5)) (add-vect edge2 (scale-vect edge1 0.5))))
(define diamond
    (segments->painter (list segment7 segment8 segment9 segment10))
)

; d: the wave painter?
; Are you kidding me? basic idea must be transfer it into many line to mimic arc structures in the graph.
; But it is so troublesome. I skip!!
```
### transforming and combining painters
flip-vert doesn't have to know how a painter works in order to flip it, it just has to know how to turn a frame upside down: the flipped painter just uses the original painter, but in the inverted frame.
```scheme
(define (transform-painter painter origin corner1 corner2)
    (lambda (frame)
        (let ((m (frame-coord-map frame)))
            (let ((new-origin (m origin)))
                (painter (make-frame new-origin (sub-vect (m corner1) new-origin) (sub-vect (m corner2) new-origin)))
            )
        )
    )
)
; very interesting, to change the frame structure to flip-vert.
(define (flip-vert painter)
    (transform-painter painter (make-vect 0.0 1.0) (make-vect 1.0 1.0) (make-vect 0 0))
)
(define (shrink-to-upper-right painter)
    (transform-painter painter (make-vect 0.5 0.5) (make-vect 1.0 0.5) (make-vect 0.5 1.0))
)
(define (rotate90 painter)
    (transform-painter painter (make-vect 1.0 0.0) (make-vect 1.0 1.0) (make-vect 0.0 0.0))
)
(define (squash-inwards painter)
    (transform-painter painter (make-vect 0.0 0.0) (make-vect 0.65 0.35) (make-vect 0.35 0.65))
)
(define (beside painter1 painter2)
    (let ((split-point (make-vect 0.5 0.0)))
        (let ((paint-left (transform-painter painter1 (make-vect 0.0 0.0) split-point (make-vect 0.0 1.0)))
              (paint-right (transform-painter painter2 split-point (make-vect 1.0 0) (make-vect 0.5 1.0)))
             )
             (lambda (frame) (paint-left frame) (paint-right frame))     
        )
    )
)
```
### ex2.50
```scheme
(define (flip-horiz painter)
    (transform-painter painter (make-vect 1.0 0) (make-vect 0 0) (make-vect 1.0 1.0))
)
(define (rotate180 painter)
    (transform-painter painter (make-vect 1.0 1.0) (make-vect 0 1.0) (make-vect 1.0 0))
)
(define (rotate270 painter)
    (transform-painter painter (make-vect 0.0 1.0) (make-vect 0.0 0.0) (make-vect 1.0 1.0))
)
```
### ex2.51
```scheme
(define (below painter1 painter2)
    (let ((paint-below (transform-painter painter1 (make-vect 0.0 0.0) (make-vect 1.0 0) (make-vect 0.0 0.5)))
          (paint-up (transform-painter painter2 (make-vect 0.0 0.5) (make-vect 1.0 0.5) (make-vect 0.0 1.0))))
        (lambda (frame) (paint-below frame) (paint-up frame))
    )
)
```

### levels of language for robust design
stratified design: the notion that a complex system should be structured as a sequence of levels that are described using a sequence of languages. Each level is constructed by combining parts that are regarded as primitive at that level, and the parts constructed at each level are used as primitives at the next level.

stratified design helps make programs robust, that is, it makes it likely that small changes in a specification will require correspondingly small changes in the program.

I want to skip ex2.52, No need to do..

## 2.3: Symbolic Data
```scheme
(define (memq item x)
    (cond ((null? x) false)
          ((eq? item (car x)) x)
          (else (memq item (cdr x)))
    )
)
```
### ex2.53
```scheme
(a b c)
((george))
((y1 y2))
(y1 y2)
#f
#f
(red shoes blue socks)
```
### ex2.54
```scheme
(define (equal? list1 list2)
    (cond  ((and (null? list1) (null? list2)) #t)
           ((eq? (car list1) (car list2)) (equal? (cdr list1) (cdr list2)))
           (else #f) 
    )
)
```
### ex2.55
```scheme
; ``abcde --> `abcde
; (car ``abcde) --> (car (list ` a b c d e)) ---> `:quote
```

### 2.3.2 symbolic differentiation
using some wishful thinking method.
```scheme
;Thinking:
;
;(variable? e): is e a variable?
;(same-variable? v1 v2): are v1 and v2 the same variable?
;(sum? e): is e a sum?
;(addend e): addend of the sum e
;(augend e): augend of the sum e
;(make-sum a1 a2): construct the sum of a1 and a2
;(product? e): is e a product?
;(multiplier e): Multiplier of the product e
;(multiplicand e): Multiplicand of the product e
;(make-product m1 m2): construct the product of m1 and m2
; ex2.56: add exponentiation
;(exponentiation? e): is e an exponentiation
;(base x): base of the exponentiation
;(exponent x): exponent of exponentiation
;(make-exponentiation v1 v2): construct the exponentiation of v1 and v2
(define (deriv exp var)
    (cond ((number? exp) 0)
          ((variable? exp) (if (same-variable? exp var) 1 0))
          ((sum? exp) (make-sum (deriv (addend exp) var) (deriv (augend exp) var)))
          ((product? exp) (make-sum (make-product (multiplier exp) (deriv (multiplicand exp) var)) (make-product (deriv (multiplier exp) var) (multiplicand exp))))
          ; ex2.56 ((exponentiation? exp) (make-product (make-product (exponent exp) (make-exponentiation (base exp) (- (exponent exp) 1))) (deriv (base exp) var)))
          (else (error "unknown expression type - DERIV" exp))
    )
)
(define (variable? x) (symbol? x))
(define (same-variable? v1 v2)
    (and (variable? v1) (variable? v2) (eq? v1 v2))
)
(define (sum? x)
    (and (pair? x) (eq? (car x) `+))
)
(define (addend s) (cadr s))
;(define (augend s) (caddr s))
(define (augend s)
    (if (= (length (cddr s)) 1)
        (caddr s)
        (cons `+ (cddr s))
    )
)
(define (product? x)
    (and (pair? x) (eq? (car x) `*))
)
(define (multiplier p)
    (cadr p)
)
(define (multiplicand p)
    (if (= (length (cddr p)) 1) 
        (caddr p)
        (cons `* (cddr p))
    )
)
(define (make-sum a1 a2)
    (cond ((=number? a1 0) a2)
          ((=number? a2 0) a1)
          ((and (number? a1) (number? a2)) (+ a1 a2))
          (else (list `+ a1 a2))
    )
)
(define (=number? exp num)
    (and (number? exp) (= exp num))
)
(define (make-product m1 m2)
    (cond ((or (=number? m1 0) (=number? m2 0)) 0)
          ((=number? m1 1) m2)
          ((=number? m2 1) m1)
          ((and (number? m1) (number? m2)) (* m1 m2))
          (else (list `* m1 m2))
    )
)
;ex2.56: exponentiation
(define (exponentiation? x)
    (and (pair? x) (eq? (car x) `**))
)
(define (base x)
    (cadr x)
)
(define (exponent x)
    (caddr x)
)
(define (power v1 v2)
    (if (= v2 0)
        1
        (* v1 (power v1 (- v2 1)))
    )
)
(define (make-exponentiation v1 v2)
    (cond ((=number? v2 0) 1)
          ((=number? v2 1) v1)
          ((and (number? v1) (number? v2)) (power v1 v2))
          (else (list `** v1 v2))
    )
)
```

### ex2.58
```scheme
;a
(define (deriv exp var)
    (cond ((number? exp) 0)
          ((variable? exp) (if (same-variable? exp var) 1 0))
          ((sum? exp) (make-sum (deriv (addend exp) var) (deriv (augend exp) var)))
          ((product? exp) (make-sum (make-product (multiplier exp) (deriv (multiplicand exp) var)) (make-product (multiplicand exp) (deriv (multiplier exp) var))))
          (else (error "unknown expression type" exp))
    )
)
(define (variable? x) (symbol? x))
(define (same-variable? v1 v2)
    (and (variable? v1) (variable? v2) (eq? v1 v2))
)
(define (sum? x)
    (and (pair? x) (eq? (cadr x) `+))
)
(define (addend x)
    (car x)
)
(define (augend x)
    (caddr x)
)
(define (=number? x num)
    (and (number? x) (= exp num))
)
(define (make-sum x1 x2)
    (cond ((=number? x1 0) x2)
          ((=number? x2 0) x1)
          ((and (number? x1) (number? x2)) (+ x1 x2))
          (else (list x1 `+ x2))
    )
)
(define (product? x)
    (and (pair? x) (eq? (cadr x) `*))
)
(define (multiplier x)
    (car x)
)
(define (multiplicand x)
    (caddr x)
)
(define (make-product x1 x2)
    (cond ((or (=number? x1 0) (=number? x2 0)) 0)
          ((=number? x1 1) x2)
          ((=number? x2 1) x1)
          ((and (number? x1) (number? x2)) (* x1 x2))
          (else (list x1 `* x2))
    )
)
(deriv `(x + (3 * (x + (y + 2)))) `x)
;b
(define (augend x)
    (if (= (length (cddr x)) 1)
        (caddr x)
        (cddr x)
    )
)
(define (multiplicand x)
    (if (= (length (cddr x)) 1)
        (caddr x)
        (cddr x)
    )
)
```
### 2.3.3
A set is simply a collection of distinct objects.
```scheme
;judge a element is in the set.
(define (element-of-set? x set)
    (cond ((null? set) false)
          ((equal? x (car set)) true)
          (else (element-of-set? x (cdr set)))
    )
)
;insert a element into the set.
(define (adjoin-set x set)
    (if (element-of-set? x set)
        set
        (cons x set)
    )
)
;find the intersection-set of set1 and set2
(define (intersection-set set1 set2)
    (cond ((or (null? set1) (null? set2)) `())
          ((element-of-set? (car set1) set2) (cons (car set1) (intersection-set (cdr set1) set2)))
          (else (intersection-set (cdr set1) set2))
    )
)
```
### ex2.59
```scheme
(define (union-set set1 set2)
    (cond ((and (null? set1) (null? set2)) `())
          ((null? set2) set1)
          ((null? set1) set2)
          ((element-of-set? (car set1) set2) (union-set (cdr set1) set2))
          (else (cons (car set1) (union-set (cdr set1) set2)))
    )
)
(union-set (list 1 2 3 4) (list 1 4 6 7))
```
### ex2.60
```scheme
;allow duplicates
;judge an element is in the set.
(define (element-of-set? x set)
    (cond ((null? set) false)
          ((equal? x (car set)) true)
          (else (element-of-set? x (cdr set)))
    )
)
;insert a element into the set.
(define (adjoin-set x set)
    (cons x set)
)
;find the intersection-set of set1 and set2
(define (intersection-set set1 set2)
    (cond ((or (null? set1) (null? set2)) `())
          ((element-of-set? (car set1) set2) (append (list (car set1) (car set1)) (intersection-set (cdr set1) set2)))
          (else (intersection-set (cdr set1) set2))
    )
)
(define (union-set set1 set2)
    (append set1 set2)
)
;quicker.
```
### sets are ordered lists
```scheme
(define (element-of-set? x set)
    (cond ((null? set) false)
          ((= x (car set)) true)
          ((< x (car set)) false)
          (else (element-of-set? x (cdr set)))
    )
)
(define (intersection-set set1 set2)
    (if (or (null? set1) (null? set2))
        `()
        (let ((x1 (car set1)) (x2 (car set2)))
             (cond ((= x1 x2) (cons x1 (intersection-set (cdr set1) (cdr set2))))
                   ((< x1 x2) (intersection-set (cdr set1) set2))
                   ((> x1 x2) (intersection-set set1 (cdr set2)))
             )
        )
    )
)
```
### ex2.61
```scheme
(define (adjoin-set x set)
   (define (join-set x set front)
        (if (element-of-set? x set)
            set
            (if (null? set)
                (append front (list x))
                (if (< x (car set))
                    (append front (cons x set))
                    (join-set x (cdr set) (append front (list (car set))))
                )
            )
        )
    )
    (join-set x set nil)
)
(adjoin-set 5 (list 1 2 3 4 5 6))
(adjoin-set 10 (list 1 2 3 4 5 6))
(adjoin-set 7 (list 1 3 5 6 9 10))
```

### ex2.62
```scheme
(define (union-set set1 set2)
    (if (and (null? set1) (null? set2))
        `()
        (if (null? set1)
            set2
            (if (null? set2)
                set1
                (let ((x1 (car set1)) (x2 (car set2)))
                    (cond ((= x1 x2) (cons x1 (union-set (cdr set1) (cdr set2))))
                          ((< x1 x2) (cons x1 (union-set (cdr set1) set2)))
                          ((> x1 x2) (cons x2 (union-set set1 (cdr set2))))
                    )
                )
            )
        )
    )
)
(union-set (list 1 2 3 4 5) (list 3 5 6 9 11))
(union-set (list 1 4 6) nil)
(union-set nil (list 2 5 7))
(union-set nil nil)
```
### sets as binary trees
```scheme
(define (entry tree) (car tree))
(define (left-branch tree) (cadr tree))
(define (right-branch tree) (caddr tree))
(define (make-tree entry left right) (list entry left right))
(define (element-of-set? x set)
    (cond ((null? set) false)
          ((= x (entry set)) true)
          ((< x (entry set)) (element-of-set? x (left-branch set)))
          ((> x (entry set)) (element-of-set? x (right-branch set)))
    )
)
;adjoin an item to a set.
(define (adjoin-set x set)
    (cond ((null? set) (make-tree x `() `()))
          ((= x (entry set)) set)
          ((< x (entry set)) (make-tree (entry set) (adjoin-set x (left-branch set)) (right-branch set)))
          ((> x (entry set)) (make-tree (entry set) (right-branch set) (adjoin-set x (right-branch set))))
    )
)
```

### ex2.63
```scheme
;way one
(define (tree->list-1 tree)
    (if (null? tree)
        `()
        (append (tree->list-1 (left-branch tree)) (cons (entry tree) (tree->list-1 (right-branch tree))))
    )
)
;way two
(define (tree->list-2 tree)
    (define (copy-to-list tree result-list)
        (if (null? tree)
            result-list
            (copy-to-list (left-branch tree) (cons (entry tree) (copy-to-list (right-branch tree) result-list)))
        )
    )
    (copy-to-list tree `())
)
(tree->list-1 (make-tree 123 (make-tree 2 nil nil) (make-tree 24 nil nil)))
(tree->list-2 (make-tree 123 (make-tree 2 nil nil) (make-tree 24 nil nil)))
```
a). same.
b). apparently the second one is better but harder to understand.

### ex2.64
```scheme
(define (list->tree elements)
    (car (partial-tree elements (length elements)))
)
(define (partial-tree elts n)
    (if (= n 0)
        (cons `() elts)
        (let ((left-size (quotient (- n 1) 2)))
             (let ((left-result (partial-tree elts left-size)))
                (let ((left-tree (car left-result))
                      (non-left-elts (cdr left-result))
                      (right-size (- n (+ left-size 1)))
                     )
                     (let ((this-entry (car non-left-elts))
                           (right-result (partial-tree (cdr non-left-elts) right-size))
                          )
                          (let ((right-tree (car right-result))
                                (remaining-elts (cdr right-result))
                               )
                               (cons (make-tree this-entry left-tree right-tree) remaining-elts)
                          )
                     )
                )
             )
        )
    )
)
;a: I draw the process with pen and paper.
;b: O(N) = 2O(N/2) + 2 = N * O(1) + 2log N = O(N)
```
### ex2.65
```scheme
(define (union-set-tree tree-set1 tree-set2)
    (define (union-set set1 set2)
        (if (and (null? set1) (null? set2))
            `()
            (if (null? set1)
                set2
                (if (null? set2)
                    set1
                    (let ((x1 (car set1)) (x2 (car set2)))
                        (cond ((= x1 x2) (cons x1 (union-set (cdr set1) (cdr set2))))
                            ((< x1 x2) (cons x1 (union-set (cdr set1) set2)))
                            ((> x1 x2) (cons x2 (union-set set1 (cdr set2))))
                        )
                    )
                )
            )
        )
    )
    (let ((set1 (tree->list-1 tree-set1))
          (set2 (tree->list-1 tree-set2))
         )
         (union-set set1 set2)
    )
)
;the intersection-set is the same.
```
### sets and information retrieval
```scheme
;in unordered list, procedure lookup can be represented as:
(define (lookup given-key set-of-records)
    (cond ((null? set-of-records) false)
          ((equal? given-key (key (car set-of-records))) (car set-of-records))
          (else (lookup given-key (cdr set-of-records)))
    )
)
```

### ex2.66
```scheme
(define (make-tree entry left right) (list entry left right))
(define (entry tree) (car tree))
(define (left tree) (cadr tree))
(define (right tree) (caddr tree))
(define (lookup given-key set-of-records)
    (cond ((null? tree) false)
          ((= x (key (entry tree))) (entry tree))
          ((< x (key (entry tree))) (lookup given-key (left set-of-records)))
          ((> x (key (entry tree))) (lookup given-key (right set-of-records)))
    )
)
```

### example:huffman encoding trees
huffman encoding trees: variable-length for different codes. The frequently used letter is given shorter code.

example of building a huffman tree, starting from the lowest weight node to build a whole tree.
```scheme
;encode and build procedure.
;build leaf first.
(define (make-leaf symbol weight)
    (list `leaf symbol weight)
)

(define (leaf? object)
    (eq? (car object) `leaf)
)

(define (symbol-leaf x) (cadr x))
(define (weight-leaf x) (caddr x))
;build node
(define (make-code-tree left right)
    (list left right (append (symbols left) (symbols right)) (+ (weight left) (weight right)))
)

(define (left-branch tree) (car tree))
(define (right-branch tree) (cadr tree))

(define (symbols tree)
    (if (leaf? tree)
        (list (symbol-leaf tree))
        (caddr tree)
    )
)
(define (weight tree)
    (if (leaf? tree)
        (weight-leaf tree)
        (cadddr tree)
    )
)
;decoding procedure, simply judge the bits has come to the leaf, then return symbol.
(define (decode bits tree)
    (define (decode-1 bits current-branch)
        (if (null? bits)
            `()
            (let ((next-branch (choose-branch (car bits) current-branch)))
                (if (leaf? next-branch)
                    (cons (symbol-leaf next-branch) (decode-1 (cdr bits) tree))
                    (decode-1 (cdr bits) next-branch)
                )
            )
        )
    )
    (decode-1 bits tree)
)
(define (choose-branch bit branch)
    (cond ((= bit 0) (left-branch branch))
          ((= bit 1) (right-branch branch))
          (else (error "bad bit - CHOOSE-BRANCH" bit))
    )
)
```

### Sets of weighted elements.
```scheme
(define (adjoin-set x set)
    (cond ((null? set) (list x))
          ((< (weight x) (weight (car set))) (cons x set))
          (else (cons (car set) (adjoin-set x (cdr set))))
    )
)
(define (make-leaf-set pairs)
    (if (null? pairs)
        `()
        (let ((pair (car pairs)))
            (adjoin-set (make-leaf (car pair) (cadr pair)) (make-leaf-set (cdr pairs)))
        )
    )
)
```
### ex2.67
```scheme
(define sample-tree
    (make-code-tree (make-leaf `A 4)
                    (make-code-tree (make-leaf `B 2) (make-code-tree (make-leaf `D 1) (make-leaf `C 1)))
    )
)
(define sample-message `(0 1 1 0 0 1 0 1 0 1 1 1 0))
(decode sample-message sample-tree)
;(A D A B B C A)
```
### ex2.68
```scheme
(define (encode message tree)
    (if (null? message)
        `()
        (append (encode-symbol (car message) tree) (encode (cdr message) tree))
    )
)
(define (element-in-set? element set)
    (if (null? set)
        #f
        (if (eq? element (car set))
            #t
            (element-in-set? element (cdr set))
        )
    )
)
; we can utilize the structure of the huffman tree.
; every node stores the symbol below.
(define (encode-symbol symbol tree)
    (define (search symbol tree)
      (if (leaf? tree) `()  
      	(if (element-in-set? symbol (symbols (left-branch tree)))
             (cons 0 (search symbol (left-branch tree)))
               (cons 1 (search symbol (right-branch tree)))     
        )
      ) 
    )
  (if (element-in-set? symbol (symbols tree)) 
       (search symbol tree)  
       (error "try to encode NO exist symbol -- ENCODE-SYMBOL" symbol))
)
```
### ex2.69
```scheme
(define (generate-huffman-tree pairs)
    (successive-merge (make-leaf-set pairs))
)
(define (make-code-tree left right)
    (list left right (append (symbols left) (symbols right)) (+ (weight left) (weight right)))
)
(define (reverse pairs)
    (if (null? (cdr pairs))
        pairs
        (append (reverse (cdr pairs)) (list (car pairs)))
    )
)
(define (successive-merge pairs)
    (define rpairs (reverse pairs))
    (if (null? pairs)
        #f
        (if (null? (cddr pairs))
            (make-code-tree (car pairs) (cadr pairs))
            (make-code-tree (car rpairs) (successive-merge (reverse (cdr rpairs))))
        )
    )
)
```
### ex2.70
```scheme
(define blist (list (list `A 2) (list `BOOM 1) (list `GET 2) (list `JOB 2) (list `NA 16) (list `SHA 16) (list `YIP 9) (list `WAH 1)))
(generate-huffman-tree blist)
; encode might be too boring to execute with these examples.
```

### ex2.71
```scheme
; least: n - 1 code. most: 1
```
### ex2.72
```scheme
; O (log N) or O (1)
; lets do a sum:
; sum it. the average might be O (1);
; cause basically we can judge from the largest entry: sum (1/2) ^ {log N - K} * (log N - k) -> O (1)
```