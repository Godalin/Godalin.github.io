+++
title = "NBE, Some Notes (1)"
author = ["Linyu Yang"]
date = 2025-11-07T00:00:00+08:00
draft = false
+++

## Hello, Racket {#hello-racket}

In this series of articles, I will follow some literature (mainly by
David Thrane Christiansen,
<https://davidchristiansen.dk/tutorials/nbe/>) to implement some kind of
**Normalization by Evaluation** method.  In addition, this will also be
the first blog article I write with **Org Mode**, **Hugo**, and the **Ox-Hugo**
emacs lisp package, automatically generate Hugo articles.  Also, I
add the support for racket language blocks directly inside Org Mode,
and use it to show evaluation results inside Org files.

First of all, let me check the functionality of racket blocks:

<a id="code-snippet--hello"></a>
```racket
(define (hello)
  'hello-world)

(hello)
```

Then I have to check that later blocks can use previously defined
functions. This is done with the following two methods:


### :session {#session}

In default `:results value` mode, only the value of the last
expression displayed.  This is not satisfiable, however, `:results
output` is worse: it does not give any output. So we use `value`.

`:session` only supports scheme mode, but in this mode racket
evaluates wiredly. So let us simply concatenate these code blocks.


### :noweb {#noweb}

`:noweb` is a method to concatenate different code blocks. It is
flexible but requires a lot of manual work.  In this project, I will
use `:noweb` to do call previous code blocks.

```racket
<<hello>>
(hello)
(hello)
```

Now, let's go to our topic: NBE.


## NBE, Untyped λ-Calculus {#nbe-untyped-λ-calculus}

```racket
#lang racket

```


### Closure, Environment, Variables {#closure-environment-variables}

Closure captures the current evaluation environment.

```racket
(struct CLOS (env var body)
  #:transparent)

```

How to get a value from the environment?

```racket
(assv 'x (list (cons 'y "peaches") (cons 'x "apples")))
(assv 'x '((x . y) (y . z)))
(assv 'x '((z . y) (y . z)))
```

```text
'(x . "apples")
'(x . y)
#f
```

Extend the eval environment with a new binding

```racket
(define (extend ρ x y)
  (cons (cons x y) ρ))

```

```racket
<<utlc>>
(define basic '((x . "world") (y . "hello")))
(extend basic 'z "god")
```

```text
'((z . "god") (x . "world") (y . "hello"))
```

The interesting part: the **evaluator**:

```racket
;; eval
(define (val ρ e)
  (match e
    [`(λ ,x ,b) #:when (symbol? x)
                (CLOS ρ x b)]
    [x #:when (symbol? x)
       (let ((xv (assv x ρ)))
         (if xv
             (cdr xv)
             (error 'val "unknown variable ~a" x)))]
    [`(,rator ,rand)
     (do-ap (val ρ rator) (val ρ rand))]))

;; apply
(define (do-ap clos arg)
  (match clos
    [(CLOS ρ x b)
     (val (extend ρ x arg) b)]))

```

This eval-apply loop is exactly how to implement a lisp inside lisp.

```racket
<<utlc>>
(val '() '(λ x (λ y y)))
(val '() '((λ x x) (λ x x)))
(val '() 'x)
```

```text
val: unknown variable x
  context...:
   body of "/var/folders/3q/654xm_6x2m7f6qt50nghq1cr0000gn/T/babel-RGfyU5/org-babel-Ra3h4s.rkt"
(CLOS '() 'x '(λ y y))
(CLOS '() 'x 'x)
```

Then we write a function to evaluate a list of expressions:

```racket
(define (run-program ρ exprs)
  (match exprs
    ['() (void)]
    [(cons `(define ,x ,e) rest)
     (let ((v (val ρ e)))
       (run-program (extend ρ x v) rest))]
    [(cons e rest)
     (displayln (val ρ e))
     (run-program ρ rest)]))

```

```racket
<<utlc>>
(run-program '()
             '((define idt
                 (λ x x))
               (define tru
                 (λ x (λ y x)))
               (define fls
                 (λ x (λ y y)))
               (define ite
                 (λ i (λ t (λ e ((i t) e)))))

               (((ite tru) tru) fls)
               (((ite fls) tru) fls)

               (define zro
                 (λ f (λ z z)))
               (define suc
                 (λ n (λ f (λ z (f ((n f) z))))))

               (suc zro)
               ))

```

```text
#(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))
#(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))
#(struct:CLOS ((n . #(struct:CLOS ((ite . #(struct:CLOS ((fls . #(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))) (tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) i (λ t (λ e ((i t) e))))) (fls . #(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))) (tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) f (λ z z))) (zro . #(struct:CLOS ((ite . #(struct:CLOS ((fls . #(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))) (tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) i (λ t (λ e ((i t) e))))) (fls . #(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))) (tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) f (λ z z))) (ite . #(struct:CLOS ((fls . #(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))) (tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) i (λ t (λ e ((i t) e))))) (fls . #(struct:CLOS ((tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) x (λ y y))) (tru . #(struct:CLOS ((idt . #(struct:CLOS () x x))) x (λ y x))) (idt . #(struct:CLOS () x x))) f (λ z (f ((n f) z))))
```


### Normalization {#normalization}


#### Fresh {#fresh}

I need a way to generate new names, since in this article, I use
named variables.  In the next article, I will implement a version with
de Bruijn indices, and in an statically typed language (maybe
... Haskell ...).

This function only adds a `*` to a symbol:

```racket
(define (add* x)
  (string->symbol
   (string-append (symbol->string x)
                  "*")))

```

```racket
<<utlc>>
(add* 'xyz)
```

```text
'xyz*
```

This function generates a fresh symbol based on a list of symbols:

```racket
(define (freshen used x)
  (if (memv x used)
      (freshen used (add* x))
      x))

```

```racket
<<utlc>>
(freshen '(a a* c) 'b)
(freshen '(a a* c) 'c)
(freshen '(a a* c) 'a)
```

```text
'b
'c*
'a**
```


#### Normal &amp; Neural Forms {#normal-and-neural-forms}

A normal form is just a value (see precious sections, values can only
be closures), or a **stuck** term, which we call **Neural Forms**. Neural
forms can only appear under binders, since it will contain free
variables, which must be governed by binders.

```racket
(struct N-var (sym)
  #:transparent)

(struct N-ap (rator rand)
  #:transparent)

```

```racket
<<utlc>>
(N-var 'x)
(N-var 'y)
(N-ap (N-var 'x) (CLOS '() 'x 'x))
```

```text
(N-var 'x)
(N-var 'y)
(N-ap (N-var 'x) (CLOS '() 'x 'x))
```

Our evaluator needs some modification.  Mainly about neural forms,
since the cases of input terms remain the same.

```racket
;; eval
(define (valn ρ e)
  (match e
    [`(λ ,x ,b) #:when (symbol? x)
                (CLOS ρ x b)]
    [x #:when (symbol? x)
       (let ((xv (assv x ρ)))
         (if xv
             (cdr xv)
             (error 'valn "unknown variable ~a" x)))]
    [`(,rator ,rand)
     (do-apn (valn ρ rator) (valn ρ rand))]))

;; apply
(define (do-apn fun arg)
  (match fun
    [(CLOS ρ x b)
     (valn (extend ρ x arg) b)]
    [neural-fun
     (N-ap fun arg)]))

```

It is subtle that if we only try to evaluate things in the empty
environment, there will be no different compared with the previous
version, since the `valn` is the same as `val` for all input cases,
and for the `do-ap` function, `TODO, explain why the same`

```racket
<<utlc>>
(valn '() '(λ x x))
(valn '() '(λ x (λ y (y x))))
(valn '() '(λ x (x x)))
```

```text
(CLOS '() 'x 'x)
(CLOS '() 'x '(λ y (y x)))
(CLOS '() 'x '(x x))
```


#### Read Back {#read-back}

```racket
(define (read-back used-names v)
  (match v
    [(CLOS ρ x body)
     (let* ((y (freshen used-names x))
            (ne-y (N-var y)))
       `(λ ,y ,(read-back (cons y used-names)
                          (valn (extend ρ x ne-y) body))))]
    [(N-var x) x]
    [(N-ap rator rand)
     `(,(read-back used-names rator) ,(read-back used-names rand))]))

(define (norm ρ e)
  (read-back '() (valn ρ e)))

```

```racket
<<utlc>>
(norm '() '(λ x x))
(norm '() '((λ x x) (λ y y)))
(read-back '() (valn '() '((λ x (λ y (x y))) (λ x x))))
```

```text
'(λ x x)
'(λ y y)
'(λ y y)
```

```racket
(define (run-program-n ρ exprs)
  (match exprs
    [(list) (void)]
    [(list `(define ,x ,e) rest ...)
     (let ((v (valn ρ e)))
       (run-program-n (extend ρ x v) rest))]
    [(list e rest ...)
     (displayln (norm ρ e))
     (run-program-n ρ rest)]))

```

```racket
<<utlc>>
(run-program-n '()
               '((define zro
                   (λ f (λ z z)))
                 (define suc
                   (λ n (λ f (λ z (f ((n f) z))))))

                 zro
                 suc
                 (suc zro)
                 (suc (suc zro))
                 (suc (suc(suc zro)))
                 ))
```

```text
(λ f (λ z z))
(λ n (λ f (λ z (f ((n f) z)))))
(λ f (λ z (f z)))
(λ f (λ z (f (f z))))
(λ f (λ z (f (f (f z)))))
```

And this is what we want.


## At the End {#at-the-end}

I find it intolerable to programming in a dynamically typed
language. I have to manually check whether a field has a correct type
all the time.  So in the next article of this series, I will switch to
a statically typed language to do it again.
