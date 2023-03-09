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