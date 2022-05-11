### Lec3a
Use cons to combine all the things together.

### closure:
we can use cons to make even more complicated structures.

List: a convention to represent sequence.
```scheme
;function list is equal to (cons (cons (cons (cons ....))))
```
### A new language
1. primitives
2. means of combination
3. means of abstraction

Escher:

closure: greatly acceralate the process to draw big picture.

```scheme
(define (coord-map rect)
    (lambda (point)
        (+ vect
            (+ vect
                (scale (xcor point) (horiz rect))
                (scale (ycor point) (vert rect))
            )
            (origin rect)
        )
    )
)

```