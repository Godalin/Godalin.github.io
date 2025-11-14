+++
title = "Capf"
author = ["Linyu Yang"]
date = 2025-11-14T00:00:00+08:00
tags = ["Emacs"]
draft = false
+++

## Capf, Compeletion at Point Function {#capf-compeletion-at-point-function}

In this article, I will look into the completion in buffer mechanism
provided by Emacs, **Completion at Point Functions**, to implement a
feature inserting skeletons according to some identifier.  Apart from
that, I also find that `cdlatex.el` is fanatic and like the fast
abbreviations provided by `cdlatex.el`, but it works bad with
`corfu.el`.  Having it work with `corfu.el` together (through
`cape.el`) will be nice, so I will also implement this feature as a
`capf`.


### A Start Capf {#a-start-capf}

```emacs-lisp
;;; try to implement a capf
(defun my/test-capf ()
  "A test capf."
  (interactive)
  (let ((bounds (bounds-of-thing-at-point 'symbol)))
    (when bounds
      (let ((beg (car bounds))
            (end (cdr bounds)))
        (list beg
              end
              (completion-table-dynamic
               (lambda (_)
                 '("foo"
                   "bar"
                   "baz")))
              :exclusive 'no
              :annotation-function (lambda (_) " test")
              :exit-function
              (lambda (string status)
                (delete-backward-char (length string))
                (my/org-hugo-rocq-block)))))))

(add-to-list 'completion-at-point-functions #'my/test-capf)
```

Without this `:exit-function` property, the completions are output
directly.  With this simple hack, we can create more flexible features.

Here is an example output, really fancy (the `my/org-hugo-rocq-block`
function is for inserting `Rocq` files direct in hugo blog with its
`rocq doc` functionality).  `Capf` is compatible with the interactive
`skeleton` toolchain.

```text
#+begin_src shell :results output html :exports results
opam switch rocq-env >> trash && eval $(opam env)
rocq c <file here>.v
rocq doc --html --no-lib-name --body-only ./<file here>.v
sed 's|<file here>\.html|/posts/<file here>/|g' ./<file here>.html
cat ./<file here>.html
#+end_src
```


### A `cdlatex` Capf {#a-cdlatex-capf}

The goal of this article is to combine cdlatex with capf(corfu) in a
better way.  So let's hack `cdlatex`.  Knowing what we can do, this is
really easy:

```emacs-lisp
;;; implement a `cdlatex' capf
(require 'cdlatex)
(require 'cl-lib)

(setq cdlatex-command-alist-comb-keys
      (cl-map 'list #'car cdlatex-command-alist-comb))

(defun cdlatex-capf ()
  "Native CAPF version of company-cdlatex-backend."
  (interactive)
  (let ((bounds (bounds-of-thing-at-point 'symbol)))
    (when bounds
      (let ((beg (car bounds))
            (end (cdr bounds)))
        (list beg
              end
              cdlatex-command-alist-comb-keys
              :exclusive 'no
              :annotation-function (lambda (_) " cd→")
              :exit-function
              (lambda (string status)
                (cdlatex-tab)))))))

(add-to-list 'completion-at-point-functions #'cdlatex-capf)
```

We can use the following prefix for completion:

```emacs-lisp
cdlatex-command-alist-comb-keys
```

And we do this test:

```text
fg<tab> gives:

\begin{figure}[htbp]
\centerline{\includegraphics[]{}}
\caption[]{AUTOLABEL ?}
\end{figure}
```

This is fairly easy.


### References {#references}

-   <https://www.gnu.org/software/emacs/manual/html_node/elisp/Completion-in-Buffers.html>
-   <https://www.gnu.org/software/emacs/manual/html_node/elisp/Basic-Completion.html>
-   <https://www.gnu.org/software/emacs/manual/html_node/elisp/Completion-Variables.html>
-   <https://github.com/cdominik/cdlatex>
-   <https://orgmode.org/manual/CDLaTeX-mode.html>
