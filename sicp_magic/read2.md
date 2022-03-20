## 1.3 formulating abstractions with higher-order procedures
use (* x x x) instead of (cubic x) will cause great disadvantage, this forces us to work always at the level of the particular operations that happen to be primitives in the language.

One of the things we should demand from a powerful programming language is the ability to build abstractions by assigning names to common patterns and then to work in terms of the abstractions directly.

Procedures that manipulate procedures are called higher-order procedures.

an example is shown below:
```scheme
;instead of showing the cases explicitly, we can abstract like below:
(define (sum term a next b)
    (if (> a b)
        0
        (+ (term a) (sum term (next a) next b))
    )
)
;you can even use this to calculate integral.
(define (integral f a b dx)
    (define (add-dx x) (+ x dx))
    (* (sum f (+ a (/ dx 2.0)) add-dx b) dx)
)
(define (cube x) (* x x x))
(integral cube 0 1 0.01)
```
### ex1.29
make notes for Simpson's Rule. It will be shown in details in appendix.

first, for normal calculate ways, the result is shown below:
```
0.24998750000000042 ---> n = 100
0.249999875000001 ---> n = 1000
```
Now we use Simpson's Rule to compute the result.
```scheme
;because the simpson is based on the even and odd, we need count to represent it better.
(define (sum term a next b count)
    (if (> a b)
        0
        (+ (term a count) (sum term (next a) next b (+ count 1)))
    )
)
(define (cube x) (* x x x))
(define (simpson-rule f a b n)
    (define (term x count)
        (if (or (= 0 count) (= n count))
            (f x)
            (if (even? count)
                (* 2 (f x))
                (* 4 (f x))
            )
        )
  )
  (define (next x) (+ x (/ (- b a) n)))
  (* (/ (- b a) (* 3 n)) (sum term a next b 0))
)

(simpson-rule cube 0 1 100)
(simpson-rule cube 0 1 1000)
```
the result is shown below:
```
drRacket has round over this simply to 1/4:
1/4
1/4
```
### ex1.30
marked here, we can use iter for sum operation.
```scheme
(define (sum term a next b)
    (define (iter a result)
        (if (= a b)
            result
            (iter (next a) (+ result (term a)))
        )
    )
    (iter a 0)
)
```
### ex1.31
```scheme
;part a:
;denote factorial:
(define (factorial start end)
    (if (= start end)
        start
        (* start (factorial (+ start 1) end))
    )
)
;denote PI4
(define (next n count)
    (if (even? count)
        n
        (+ 2 n)
    )
)
(define (product start n count result)
    (if (= n 0)
        1
        (if (= count n)
            (* start result)
            (product (next start count) n (+ 1 count) (* start result))
        )
    )
)
(define (pi4 n)   
    (/ (product 2 n 1 1) (product 3 (+ n 1) 2 1))
)

;change product into recursive version
(define (product_plus start n count)
    (if (= n 0)
        1
        (if (= count n)
            start
            (* start (product_plus (next start count) n (+ 1 count)))
        )
   )
)
(define (pi4 n)
    (/ (product_plus 2 n 1) (product_plus 3 (+ n 1) 2))
)
```
### ex1.32
```scheme
(define (accumulate combiner null-value term a next b)
    (if (= a b)
        null-value
        (combiner (term a) (combiner null-value term (next a) next b))
    )
)

(define (product term a next b)
    (accumulate * (term a) term a next b)
)
(define (sum term a next b)
    (accumulate + (term a) term a next b)
)

(define (accumulate combiner null-value term a next b result)
    (if (= a b)
        (combiner (term a) result)
        (accumulate combiner null-value term (next a) next b (combiner (term a) result))
    )
)
```
### ex1.33
I prefer to use iteration, so `null-value` is useless here, but for some reasons I still keep them here. You can simply delete null-value to lower the space complexity.
```scheme
(define (filtered-accumulate condition? combiner null-value term a next b result)
    (if (= a b)
        (if (condition? a)
            (combiner (term a) result)
            result
        )
        (if (condition? a)
            (filtered-accumulate condition? combiner null-value term (next a) next b (combiner (term a) result))
            (filtered-accumulate condition? combiner null-value term (next a) next b result)
        )
    )
)
(define (square a) (* a a))
(define (prime? a)
    (define (prime-test test a)
        (if (> (square test) a)
            #t
            (if (divides? test a)
                #f
                (prime-test (+ 1 test) a)
            )
        )
    )
    (prime-test 2 a)
)
(define (divides? a b)
    (= (remainder b a) 0)
)
(define (identity x) x)
(define (next a) (+ 1 a))
;because term only has one parameter, we have to know the n value beforehand. assume the n is 20.
(define (gcd a b)
    (if (= b 0)
        a
        (gcd b (remainder a b))
    )
)
(define (test? a)
    (if (= 1 (gcd 20 a))
        #t
        #f
    )
)
(filtered-accumulate prime? + 0 identity 2 next 10 0)
(filtered-accumulate test? * 1 identity 1 next 20 1)
```
### 1.3.2
To avoid ineffiency in naming, we use lambda as anonymous function.

```scheme
(lambda (x) (+ x 4))
(lambda (x) (/ 1.0 (* x (+ x 2))))
```
below is a good example:
```scheme
(define (f x y)
    ((lambda (a b) (+ (* x (square a)) (*y b) (* a b))) (+ 1 (* x y)) (- 1 y))
)
;we can use let to omit more things.

(define (f x y)
    (let ((a (+ 1 (* x y)))) (b (-1 y))); a, b is denoted as parameters.
    (+ (* x (square a)) (* y b) (* a b))
)
;the general form is shown below
(let (<var1> <exp1>)
     (<var2> <exp2>)
     ...............
     (<varn> <expn>)
<body>)
```

### ex.1.34
lack parameters for the procedure, exit with error.

### 1.3.3
Try to get rid of particular functions involved, use general methods for finding zeros and fixed points of functions.

```scheme
(define (search f neg-point pos-point)
    (let ((midpoint (average neg-point pos-point)))
        (if (close-enough? neg-point pos-point)
            midpoint
            (let ((test-value (f midpoint)))
                (cond ((positive? test-value) (search f neg-point midpoint))
                      ((negative? test-value) (search f midpoint pos-point))
                      (else midpoint)
                )
            )
        )
    )
)
(define (close-enough? x y)
    (< (abs (- x y)) 0.001)
)
(define (half-interval-method f a b)
    (let ((a-value (f a)) 
          (b-value (f b)))
          
        (cond ((and (negative? a-value) (positive? b-value)) (search f a b))
              ((and (negarive? b-value) (positive? a-value)) (search f a b))
              (else (error "values are not of opposite sign" a b))
        )
    )
)
```
a number x is called a fixed point of a function f if x satisfies the equations: $f(x) = x$.
```scheme
(define tolerance 0.00001)

(define (fixed-point f first-guess)
    (define (close-enough? v1 v2)
        (< (abs (- v1 v2)) tolerance)
    )
    (define (try guess)
        (let ((next (f guess)))
            (if (close-enough? guess next)
                next
                (try next)
            )
        )
    )
    (try first-guess)
)
```
but not all the function can use this function, the oscillations might be a little big, then the changing rate is too high, for example: $f(x) = y / x$.
```scheme
;instead, we use 1/2 * (y + x / y). This is a better way to find fixed point.
(define (sqrt x)
    (fixed-point (lambda (y) (average y (/ x y))) 1.0)
)
```
### ex1.35:
```scheme
(fixed-point (lambda (y) (+ 1 (/ 1 y))) 1.0)
```
### ex1.36
```scheme
(define (fixed-point f first-guess)
    (define (close-enough? v1 v2)
        (< (abs (- v1 v2)) tolerance)
    )
    (define (try guess)
        (newline)
        (let ((next (f guess)))
            (if (close-enough? guess next)
                (display next)
                (try next)
            )
        )
    )
    (try first-guess)
)
(define (average a b) (/ 2 (+ a b)))
(fixed-point (lambda (y) (/ (log 1000) (log y))) 10.0)
(fixed-point (lambda (y) (average (/ (log 1000) (log y)) y)) 10.0)
```
result
```
(33 newline)
4.555532257016376
(8 newline)
4.555536206185039
```
### ex1.37
```scheme
(define (cont-frac Ni Di k)
    (define (cont-frac-end begin)
        (if (= begin k)
            (/ (Ni k) (Di k))
            (/ (Ni begin) (+ (cont-frac-end (+ begin 1)) (Di begin)))
        )
    )
    (cont-frac-end 1)
)
(define (close-enough? a b)
  (< (abs (- a b)) 0.0001)
  )
(define (test k)
    (if (close-enough? 0.6180339887498949 (cont-frac (lambda (i) 1.0) (lambda (i) 1.0) k))
        (display k)
        (test (+ k 1))
    )
)
(test 4)
```
result
```
9
```
We can do this iteratively.
```scheme
;remains to be seen
(define (cont-frac-1 Ni Di k)
    (define (cont-frac-end-1 begin result)
        (if (= begin k)
            result
            (cont-frac-end-1 (+ begin 1) (/ (Ni (- k begin)) (+ result (Di (- k begin)))))
        )
    )
    (cont-frac-end-1 1 (/ (Ni k) (Di k)))
)
;the generate methods in iteration and recursion is different.
```
### ex1.38
```scheme
(cont-frac (lambda (i) 1.0) 
(lambda (i)
    (if (= (remainder i 3) 2)
        (* (/ (+ 1 i) 3) 2)
        1
    )
)
10000)
```
the result is shown below:
```
0.7182818284590453
```
### ex1.39
```scheme
(define (tan-cf x k)
    (cont-frac 
        (lambda (i)
            (if (= 1 i)
                (* -1 x)
                (* (* -1 x) x)
            )
        )
        (lambda (i)
            (* -1 (- (* 2 i) 1))
        )
        k
    )
)
```
the result is shown below:
```
if we use 1.0 as the starting point, the result is:
1.557407724654902
```

### 1.3.4
we can express the idea of average damping by means of the following procedure:
```scheme
(define (average-damp f)
    (lambda (x) (average x (f x)))
)
;use the method, we can reformulate the square-root procedure as follows:
(define (sqrt x)
    (fixed-point (average-damp (lambda (y) (/ x y))) 1.0)
)
(define (cube-root x)
    (fixed-point (average-damp (lambda (y) (/ x (square y)))) 1.0)
)
```
Newton's Method: the use of the fixed-point method.
$$ f(x_{k} + t) = f(x_{k}) + f'(x_{k})t + \frac{1}{2} f''(x_k)t^2$$
To get the value of t, we use diffrentiate method for t:
$$ 0 = f'(x_k)t + \frac{1}{2} f''(x_k)t^2$$
$$t=-\frac{f'(x_k)}{f''(x_k)}$$
Since each iteration doubles the number-of-digits accuracy, this is much more rapidly than the half-interval method.
```scheme
(define (deriv g)
    (lambda (x)
        (/ (- (g (+ x dx)) (g x)) dx)
    )
)
(define dx 0.00001)
(define (cube x) (* x x x))
((deriv cube) 5)
(define (newtown-transform g)
    (lambda (x)
        (- x (/ (g x) ((deriv g) x)))
    )
)
(define (newtons-method g guess)
    (fixed-point (newton-transform g) guess)
)
(define (sqrt x)
    (newtons-method (lambda (y) (- (square y) x)) 1.0)
)
;Two ways of transform, we can pack it.
(define (fixed-point-of-transform g transform guess)
    (fixed-point (transform g) guess)
)
(define (sqrt x)
    (fixed-point-of-transform (lambda (y) (/ x y)) average-damp 1.0)
)
(define (sqrt x)
    (fixed-point-of-transform (lambda (y) (- (square y) x)) newton-transform 1.0)
)
```
### ex1.40
```scheme
(define dx 0.00001)
(define (deriv g)
    (lambda (x)
        (/ (- (g (+ x dx)) (g x)) dx)
    )
)
(define (fixed-point f first-guess)
    (define (close-enough? a b)
        (< (abs (- a b)) 0.0001)
    )
    (define (try guess)
        (let ((next (f guess)))
            (if (close-enough? guess next)
                next
                (try next)
            )
        )
    )
    (try first-guess)
)
(define (newtons-transform g)
    (lambda (x)
        (- x (/ (g x) ((deriv g) x)))
    )
)
(define (newtons-method g guess)
    (fixed-point (newtons-transform g) guess)
)
(define (cubic a b c)
    (lambda (x)
        (+ (* x x x) (* a (* x x)) (* b x) c)
    )
)
(newtons-method (cubic 2 -3 0) -9)
(newtons-method (cubic 2 -3 0) 4)
```

### ex1.41
```scheme
(define (double f)
    (lambda (x)
        (f (f x))
    )
)
```
I predict it to be 21, and it proves I am right.
### ex1.42
```scheme
(define (compose f g)
    (lambda (x)
        (f (g x))
    )
)
((compose (lambda (x) (* x x)) (lambda (x) (+ 1 x))) 6)
```
### ex1.43
```scheme
(define (repeated f n)
    (lambda (x)
        (if (= 1 n)
            (f x)
            ((repeated f (- n 1)) (f x))
        )
    )
)
((repeated (lambda (x) (* x x)) 2) 5)
```
Instead of Compose, we can use recursion or iteration to solve this problem.

### ex1.44
```scheme
(define (repeated f n)
    (lambda (x)
        (if (= 1 n)
            (f x)
            ((repeated f (- n 1)) (f x))
        )
    )
)
(define dx 0.00001)
(define (smoothing f)
    (lambda (x)
        (/ 3 (f (- x dx)) (f x) (f (+ x dx)))
    )
)
(repeated (smoothing f) n)
```

### ex1.45
```scheme
(define (repeated f n)
    (lambda (x)
        (if (= 1 n)
            (f x)
            ((repeated f (- n 1)) (f x))
        )
    )
)
(define (fixed-point f first-guess)
    (define (close-enough? a b)
        (< (abs (- a b)) 0.0001)
    )
    (define (try guess count)
        (let ((next (f guess)))
            (if (> count 1000)
                #f
                (if (close-enough? guess next)
                        next
                        (try next (+ 1 count))
                )
            )
        )
    )
    (try first-guess 0)
)
(define (average-damp f)
    (lambda (x) (average x (f x)))
)
(define (average a b)
    (/ (+ a b) 2)
)
(define (power y n)
    (if (= 1 n)
        y
        (* y (power y (- n 1)))
    )
)
(define (n-root x n guess-time)
    (fixed-point ((repeated average-damp guess-time) (lambda (y) (/ x (power y (- n 1))))) 1)
)
(n-root 243 4 2)
;;It looks like end up with O(log N)
;;How to prove it? Appendix may help.
```

### ex1.46
```scheme
(define (iterative-improve good-enough? improve)
    (define (test x)
        (if (good-enough? x)
            x
            (test (improve x))
        )
    )
    test
)
(define (sqrt x)
    ((iterative-improve (lambda (y) (< (abs (- (* y y) x)) 0.0001)) (lambda (y) (/ (+ y (/ x y)) 2))) 1)
)
(sqrt 4)
(define (fixed-point f guess)
    ((iterative-improve (lambda (x) (< (abs (- x (f x))) 0.0001)) f) guess)
)
(fixed-point (lambda (x) (/ 1 (+ 1 x))) 1)
```

## We have finished Unit 1, congradulations!!!