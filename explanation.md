```
(def checkbounds (v max)
  (or (< -1 v max) (ret 'impossible-move)))
```

"If `v` isn't (strictly) between -1 and `max`, return `'impossible-move`."

Note that `ret` isn't defined by Bel; it needs to be in dynamic scope for this
function to work. Fortunately, in the two callsites, it is.

```
(def cdddr (x) (cdr (cddr x)))
```

Bel itself defines `cadr`, `cddr`, and `caddr`, but not this one.

```
(mac actions args
  (if (car args)
      `(append (list (cons ,(car args) (fn () ,(caddr args))))
               (actions ,@(cdddr args)))))
```

This macro defines a DSL which we use later. It reads things in threes and
builts a list of pairs `(n . f)` where `n` is a number and `f` is a function.
The middle element is ignored; it's just pretty labels.

```
(mac bindret body
  (letu v
    `(ccc (fn (,v)
       (bind ret ,v
         ,@body)))))
```

The most magic component, piping a `ccc` into a `bind`. The `ccc` allows us to
do "non-local control flow" (in this case, return straight out of the whole
`run` function). The `bind` allows us to pass this ability around to other
parts of the program through the dynamic scope.

```
(def run ((w h) (x y) cmds)
  (withs (direction -i
          move (fn (f)
                 (checkbounds (++ x (f (rpart direction))) w)
                 (checkbounds (++ y (f (ipart direction))) h))
          turn (fn (angle)
                 (zap * direction angle))
          as (actions
            (0 quit             (ret (list x y)))
            (1 forward          (move idfn))
            (2 backward         (move inv))
            (3 clockwise        (turn +i))
            (4 counterclockwise (turn -i))))
    (bindret
      (map [aif (get _ as) ((cdr it)) (ret 'illegal-command)] cmds)
      (list x y))))
```

This function is too big to talk about as a unit. Let's break it down into parts.

```
  (withs (direction -i
          move (fn (f)
                 (checkbounds (++ x (f (rpart direction))) w)
                 (checkbounds (++ y (f (ipart direction))) h))
          turn (fn (angle)
                 (zap * direction angle))
          as (actions
            (0 quit             (ret (list x y)))
            (1 forward          (move idfn))
            (2 backward         (move inv))
            (3 clockwise        (turn +i))
            (4 counterclockwise (turn -i))))
```

This first part of preliminary local definitions allows us to do what we want
effectively in the short action part. The `withs` macro lets us define a
sequence of lexical variables, each one able to see the previous ones.

Let's look at the definitions in turn.

```
  (withs (direction -i
```

After some deliberation, I gave up and made `direction` a complex number
instead of a 2-element list. The math just comes out much easier this way,
since 90-degree rotations correspond to multiplying by +i or -i.

```
          move (fn (f)
                 (checkbounds (++ x (f (rpart direction))) w)
                 (checkbounds (++ y (f (ipart direction))) h))
```

The local `move` function does two things:

* Updates `x` and `y` based on (a transform `f` of) `direction`.
* Checks that we're still in bound (or returns `'impossible-move`)

The `++` function is handy here. It can update a variable in place, and it also
takes an optional second argument which means "increase by how much". We end up
sending it +1, 0, or -1, all of which are fine.

The `f` transform ends up being either `idfn` (identity transform) or `inv`
(negation), depending on if we move forwards or backwards, respectively.

```
          turn (fn (angle)
                 (zap * direction angle))
```

Something was nagging at me about expressing the direction in a natural way,
until I realized it's best described by (a small subset of) complex
multiplication.

`zap` takes an operator, applies it to a place and some arguments, and then
_stores the result back_ in the place. In other words, it turns a pure function
into something mutating.

Depending on the value of `angle`, `direction` will take on these values:

* `angle +i`: `-i`, `+1`, `+i`, `-1`, `-i`, `+1`, `+i`, `-1`...
* `angle -i`: `-i`, `-1`, `+i`, `+1`, `-i`, `-1`, `+i`, `+1`...

If we think of the complex numbers as "two-dimensional numbers", then `-i`
corresponds to `(0, -1)`. So it's not much of an abuse to use complex numbers
as vectors.

```
          as (actions
            (0 quit             (ret (list x y)))
            (1 forward          (move idfn))
            (2 backward         (move inv))
            (3 clockwise        (turn +i))
            (4 counterclockwise (turn -i))))
```

This is the DSL. The result (which we store into `as`) is a list of five
functions.

In my very first version of this program, I used a `pcase` instead to switch on
the commands. But it didn't come out quite idiomatic. It was only in later
versions I realized what I really wanted to do was a lookup into a list.

That takes care of the definitions. Now back to the final part, where we
actually interpret the commands.

```
    (bindret
      (map [aif (get _ as) ((cdr it)) (ret 'illegal-command)] cmds)
      (list x y))))
```

`bindret` was our macro that establishes the dynamic variable `ret` as meaning
"abort the computation immediately with a value". I originally called it
`withret`, but since the main point is that `ret` is dynamic (so that we can
use it in `checkbounds` and `quit`), `bindret` was a better name.

`map` iterates over all the `cmds`, in order. I'm usually wary of doing side
effects with `map`, but in Bel, that's idiomatic.

`aif` is an "anaphoric `if`"; if it succeeds (in getting an element), it calls
the result `it` in the "then" part. What we do with `it` is extract the `cdr`
(the function), and call it. If we didn't find anything, we return immediately
because the command is illegal.

Finally, we return the resulting `x y`, as a list.
