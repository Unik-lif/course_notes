## special basic procedure:
```scheme
(-1 + 3); <=> (- 3 1)
(1 + 3); <=> (+ 1 3)
; a very interesting recursion
(define (+ x y)
    (if (= x 0)
        y
        (+ (-1 + x) (1 + y)))); tail recursion: iteration
;time consumption O(x), space consumption O(1) (use 1 and delete 1)

(define (+ x y)
    (if (= x 0)
        y
        (1+ (+ (-1 + x) y)))); common recursion.(+ (-1 + x) y) this part is the recursion.
;tim consumption O(x), space consumption O(x). Linear Recursion.
```
### iteration cases
![photos1](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-02-04%20101622.png)
### recursion cases
![photos2](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-02-04%20101834.png)

### fibnacci
#### time complexity: O(fib(n)) 
(use recursion tree to analyse this)

#### space complexity: O(n) 
(fib(4)的计算要深入到primitive部分，同时要知道向上进行累加时的累加情况与关系，因此记录整个树的深度，即为n)

