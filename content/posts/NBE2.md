+++
title = "NBE, Some Notes (2)"
author = ["Linyu Yang"]
date = 2025-11-08T00:00:00+08:00
tags = ["pl", "tt", "λ"]
draft = false
+++

## Hello, Racket Contract {#hello-racket-contract}

In the previous article ([{{< relref "NBE" >}}]({{< relref "NBE" >}})) I said that in new following
articles, I will use Haskell to continue the programming.  However, I
find that it may be difficult to set the correct package configuration
in Org Mode, so I will continue using Racket.  To make structures
easier to use, I start using `racket/contract` doing dynamic checks on
the parameters of constructors.

The topic of this article: I will follow
<https://davidchristiansen.dk/tutorials/nbe/> implementing a bidirectional
type-checking algorithm for the simply typed λ-calculus.  Before that,
let's do some "safe programming" with `racket/constract`.

```racket
#lang racket
(require racket/contract)

(struct/contract point ([x number?] [y number?])
                 #:transparent)

(point 1 2)
;; (point 'x 'y) ;; error
(define p (point 1 2))
(point? p)
```

```text
(point 1 2)
#t
```

It's really easy to use!  We can construct more complicated data types
with the guarantee of correctness of argument types.  Aside from using
typed racket, the contract module can be viewed as a good compromise
if we want to keep the flexibility feature of racket.


## Bidirectional Type Checking, Simply Typed λ-Calculus {#bidirectional-type-checking-simply-typed-λ-calculus}

In this small calculus, we add a natural number type and a Boolean
type as the base type.  Except for that type, we also have function
types and product types.

```racket
#lang racket
(require racket/contract)

```


### Simple Types {#simple-types}

First we write a function to check the equality of two type
expressions.

```racket
(define (type=? t1 t2)
  (match* (t1 t2)
    [('Nat 'Nat) #t]
    [('Bool 'Bool) #t]
    [(`(,s1 → ,t1) `(,s2 → ,t2))
     (and (type=? s1 s2) (type=? t1 t2))]
    [(_ _) #f]))

```

```racket
<<stlc>>
(type=? 'Nat 'Nat)
(type=? 'Nat 'Bool)
(type=? 'Bool 'Bool)
(type=? '(Nat → Nat) '(Nat → Nat))
(type=? '(Nat → (Nat → Nat)) '(Nat → (Nat → Nat)))
(type=? '((Nat → Nat) → Nat) '(Nat → (Nat → Nat)))
```

```text
#t
#f
#t
#t
#t
#f
```

Here is trick to implement a `type?` checking function.  To check
whether an expression is a type, we simply have to check whether an
expression is `type​=​?` to itself:

```racket
(define (type? t)
  (type=? t t))

```

```racket
<<stlc>>
(type? 'Nat)
(type? 'Bool)
(type? '(Nat → Bool))
(type? '(→ Nat Bool))
```

```text
#t
#t
#t
#f
```


### Type Checking {#type-checking}

-   Synthesize \\( \Rightarrow \\) Check:
    -   to check \\( t \Leftarrow A \\)
    -   first synthesize \\( t \Rightarrow B \\)
    -   then equality check \\( A =?\ B \\)
        -   some non-trivial type check procedure can happen here, for
            example, sub-type checking
-   Check \\( \Rightarrow \\) Synthesize:
    -   to synthesize \\( t \Rightarrow ? \\) but there's no rule
    -   first add type annotation \\( t : A \\)
    -   then check whether \\( t \\) has type \\( A \\), i.e., \\( t \Leftarrow
            A \\)
    -   if type checks, we can return \\( t \Rightarrow A \\)

In logical words, i.e.:

\\[\frac{
\Gamma \vdash t \Rightarrow B \qquad
\Gamma \vdash A = B
}{
\Gamma \vdash t \Leftarrow A
}
\qquad\hbox{and}\qquad
\frac{
\Gamma \vdash t \Leftarrow A
}{
\Gamma \vdash (t \in A) \Rightarrow A
}.
\\]

For **introduction** forms, we do type check; for **elimination** forms,
we do type synthesis.

There is no reason to include a redex inside the program, where
redexes needs more type annotations in the introduction forms.

```racket
(define (bind res k)
  (if (eq? res 'ok)
      k
      res))

(define (synth Γ e)
  (match e
    ;; annotation
    [`(,e : ,t)
     (if (type? t)
         (bind (check Γ e t)
               t)
         `(:Expr ,e :NotType ,t))]

    ;; vars
    [x
     #:when (and (symbol? x)
                 (not (memv x '(rec ite tru fls λ))))
     (let ((xt (assv x Γ)))
       (if xt
           (cdr xt)
           `(:Var ,x :VarNotInScope)))]

    ;; rec
    [`(rec ,t ,en ,ez ,es)
     (if (type? t)
         (let ((tn (synth Γ en)))
           (if (eq? tn 'Nat)
               (bind (check Γ ez t)
                     (bind (check Γ es `(Nat → (,t → ,t)))
                           t))
               `(:Expr ,en :WantType Nat :HaveType ,tn)))
         `(:Type t :NotType))]

    ;; ite
    [`(ite ,t ,eb ,et ,ef)
     (if (type? t)
         (let ((tb (synth Γ eb)))
           (if (eq? tb 'Bool)
               (bind (check Γ et t)
                     (bind (check Γ ef t)
                           t))
               `(:Expr ,eb :WantType Bool :HaveType ,tb)))
         `(:Type t :NotType))]

    ;; application
    [`(,e1 ,e2)
     (let ((t (synth Γ e1)))
       (match t
         [`(,t1 → ,t2)
          (let ((res (check Γ e2 t1)))
            (if (eq? res 'ok)
                t2
                res))]
         [_ `(:Expr ,e1 :WantType (? → ?))]))]

    [_ `(:Expr ,e :TryAnnotate)]))

(define (check Γ e t)
  (match e
    ;; Nat
    ['zro
     (if (type=? t 'Nat)
         'ok
         `(:Expr ,e :WantType Nat :HaveType ,t))]
    [`(suc ,e1)
     (if (type=? t 'Nat)
         (check Γ e1 'Nat)
         `(:Expr ,e :WantType Nat :HaveType ,t))]

    ;; Bool
    ['tru
     (if (type=? t 'Bool)
         'ok
         `(:Expr ,e :WantType Bool :HaveType ,t))]
    ['fls
     (if (type=? t 'Bool)
         'ok
         `(:Expr ,e :WantType Bool :HaveType ,t))]

    ;; lambda
    [`(λ ,x ,b)
     (match t
       [`(,t1 → ,t2)
        (if (eq? x '_)
            (check Γ b t2)
            (check (cons (cons x t1) Γ) b t2))]
       [_ `(:Expr ,e :WantType (? → ?))])]

    ;; to synth
    [other
     (let ((t1 (synth Γ other)))
       (if (type=? t t1)
           'ok
           `(:Expr ,e :WantType ,t :HaveType ,t1)))]))

```

```racket
<<stlc>>
':BaseTypes
(check '() 'tru 'Bool)
(check '() '(suc (suc zro)) 'Bool)
(check '() '(suc tru) 'Bool)

(synth '() 'tru)
(synth '() '(fls : Bool))
(synth '((b . Bool)) '(ite Nat b zro (suc zro)))
(synth '((b . Bool)) '(ite Bool b tru fls))
(synth '((n . Nat)) '(rec Bool n tru (λ _ (λ _ fls))))

':FuncTypes
(check '() '(λ _ _) '(Bool → Bool))
(check '() '(λ _ tru) '(Bool → Bool))
(check '() '(λ _ tru) '(Nat → Bool))
(check '() '(λ x x) '(Bool → Bool))
(check '() '(λ x x) '(Nat → Bool))
(check '() '(λ x (λ y x)) '(Bool → (Bool → Bool)))
(check '() '(λ x (λ y x)) '(Bool → (Nat → Bool)))
(synth '((f . (Nat → Nat)) (x . Nat)) '(f x))
(synth '((f . (Nat → Nat)) (x . Bool)) '(f x))
```

```text
':BaseTypes
'ok
'(:Expr (suc (suc zro)) :WantType Nat :HaveType Bool)
'(:Expr (suc tru) :WantType Nat :HaveType Bool)
'(:Expr tru :TryAnnotate)
'Bool
'Nat
'Bool
'Bool
':FuncTypes
'(:Expr _ :WantType Bool :HaveType (:Var _ :VarNotInScope))
'ok
'ok
'ok
'(:Expr x :WantType Bool :HaveType Nat)
'ok
'ok
'Nat
'(:Expr x :WantType Nat :HaveType Bool)
```

```racket
(define (check-program Γ exprs)
  (match exprs
    ['() (void)]
    [(cons `(define ,n : ,t := ,e) rest)
     (bind (check Γ e t)
           (check-program (cons (cons n t) Γ) rest))]

    [(cons e rest)
     (let ((t (synth Γ e)))
       (displayln (format "~v" `(:Expr ,e :HasType ,t)))
       (check-program Γ rest))]))

```

```racket
<<stlc>>
(check-program '()
               '((define one : Nat := (suc zro))
                 (define two : Nat := (suc (suc zro)))
                 (define isz : (Nat → Bool)
                               := (λ n (rec Bool n
                                            tru
                                            (λ _ (λ _ fls)))))
                 (define add : (Nat → (Nat → Nat))
                               := (λ n (λ m (rec Nat n
                                                 m
                                                 (λ _ (λ s (suc s)))))))
                 (define mul : (Nat → (Nat → Nat))
                               := (λ n (λ m (rec Nat n
                                                 zro
                                                 ;; (suc n1) × m = m + n1 × m
                                                 (λ _ (λ p ((add m) p)))))))

                 (isz one)
                 (isz two)
                 ((add one) two)
                 ((mul two) two)
                 ))
```

```text
'(:Expr (isz one) :HasType Bool)
'(:Expr (isz two) :HasType Bool)
'(:Expr ((add one) two) :HasType Nat)
'(:Expr ((mul two) two) :HasType Nat)
```


### Normalization by Evaluation {#normalization-by-evaluation}

Then we go follow what we did in utlc (NBE1 [{{< relref "NBE" >}}]({{< relref "NBE" >}})) to establish a nbe
algorithm for stlc.


#### <span class="org-todo todo TODO">TODO</span> Values, Normal Forms &amp; Neutral Forms {#values-normal-forms-and-neutral-forms}

Values start from constructors, until a stuck form;

```racket
(struct CLOS (env var body)
  #:transparent)

;; values
(struct vzro ()
  #:transparent)

(struct vsuc (pred)
  #:transparent)

(struct vtru ()
  #:transparent)

(struct vfls ()
  #:transparent)

(struct vneu (type ne)
  #:transparent)

(define (value? v)
  (or (vzro? v)
      (vsuc? v)
      (vtru? v)
      (vfls? v)
      (vneu? v)
      (CLOS? v)))

;; neutral forms (stuck forms)
(struct nvar (x)
  #:transparent)

(struct napp (rator rand)
  #:transparent)

(struct nrec (type target s v)
  #:transparent)

(struct nite (type target t f)
  #:transparent)

(define (neutral? n)
  (or (nvar? n)
      (napp? n)
      (nrec? n)
      (nite? n)))

(struct vthe (type value)
  #:transparent)

(define (norm? v)
  (vthe? v))

;; evaluation
(define (ev ρ e)
  (match e
    [`(,e : ,t) (ev ρ e)]
    [`zro (vzro)]
    [`(suc ,e) (vsuc (ev ρ e))]
    [`fls (vfls)]
    [`tru (vtru)]
    [x
     #:when (and (symbol? x)
                 (not (memv x '(rec ite tru fls λ))))
     (let ((xv (assv x ρ)))
       (if xv
           (cdr xv)
           `(:Var ,x :NotFound)))]
    [`(λ ,x ,body) (CLOS ρ x body)]
    [`(rec ,t ,n ,z ,s)
     (do-rec t (ev ρ n) (ev ρ z) (ev ρ s))]
    [`(ite ,t ,b ,et ,ef)
     (do-ite t (ev ρ b) (ev ρ et) (ev ρ ef))]
    [`(,rator ,rand)
     (do-app (ev ρ rator) (ev ρ rand))]
    [_ `(:Expr ,e :BadSyntax)]))


;; apply(s)
(define (do-app rator rand)
  (match rator
    [(CLOS ρ x b)
     (ev (cons (cons x rand) ρ) b)]
    [(vneu `(,A → ,B) ne)
     (vneu B (napp ne (vthe A rand)))]))

(define (do-rec type n z s)
  (match n
    [(vzro) z]
    [(vsuc n)
     (do-app (do-app s n)
             (do-rec type n z s))]
    [(vneu 'Nat ne)
     (vneu type
           (nrec type
                 ne
                 (vthe type z)
                 (vthe `(Nat → (,type → ,type)) s)))]))

(define (do-ite type b vt vf)
  (match b
    [(vtru) vt]
    [(vfls) vf]
    [(vneu 'Bool ne)
     (vneu type
           (nite type
                 ne
                 (vthe type vt)
                 (vthe type vf)))]))
```

```racket
<<stlc>>
(ev '() '(λ x x))
(ev '() 'zro)
(ev '() '(suc zro))
(ev '() '(suc fls))
(ev '() 'fls)
(ev '() '((λ x x) zro))
(ev '() '(rec Nat zro zro zro))
(ev '() '(rec Nat (suc zro) zro (λ _ (λ _ fls))))
(ev '() '(ite Nat tru zro (suc zro)))
(ev '() '(ite Nat fls zro (suc zro)))
```

```text
(CLOS '() 'x 'x)
(vzro)
(vsuc (vzro))
(vsuc (vfls))
(vfls)
(vzro)
(vzro)
(vfls)
(vzro)
(vsuc (vzro))
```


#### <span class="org-todo todo TODO">TODO</span> Read Back {#read-back}

Now, the difference is: if we want to read back from values to
expressions, we have to consider types and η-expansions.  Before that,
we recover some functions from the previous article:

```racket
(define (add* x)
  (string->symbol
   (string-append (symbol->string x) "*")))

(define (freshen used x)
  (if (memv x used)
      (freshen used (add* x)) x))

```

Then we can construct our read back function:

```racket
(define (read-back used type value)
  (match type
    ['Nat
     (match value
       [(vzro) 'zro]
       [(vsuc v) `(suc ,(read-back used 'Nat v))]
       [(vneu _ ne)
        (read-back-ne used ne)])]
    ['Bool
     (match value
       [(vtru) 'tru]
       [(vfls) 'fls]
       [(vneu _ ne)
        (read-back-ne used ne)])]
    [`(,A → ,B)
     (let ((x (freshen used 'x)))
       `(λ ,x ,(read-back (cons x used)
                          B
                          (do-app value
                                  (vneu A (nvar x))))))]))

(define (read-back-ne used ne)
  (match ne
    [(nvar x) x]
    [(napp n-fun (vthe type-arg v-arg))
     `(,(read-back-ne used n-fun)
       ,(read-back used type-arg v-arg))]
    [(nrec type target (vthe type-zro v-zro) (vthe type-suc v-suc))
     `(rec ,type
           ,(read-back-ne used target)
           ,(read-back used type-zro v-zro)
           ,(read-back used type-suc v-suc))]
    [(nite type target (vthe type-tru v-tru) (vthe type-fls v-fls))
     `(rec ,type
           ,(read-back-ne used target)
           ,(read-back used type-tru v-tru)
           ,(read-back used type-fls v-fls))]
    [_ `(:Neu ,ne :NotValudNeu)]))

```


#### <span class="org-todo todo TODO">TODO</span> Put Them Together {#put-them-together}

```racket
(struct def (type value)
  #:transparent)

(define (def->ctx Δ)
  (match Δ
    ['() '()]
    [(cons (cons name (def type _)) rest)
     (cons (cons name type)
           (def->ctx rest))]))

(define (def->env Δ)
  (match Δ
    ['() '()]
    [(cons (cons name (def _ value)) rest)
     (cons (cons name value)
           (def->env rest))]))

(define (run-program Δ prog)
  (let ((Γ (def->ctx Δ))
        (ρ (def->env Δ)))
    (match prog
      ['() '()]
      [(cons `(define ,n : ,t := ,e) rest)
       (bind (check Γ e t)
             (let ((v (ev ρ e)))
               (displayln (format "~v"
                                  `(:Defn ,n : ,t
                                          := ,(read-back (map car Γ) t v))))
               (run-program (cons (cons n (def t v)) Δ) rest)))]
      [(cons e rest)
       (let ((t (synth Γ e)))
         (if (type? t)
             (let ((v (ev ρ e)))
               (displayln (format "~v"
                                  `(:Expr ,e : ,t
                                          => ,(read-back (map car Γ) t v))))
               (run-program Δ rest))
             t))])))

```

```racket
<<stlc>>
(run-program '()
             '((define one : Nat := (suc zro))
               (define two : Nat := (suc (suc zro)))
               (define thr : Nat := (suc two))
               (define fou : Nat := (suc thr))
               (define isz : (Nat → Bool)
                             := (λ n (rec Bool n
                                          tru
                                          (λ _ (λ _ fls)))))
               (define add : (Nat → (Nat → Nat))
                             := (λ n (λ m (rec Nat n
                                               m
                                               (λ _ (λ s (suc s)))))))
               (define mul : (Nat → (Nat → Nat))
                             := (λ n (λ m (rec Nat n
                                               zro
                                               ;; (suc n1) × m = m + n1 × m
                                               (λ _ (λ p ((add m) p)))))))
               (define fct : (Nat → Nat)
                             := (λ n (rec Nat n
                                          one
                                          ;; (suc n) ! = (suc n) × n !
                                          (λ m (λ f ((mul (suc m)) f))))))

               (isz zro)
               (isz one)
               (isz two)
               ((add one) two)
               ((mul two) two)
               ((mul two) thr)
               (fct zro)
               (fct one)
               (fct two)
               (fct thr)
               ))
```

```text
'(:Defn one : Nat := (suc zro))
'(:Defn two : Nat := (suc (suc zro)))
'(:Defn thr : Nat := (suc (suc (suc zro))))
'(:Defn fou : Nat := (suc (suc (suc (suc zro)))))
'(:Defn isz : (Nat → Bool) := (λ x (rec Bool x tru (λ x* (λ x** fls)))))
'(:Defn add : (Nat → (Nat → Nat)) := (λ x (λ x* (rec Nat x x* (λ x** (λ x*** (suc x***)))))))
'(:Defn mul : (Nat → (Nat → Nat)) := (λ x (λ x* (rec Nat x zro (λ x** (λ x*** (rec Nat x* x*** (λ x**** (λ x***** (suc x*****))))))))))
'(:Defn fct : (Nat → Nat) := (λ x (rec Nat x (suc zro) (λ x* (λ x** (rec Nat x** (rec Nat x* zro (λ x*** (λ x**** (rec Nat x** x**** (λ x***** (λ x****** (suc x******))))))) (λ x*** (λ x**** (suc x****)))))))))
'(:Expr (isz zro) : Bool => tru)
'(:Expr (isz one) : Bool => fls)
'(:Expr (isz two) : Bool => fls)
'(:Expr ((add one) two) : Nat => (suc (suc (suc zro))))
'(:Expr ((mul two) two) : Nat => (suc (suc (suc (suc zro)))))
'(:Expr ((mul two) thr) : Nat => (suc (suc (suc (suc (suc (suc zro)))))))
'(:Expr (fct zro) : Nat => (suc zro))
'(:Expr (fct one) : Nat => (suc zro))
'(:Expr (fct two) : Nat => (suc (suc zro)))
'(:Expr (fct thr) : Nat => (suc (suc (suc (suc (suc (suc zro)))))))
'()
```


### <span class="org-todo todo TODO">TODO</span> Summary {#summary}
