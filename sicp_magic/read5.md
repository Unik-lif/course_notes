## 3. Modularity, Objects, and State
We need strategies to help us structure large system so that they will be modular.

Two prominent organizational strategies:
1. objects-based
2. stream-processing

## 3.1 Assignment and Local State
We should always use an assigment operator to enable us to change the value associated with a name. It is important in Objects-based programming.
### 3.1.1
```scheme
; set won't return anything, it will simply set balance to be b(alance - amount), so we neew 'begin' keyword to return balance here.
; recall the usage of 'if' here, haha, it has been one year passed.
(define balance 100)
(define (withdraw amount)
    (if (>= balance amount)
        (begin (set! balance (- balance amount)) balance)
        "Insufficient funds"
    )
)
```
The code above will satisfy our demands, however, the `balance` above should better to be a local variable, therefore the author provides a variant below.
```scheme
(define new-withdraw
    (let ((balance 100))
        (lambda (amount)
            (if (>= balance amount)
                (begin (set! balance (- balance amount)) balance)
                "Insufficient funds"
            )
        )
    )
)
```
other ways, embed with arguments of function.
```scheme
(define (make-withdraw balance)
    (lambda (amount)
        (if (>= balance amount)
            (begin (set! balance (- balance amount)) balance)
            "Insufficient funds"
        )
    )
)
; function above can return a withdraw function
(define w1 (make-withdraw 100))
(w1 50)

(define (make-account balance)
    ; withdraw function is implemented inside
    (define (withdraw amount)
        (if (>= balance amount)
            (begin (set! balance (- balance amount)) balance)
            "Insufficient funds"
        )
    )
    ; deposit function is implementd inside
    (define (deposit amount)
        (set! balance (+ balance amount))
        balance
    )
    (define (dispatch m)
        (cond ((eq? m 'withdraw) withdraw)
              ((eq? m 'deposit) deposit)
              (else (error "Unknown request: Make-ACCOUNT" m))
        )
    )
    dispatch
)
; usage: you should first get a function bounded with balance.
; then, use this function to fetch deposit or withdraw function.
(define my-account (make-withdraw 100))
((my-account 'withdraw) 20)
```
### ex3.1
```scheme
(define (make-accumulator init)
    (lambda (num)
        (begin (set! init (+ num init)) init)
    )
)
```
### ex3.2
```scheme
(define (make-monitored f)
    (define counter 0)
    (define (dispatch m)
        (cond ((eq? m 'how-many-calls?) counter)
              ((eq? m 'reset-count) (begin (set! counter 0) counter))
              (else (begin (set! counter (+ counter 1)) (f m)))
        )
    )
    dispatch
)
```
### ex3.3
```scheme
(define (make-account balance password)
    (define (withdraw amount)
        (if (>= balance amount)
            (begin (set! balance (- balance amount)) balance)
            "Insufficient Funds"
        )
    )
    (define (deposit amount)
        (set! balance (+ balance amount))
        balance
    )
    ; use secret to get into the account.
    (define (dispatch pass func)
        (if (eq? pass password)
            (cond ((eq? func 'withdraw) withdraw)
                  ((eq? func 'deposit) deposit)
                  (else (error "this might fault" func))
            )
            (error "fail to authenticate for bad password" pass)
        )
    )
    dispatch
)
(define acc (make-account 100 'secret-password))
((acc 'secret-password 'withdraw) 40)
((acc 'some-other-password 'deposit) 50)
```
### ex3.4
```scheme
(define (make-account balance password)
    (define fail 0)
    (define (withdraw amount)
        (if (>= balance amount)
            (begin (set! balance (- balance amount)) balance)
            "Insufficient Funds"
        )
    )
    (define (deposit amount)
        (set! balance (+ balance amount))
        balance
    )
    ; use secret to get into the account.
    (define (dispatch pass func)
        (if (eq? pass password)
            (begin (set! fail 0) (cond ((eq? func 'withdraw) withdraw)
                  ((eq? func 'deposit) deposit)
                  (fault-op)
            ))  
            (if (eq? fail 7)
                call-the-cops
                (begin (set! fail (+ fail 1))
                fault-password)
            )
        )
    )
    (define (fault-op amount)
        (display "this might fault")
    )
    (define (fault-password amount)
        (display "fail to authenticate for bad password")
    )
    (define (call-the-cops amount)
        (error "You've been arrested")
    )
    dispatch
)
(define acc (make-account 100 'secret-password))
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
((acc 'some-other-password 'deposit) 50)
```
### 3.1.2
assume we have a procedure `rand-update`.
```scheme
; random-init is a fixed local value.
(define rand 
    (let ((x random-init))
        (lambda ()
            (set! x (rand-update x))
            x
        )
    )
)
```

#### Monte Carlo simulation: 
choosing sample experiments at random from a large set and then making deductions on the basis of the probabilities estimated from tabulating the results of those experiments.

```scheme
(define (estimate-pi trials)
    (sqrt (/ 6 (monte-carlo trials cesaro-test)))
)
(define (cesaro-test)
    (= (gcd (rand) (rand)) 1)
)
(define (monte-carlo trials experiment)
    (define (iter trails-remaining trails-passed)
        (cond ((= trails-remaining 0) (/ trails-passed trials))
              ((experiment) (iter (- trails-remaining 1) (+ trails-passed 1)))
              (else (iter (- trails-remaining 1) trails-passed))
        )
    )
    (iter trails 0)
)
```
Other way of monte-carlo in the SICP book Betrays some painful breaches of modularity.
### ex3.5
```scheme
; Here we assume the rectangle is formed by the diagonal lineL: (0, 0) -> (1, 1). So the squre of the rectangle is relative easy to compute.

; stackoverflow: a way to generate random.
(define random
  (let ((a 69069) (c 1) (m (expt 2 32)) (seed 19380110))
    (lambda new-seed
      (if (pair? new-seed) ; if (random x)
          (set! seed (car new-seed))
          (set! seed (modulo (+ (* seed a) c) m)))
      (/ seed m))))

; random-in-range (btw, in a simplified case we won't use this function)
(define (rand-range . args)
  (cond ((= (length args) 1)
          (* (random) (car args)))
        ((= (length args) 2)
          (+ (car args) (* (random) (- (cadr args) (car args)))))
        (else (error 'randint "usage: (randint [lo] hi)"))))

; form a point in the rectangle.
(define (point x y)
    (cons x y)
)

(define (distance point)
    (sqrt (+ (* (car point) (car point)) (* (cdr point) (cdr point))))
)

(define (insideornot point)
    (define dist (distance point))
    (if (> dist 1)
        #f ;; outside of the circle.
        #t ;; inside the circle.
    )
)

(define (circle-test)
    (define test-point (point (random) (random)))
    (insideornot test-point)
)

;; trails: number of tests.
(define (monte-carlo trials experiment)
    (define (iter trails-remaining trails-passed)
        (cond ((= trails-remaining 0) (/ trails-passed trials))
              ((experiment) (iter (- trails-remaining 1) (+ trails-passed 1)))
              (else (iter (- trails-remaining 1) trails-passed))
        )
    )
    (iter trials 0)
)

; compute the size of 1/4 circle.
(define (estimate-pi trials)
    (* 4 (monte-carlo trials circle-test))
)

;result is shown below
<!-- 
> (estimate-pi 1000)
3 + 11/50
> (estimate-pi 1000000)
3 + 8713/62500 
-->
```
### ex3.6
```scheme
(define (r)
    (let ((a 69069) (c 1) (m (expt 2 32)) (seed 19380110))
        (define (dispatch text)
            (cond ((eq? text 'generate) (begin (set! seed (modulo (+ (* seed a) c) m)) (/ seed m)))
                  ((eq? text 'reset) (lambda (newseed) (set! seed newseed)))
                  (else (error "error operations!" text))
            )
        ) 
        dispatch
    )
)
; very interesting task! mind the usage of dispatch, what r should return should be a function that has its stack value kept in the stack frame.
; Else, the change for the seed won't occur!
(define random (r))
(random 'generate)
((random 'reset) 23)
(random 'generate)
((random 'reset) 33)
(random 'generate)

#|  
    (lambda new-seed
      (if (pair? new-seed) ; if (random x)
          (set! seed (car new-seed))
          (set! seed (modulo (+ (* seed a) c) m)))
      (/ seed m)))) 
|#
```