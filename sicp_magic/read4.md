## 2.4: Multiple Representations for abstract Data
Learn how to cope with data that may be represented in different ways by different parts of a program --> generic procedures.

type tags, data-directed programming.
```
Programs that use complex numbers
--- add-complex sub-complex mul-complex div-complex  ---

Complex-arithmetic package

--------------------------------------------------------
                            |
Rectangular representation  |      Polar representation
--------------------------------------------------------
      List structure and primitive machine arithmetic
```

```scheme
; a + b * i = z
(define (real-part z) (car z))
(define (imag-part z) (cdr z))
(define (magnitude z)
      (sqrt (+ (square (real-part z)) (square (imag-part z))))
)
(define (angle z)
      (atan (imag-part z) (real-part z))
)
(define (make-from-real-imag x y) (cons x y))
(define (make-from-mag-ang r a) (cons (* r (cos a)) (* r (sin a))))
```
Another way is shown below.
```scheme
; x = r cos A, y = r sin A, r = sqrt(x ^ 2 + y ^ 2), A = arctan(y, x)
(define (real-part z)
      (* (magnitude z) (cos (angle z)))
)
(define (imag-part z)
      (* (magnitude z) (sin (angle z)))
)
(define (magnitude z) (car z))
(define (angle z) (cdr z))
(define (make-from-real-imag x y)
      (cons (sqrt (+ (square x) (square y))) (atan y x))
)
(define (make-from-mag-ang r a) (cons r a))
```
The common part to compute basic instructions.
```scheme
(define (add-complex z1 z2) (make-from-real-imag (+ (real-part z1) (real-part z2)) (+ (imag-part z1) (imag-part z2))))
(define (sub-complex z1 z2) (make-from-real-imag (- (real-part z1) (real-part z2)) (- (imag-part z1) (imag-part z2))))
(define (mul-complex z1 z2) (make-from-mag-ang (* (magnitude z1) (magnitude z2)) (+ (angle z1) (angle z2))))
(define (div-complex z1 z2) (make-from-mag-ang (/ (magnitude z1) (magnitude z2)) (- (angle z1) (angle z2))))
```
TO distinguish different data, use type tag as part of each complex number.
```scheme
(define (attach-tag type-tag contents)
      (cons type-tag contents)
)
(define (type-tag datum)
      (if (pair? datum)
            (car datum)
            (error "Bad tagged datum - TYPE-TAG" datum)
      )
)
(define (contents datum)
      (if (pair? datum)
            (cdr datum)
            (error "Bad tagged datum - CONTENTS" datum)
      )
)
(define (rectangular? z)
      (eq? (type-tag z) `rectangular)
)
(define (polar? z)
      (eq? (type-tag z) `polar)
)
; this way, these two representation can be used in the same system.
; modify the above code with tag. Now each generic selector is implemented as a procedure that checks the tag of its argument and calls the appropriate procedure for handling data of that type.

(define (real-part z)
      (cond ((rectangular? z) (real-part-rectangular (contents z)))
            ((polar? z) (real-part-polar (contents z)))
            (else (error "Unknown type - REAL-PART" z))
      )
)
(define (imag-part z)
      (cond ((rectangular? z) (imag-part-rectangular (contents z)))
            ((polar? z) (imag-part-polar (contents z)))
            (else (error "Unknown type - IMAG-PART" z))
      )
)
;;;; still a lot.
```
if no ome programmer knew all the interface procedures or all the representations, how can they deal with large-scale data-base-management systems?

modularizing the system design even further --> data-directed programming.

manage the function like table, or Database, then using put & get function to manage them.
```
(put <op> <type> <item>)
(get <op> <type>)
```
Users can define a collection of procedures, or package, interfaces these to the rest of the system by adding entries to the table that tell the system how to operate on rectangular numbers.

```scheme
(define (install-rectangular-package)
      ;; internal procedures
      (define (real-part z) (car z))
      (define (imag-part z) (cdr z))
      (define (make-from-real-imag x y) (cons x y))
      (define (magnitude z)
            (sqrt (+ (square (real-part z)) (square (imag-part z))))
      )
      (define (angle z)
            (atan (imag-part z) (real-part z))
      )
      (define (make-from-mag-ang r a)
            (cons (* r (cos a)) (* r (sin a)))
      )
      
      ;; interface to the rest of the system
      (define (tag x) (attach-tag `rectangular x))
      (put `real-part `(rectangular) real-part)
      (put `imag-part `(rectangular) imag-part)
      (put `magnitude `(rectangular) magnitude)
      (put `angle `(rectangular) angle)
      (put `make-from-real-imag `rectangular (lambda (r a) (tag (make-from-real-imag x y))))
      (put `make-from-mag-ang `rectangular (lambda (r a) (tag (make-from-mag-ang r a))))
      `done
)
```
the user needn't worry about name conflicts with other procedures outside the rectangular package.

```scheme
(define (apply-generic op . args)
      (let ((type-tags (map type-tag args))) ; get all the tags of args.
           (let ((proc (get op type-tags))) ; get procedure from the database.
                (if proc
                  (apply proc (map contents args)) ; all the args should choose the contents part, omit the type-tag part. Then, we can apply the proc from the base.
                  (error "No method for these types - APPLY-GENERIC" (list op type-tags))
                )
           )
      )
)
(define (real-part z) (apply-generic `real-part z))
(define (imag-part z) (apply-generic `imag-part z))
(define (magnitude z) (apply-generic `magnitude z))
(define (angle z) (apply-generic `angle z))
(define (make-from-real-imag x y) ((get `make-from-real-imag `rectangular) x y))
(define (make-from-mag-ang r a) ((get `make-from-mag-ang `polar) r a))
```

### ex2.73
```scheme
; a: To use the 'deriv' in the function database, 'exp' should be a pair. However, number and variable is not a pair. `deriv' need tag whereas these two don't have it.
; b:
(define (install-sum-package)
      (define (addend s) (cadr s))
      (define (augend s) (caddr s))
      (define (make-sum a1 a2) 
            (cond ((or (=number? m1 0) (=number? m2 0)) 0)
                  ((=number? m1 1) m2)
                  ((=number? m2 1) m1)
                  ((and (number? m1) (number? m2)) (* m1 m2))
                  (else (list `* m1 m2))
            )
      )
      (define (deriv-sum exp var)
            (make-sum (deriv (addend exp) var) (deriv (augend exp) var))
      )
      (define (tag x) (attach-tag `+ x))
      
      (put `make-sum `+ (lambda (x y) (tag (make-sum x y))))
      (put `deriv `(+) deriv-sum)
)
(define (install-product-package)
      (define (multiplier x) (cadr x))
      (define (multiplicand x) (caddr x))
      (define (make-sum a1 a2) 
            (cond ((or (=number? m1 0) (=number? m2 0)) 0)
                  ((=number? m1 1) m2)
                  ((=number? m2 1) m1)
                  ((and (number? m1) (number? m2)) (* m1 m2))
                  (else (list `* m1 m2))
            )
      )
      (define (make-product x1 x2)
            (cond ((or (=number? x1 0) (=number? x2 0)) 0)
                  ((=number? x1 1) x2)
                  ((=number? x2 1) x1)
                  ((and (number? x1) (number? x2)) (* x1 x2))
                  (else (list x1 `* x2))
            )
      )
      (define (deriv-product exp var)
            (make-sum (make-product (multiplier exp) (deriv (multiplicand exp) var)) (make-product (multiplicand exp) (deriv (multiplier exp) var)))
      )
      (define (tag x) (attach-tag `* x))

      (put `make-product `* (lambda (x y) (tag (make-product x y))))
      (put `make-sum `* (lambda (x y) (tag (make-sum x y))))
      (put `deriv `(*) deriv-product)
)
; c. add exponents
(define (install-exponent-package)
      (define (base x) (cadr x))
      (define (exponent x) (caddr x))
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
      (define (deriv-exponentiation v1 v2)
            (make-product (make-product (exponent exp) (make-exponentiation (base exp) (- (exponent exp) 1))) (deriv (base exp) var))
      )
      (define (tag x) (attach-tag `** x))

      (put `make-exponentiation `** (lambda (x y) (tag (make-exponentiation x y))))
      (put `power `(**) (lambda (x y) (tag (power x y))))
      (put `deriv `(**) deriv-exponentiation)
)
;d: little change for get and put usage.
```
### ex2.74
```scheme
; background:
; Each division's personnel records consist of a single file, which contains a set of records keyed on employees' names.
; a record in division file ---> a single file keyed with some employees' names.
; a record of employee ---> a set that contains information keyed under identifiers such as address and salary.

; a: implement for headquarters a get-record procedure that retrieves a specified employee's recod from a specified personnel file.
; tag: get the tag part of sth.
(define (get-record employee file)
      ((get `decode `(tag file)) ((get `record `(tag file)) file) employee)
)
;(define (record file)) ;get record part of file.
;(define (decode record employee)) ; get the employee's own message.

; b: get-salary.
(define (get-salary employeerecord employee)
      ((get `salary `(tag employee)) employeerecord)
)
(define (get-address employeerecord employee)
      ((get `address `(tag employee)) employeerecord)
)

; c: find-employee-record-procedure
(define (find-employee-record employee files)
      (if (pair? files)
            (cons (get-record employee (car files)) (find-employee-record employee (cdr files)))
            (get-record employee files)
      )
)

; d: change in tags.
```
### Message passing.
An alternative implementation strategy is to decompose the talbe into columns and instead of using "intelligent operations", we can dispatch on operation names.
```scheme
(define (make-from-real-imag x y)
      (define (dispatch op)
            (cond ((eq? op `real-part) x)
                  ((eq? op `imag-part) y)
                  ((eq? op `magnitude) (sqrt (+ (square x) (square y))))
                  ((eq? op `angle) (atan x y))
                  (else (error "Unknonw op - MAKE-FORM-REAL-IMAG" op))
            )
      )
      dispatch
)
```
### ex2.75
(define (make-from-mag-ang magnitude angle)
      (define (dispatch op)
            (cond ((eq? op `real-part) (* (cos angle) magnitude))
                  ((eq? op `imag-part) (* (sin angle) magnitude))
                  ((eq? op `magnitude) magnitude)
                  ((eq? op `angle) angle)
                  (else (error "Unknown op - MAKE-FROM-MAG-ANG" op))
            )
      )
      dispatch
)
### ex2.76
for new types, message-passing and data-directed style is good. Whereas, new operations are easier to be added using explicit dispatch.

## 2.5: systems with generic operations
Use data-directed techniques to construct a package of arithmetic operations that incorporates all the arithmetic packages we have already constructed.

Expand the complex number arithemtic to ordinary numbers computing.

```scheme
(define (install-scheme-number-package)
      (define (tag x)
            (attach-tag `scheme-number x)
      )
      (put `add `(scheme-number scheme-number) (lambda (x y) (tag (+ x y))))
      (put `sub `(scheme-number scheme-number) (lambda (x y) (tag (- x y))))
      (put `mul `(scheme-number scheme-number) (lambda (x y) (tag (* x y))))
      (put `div `(scheme-number scheme-number) (lambda (x y) (tag (/ x y))))
      (put `make `(scheme-number scheme-number) (lambda (x) (tag x)))
      `done
)
(define (make-scheme-number n) ((get `make `scheme-number) n))
;
; --> complex --> rectangular --> 3 4
;
```

### ex2.77
of course it works. No complex tag is inserted before these four 'put' procedure.

2 times apply-generic called
complex:(magnitude z) -> polar:(magnitude z) -> (magnitude z): (sqrt ...)

### ex2.78
```scheme
(define (attach-tag type-tag contents)
      (if (number? contents)
            contents
            (cons type-tag contents)
      )
)
(define (type-tag datum)
      (if (number? datum)
            `scheme-number
            (if (pair? datum)
                  (car datum)
                  (error "False." datum)
            )
      )
)
(define (contents datum)
      (if (number? datum)
            datum
            (if (pair? datum)
                  (cadr datum)
                  (error "False." datum)
            )
      )
)
```
### ex2.79 & ex2.80
```scheme
(define (install-scheme-number-package)
      (put `equ? `(scheme-number scheme-number) =)
      (put `=zero? `(scheme-number) (lambda (x) (if (= x 0) #t #f)))
`done)

(define (install-rational-number-package)
      (define (tolerance) 0.000001)
      (put `equ? `(rational-number rational-number) (lambda (x1 x2) (if (< (abs (- x1 x2)) tolerance) #t #f)))
      (put `=zero? `(rational-number) (lambda (x) (if (< (abs x) tolerance) #t #f)))
`done)

(define (install-complex-number-package)
      (define (tolerance) 0.000001)
      (put `equ? `(complex-number complex-numver) (lambda (x1 x2) 
                                                      (if (and (< (abs (- (real x1) (real x2))) tolerance) (< (abs (- (img x1) (img x2))) tolerance))
                                                            #t
                                                            #f
                                                      )))
      (put `=zero? `(complex-number) (lambda (x) (if (and (< (abs (real x)) tolerance) (< (abs (img x)) tolerance)) #t #f)))
)
```
### 2.5.2: combining Data of Different Types
It is meaningful to define operations that cross the type boundaries.

One way to handle cross-type operations is to design a different procedure for each possible combination of types for which the operation is valid. But it is cumbersome.

A better way might be coercion. We can arithmetically combine an ordinary number with a complex number, we can view the ordinary number as a complex number whose imaginary part is zero.


```scheme
(define (scheme-number->complex n)
      (make-complex-from-real-imag (contents n) 0)
)
(put-coercion `scheme-number `complex scheme-number->complex)

(define (apply-generic op . args) ;; args can be seen as a list.
      (let ((type-tags (map type-tag args)))
            (let ((proc (get op type-tags))) ;; find the corresponding procedure.
                  (if proc ;; this procedure exists.
                        (apply proc (map contents args)) ;; get the type of the args.
                        (if (= (length args) 2) ;; Two args.
                              (let ((type1 (car type-tags))
                                    (type2 (cadr type-tags))
                                    (a1 (car args))
                                    (a2 (cadr args)))
                                   ;(if (eq? type1 type2)
                                         ;(apply-generic op a1 a2)  
                                          (let ((t1->t2 (get-coercion type1 type2))
                                                (t2->t1 (get-coercion type2 type1)))
                                                (cond (t1->t2 (apply-generic op (t1->t2 a1) a2)) ;; depends on whether the base has these transfer method.
                                                      (t2->t1 (apply-generic op a1 (t2->t1 a2)))
                                                      (else (error "No method for these types" (list op type-tags)))
                                                )
                                          )
                                    ;)
                              )
                              (error "No method for these types" (list op type-tags))
                        )
                  )
            )
      )
)
```
### ex2.81
```scheme
(define (scheme-number->scheme-number n) n)
(define (complex->complex z) z)
(put-coercion `scheme-number `scheme-number scheme-number->scheme-number)
(put-coercion `complex `complex complex->complex)
; a:
; when louis's coercion procedures installed, nothing strange will happen. Simply do like t1->t2.
; convert x and y to complex or scheme-number beforhand, then call expt to get the result.

; b:
; Unfortunately, this procedure can not be omitted, Or error will happens because the coercion table don't have these elements.

; c:
; (added as comment above.)
```
### ex2.82
```scheme
(define (apply-generic op . args)
      (define (type-tags args)
            (map type-tag args)
      )
      (define (iterate pivot)
            (if (null? pivot)
                  (error "No arguments are given" args)
                  (let (pivot-type (type-tag (car pivot)))
                        (let (
                              (proc 
                                    (get op 
                                          (type-tags 
                                                (map 
                                                      (lambda (x) 
                                                            (let (coercion (get-coercion (type-tag x) pivot-type))
                                                                  (if coercion
                                                                        (coercion x)
                                                                        x
                                                                  )
                                                            )
                                                      ) 
                                                args)
                                          )
                                    )
                              ))
                              (if proc
                                    (apply proc (map contents args))
                                    (iterate (cdr pivot))
                              )
                        )
                  )
            )
      )
      (let ((proc (get op type-tags)))
            (if proc
                  (apply proc (map contents args)) ; proc exists -> same type -> apply this procedure..
                  (iterate args)
            )
      )
)
```
### ex2.83
```scheme
; complex <- real <- rational <- integer
; design a procedure that raises objects of that type one level in the tower.
; show how to install a generic raise operation that will work for each type.
(put `raise (list `integer `rational) integer->rational) ; integer->rational for example
(define (raise type1 type2)
      (get `raise (list type1 type2)) ; raise from type1 to type2.
)
```
### ex2.84
```scheme
; modify the raise operation.
; modify the apply-generic procedure so that it coerces its arguments to have the same type by the method of successive raising, as discussed in this section.
; The raise order will depend solely on raise function exists.

(define (apply-generic op . args) ;; args can be seen as a list.
      (let ((type-tags (map type-tag args)))
            (let ((proc (get op type-tags))) ;; find the corresponding procedure.
                  (if proc ;; this procedure exists.
                        (apply proc (map contents args)) ;; get the type of the args.
                        (if (= (length args) 2) ;; Two args.
                              (let ((type1 (car type-tags))
                                    (type2 (cadr type-tags))
                                    (a1 (car args))
                                    (a2 (cadr args)))
                                   ;(if (eq? type1 type2)
                                         ;(apply-generic op a1 a2)  
                                          (let ((t1->t2 (raise type1 type2))
                                                (t2->t1 (raise type2 type1))) ; judge whether this coercion exists.
                                                (cond (t1->t2 (apply-generic op (t1->t2 a1) a2)) ;; depends on whether the base has these transfer method.
                                                      (t2->t1 (apply-generic op a1 (t2->t1 a2)))
                                                      (else (error "No method for these types" (list op type-tags)))
                                                )
                                          )
                                    ;)
                              )
                              (error "No method for these types" (list op type-tags))
                        )
                  )
            )
      )
)
```

### ex2.85
```scheme
; define a procedure called drop to lower the type in the tower of types.
; when we project it and raise the result back to the type we started with, we end up with something equal to what we started with.
; this is the same(;)
; I guess it not so meaningful to write all the procedure and build such a big system..
```

### ex2.86
```scheme
(define (sine x)
      (lambda (x) (tag (sin x)))
)
(define (cosine x)
      (lambda (x) (tag (cos x)))
)
```
## 2.5.3: symbolic algebra
### Polynomials
$5x^{2} + 3x + 7$ is a simple polynomial in x, and $(y^{2} + 1)x^{3} + (2y)x + 1$ is a polynomial in x whose coefficients are polynomials in y.

Using lagrange Interpolation Polynomial we can recover the coefficients of a polynomial of degree n given the values of the polynomial at n + 1 points.

We can represent polynomials using a data structure called a poly, which consists of a variable and a collection of terms. We assume that we have selectors variable and term-list that extract those parts from a poly and a constructor make-poly that assembles a poly from a given variable and a term list.
```scheme
(define (add-poly p1 p2)
      (if (same-variable? (variable p1) (variable p2))
            (make-poly (variable p1) (add-terms (term-list p1) (term-list p2)))
            (error "polys not in same var - ADD-POLY" (list p1 p2))
      )
)
(define (mul-poly p1 p2)
      (if (same-variable? (variable p1) (variable p2))
            (make-poly (variable p1) (mul-terms (term-list p1) (term-list p2)))
            (error "Polys not in same var - MUL-POLY" (list p1 p2))
      )
)
(define (install-polynomial-package)
      ;; internal procedures.
      ;; representation of poly
      (define (make-poly variable term-list) (cons variable term-list))
      (define (variable p) (car p))
      (define (term-list p) (cdr p))
      ;; representation of terms 
      
      ;(define (add-poly p1 p2) ...)
      ;(define (mul-poly p1 p2) ...)
      
      ;; interface to rest of the system.
      (define (tag p) (attach-tag `polynomial p))
      (put `add `(polynomial polynomial) (lambda (p1 p2) (tag (add-poly p1 p2))))
      (put `mul `(polynomial polynomial) (lambda (p1 p2) (tag (mul-poly p1 p2))))
      (put `make `polynomial (lambda (var terms) (tag (make-poly var terms))))
`done
)
(define (add-terms L1 L2)
      (cond ((empty-termlist? L1) L2)
            ((empty-termlist? L2) L1)
            (else
                  (let ((t1 (first-term L1)) (t2 (first-term L2)))
                        (cond ((> (order t1) (order t2)) (adjoin-term t1 (add-terms (rest-terms L1) L2)))
                              ((< (order t1) (order t2)) (adjoin-term t2 (add-terms (rest-terms L2) L1)))
                              (else (adjoin-term (make-term (order t1) (add (coeff t1) (coeff t2))) (add-terms (rest-terms L1) (rest-terms L2))))
                        )
                  )
            )
      )
)
(define (mul-terms L1 L2)
      (if (empty-termlist? L1)
            (the-empty-termlist)
            (add-terms (mul-term-by-all-terms (first-term L1) L2) (mul-terms (rest-terms L1) L2))
      )
)
(define (mul-term-by-all-terms t1 L)
      (if (empty-termlist? L)
            (the-empty-termlist)
            (let ((t2 (first-term L)))
                  (adjoin-term (make-term (+ (order t1) (order t2)) (mul (coeff t1) (coeff t2))) (mul-term-by-all-terms t1 (rest-terms L)))
            )
      )
)
```
### Representing term lists:
A polynomial is said to be dense if it has nonzero coefficients in terms of most orders. If it has many zero terms it is said to be sparse.
```scheme
(define (adjoin-term term term-list)
      (if (=zero? (coeff term))
            term-list
            (cons term term-list)
      )
)
(define (the-empty-termlist) `())
(define (first-term term-list) (car term-list))
(define (rest-term term-list) (cdr term-list))
(define (empty-termlist? term-list) (null? term-list))

(define (make-term order coeff) (list order coeff))
(define (order term) (car term))
(define (coeff term) (cadr term))
(define (make-polynomial var terms)
      ((get `make `polynomial) var terms)
)
```
### ex2.87
```scheme
(define (=zero? x)
      (if (pair? x)
            #f
            (= x 0)
      )
)
```

