{:title "Font Locking Required Namespaces"
 :layout :post
 :tags ["cider" "bug"]
 :toc true}

## Description of the bug ##

This is a walkthrough of what I've done so far to diagnose the bug report for CIDER [bug 1985](https://github.com/clojure-emacs/cider/issues/1985). When we are loading dependencies, external or just other project namespaces, we are losing the font-locking.

```clojure
(ns stuff.core
  (:require [stuff.other :as other]))

(other/function :foo) ;; is not font-locked

```

## Moving Parts ##

### The Handler ###

The main entry-point for font-locking is in `cider-repl--state-handler`. This watches the nrepl output and when it sees a state message, it inspects it, sets the repl type, watches for new namespaces to cache, and updates the font-locking. CIDER maintains a cache of vars just watching for changes.

```emacs-lisp
(defun cider-repl--state-handler (response)
  ....
                (when-let ((ns-dict (or (nrepl-dict-get changed-namespaces (cider-current-ns))
                                        (let ((ns-dict (cider-resolve--get-in (cider-current-ns))))
                                          (when (seq-find (lambda (ns) (nrepl-dict-get changed-namespaces ns))
                                                          (nrepl-dict-get ns-dict "aliases"))
                                            ns-dict)))))
                  (cider-refresh-dynamic-font-lock ns-dict)))))))))) ;; beginning of the font-locking
```

### cider-refresh-dynamic-font-lock ###

This function just grabs the necessary information and compiles the regex that is used by the font-locking mechanism. For our purposes, it has two important parts: gets the symbols to font-lock in `(cider-resolve-ns-symbols ns)` (which notable puts the separator `/`) and then calls emacs font-locking mechanism with the compiled font-lock keywords.

```emacs-lisp
(setq-local cider--dynamic-font-lock-keywords
            (cider--compile-font-lock-keywords
             symbols (cider-resolve-ns-symbols (cider-resolve-core-ns))))
(font-lock-add-keywords nil cider--dynamic-font-lock-keywords 'end)
```

### cider--compile-font-lock-keywords ###

Oddly enough, this function seemingly works correctly. I modified this function so that it does less work and is easier to step through so we can investigate what is going on.

    (defun cider--compile-font-lock-keywords (symbols-plist core-plist)
      "Return a list of font-lock rules for the symbols in SYMBOLS-PLIST and CORE-PLIST."
      (let ((cider-font-lock-dynamically ;; (if (eq cider-font-lock-dynamically t)
                                         ;;     '(function var macro core deprecated)
             ;;   cider-font-lock-dynamically)
             '(function)
                                         )
            deprecated enlightened
            macros functions vars instrumented traced)
        (cl-labels ((handle-plist
                     (plist)
                     (let ((do-function (memq 'function cider-font-lock-dynamically))
                           (do-var (memq 'var cider-font-lock-dynamically))
                           (do-macro (memq 'macro cider-font-lock-dynamically))
                           (do-deprecated (memq 'deprecated cider-font-lock-dynamically)))
                       (while plist
                         (let ((sym (pop plist))
                               (meta (pop plist)))
                           ;; (pcase (nrepl-dict-get meta "cider.nrepl.middleware.util.instrument/breakfunction")
                           ;;   (`nil nil)
                           ;;   (`"#'cider.nrepl.middleware.debug/breakpoint-if-interesting"
                           ;;    (push sym instrumented))
                           ;;   (`"#'cider.nrepl.middleware.enlighten/light-form"
                           ;;    (push sym enlightened)))
                           ;; ;; The ::traced keywords can be inlined by MrAnderson, so
                           ;; ;; we catch that case too.
                           ;; ;; FIXME: This matches values too, not just keys.
                           ;; (when (seq-find (lambda (k) (and (stringp k)
                           ;;                                  (string-match (rx "clojure.tools.trace/traced" eos) k)))
                           ;;                 meta)
                           ;;   (push sym traced))
                           ;; (when (and do-deprecated (nrepl-dict-get meta "deprecated"))
                           ;;   (push sym deprecated))
                           (cond ((and do-macro (nrepl-dict-get meta "macro"))
                                  (push sym macros))
                                 ((and do-function (nrepl-dict-get meta "arglists"))
                                  (push sym functions))
                                 (do-var (push sym vars))))))))
          (when (memq 'core cider-font-lock-dynamically)
            (let ((cider-font-lock-dynamically '(function var macro core deprecated)))
              (handle-plist core-plist)))
          (handle-plist symbols-plist))
        `(
          ,@(when macros
              `((,(concat (rx (or "(" "#'")) ; Can't take the value of macros.
                          "\\(" (regexp-opt macros 'symbols) "\\)")
                 1 (cider--unless-local-match font-lock-keyword-face))))
          ,@(when functions
              `((,(regexp-opt functions 'symbols) 0
                 (cider--unless-local-match font-lock-function-name-face))))
          ;; ,@(when vars
          ;;     `((,(regexp-opt vars 'symbols) 0
          ;;        (cider--unless-local-match font-lock-variable-name-face))))
          ;; ,@(when deprecated
          ;;     `((,(regexp-opt deprecated 'symbols) 0
          ;;        (cider--unless-local-match 'cider-deprecated-face) append)))
          ;; ,@(when enlightened
          ;;     `((,(regexp-opt enlightened 'symbols) 0
          ;;        (cider--unless-local-match 'cider-enlightened-face) append)))
          ;; ,@(when instrumented
          ;;     `((,(regexp-opt instrumented 'symbols) 0
          ;;        (cider--unless-local-match 'cider-instrumented-face) append)))
          ;; ,@(when traced
          ;;     `((,(regexp-opt traced 'symbols) 0
          ;;        (cider--unless-local-match 'cider-traced-face) append)))
          )))

You can see the way that font locking is divided up: `(function var macro core deprecated)` We only want to investigate functions so we set it to that. This also prevents the core from being font-locked as well, as this will make the result quite large. There are several accumulators setup for macros, deprecated, vars, etc. The important bit for this here is

```emacs-lisp
  ((and do-function (nrepl-dict-get meta "arglists"))
   (push sym functions))
```

So there's the secret sauce for font-locking: it looks for arglists metadata. If so, you get font-locked. Since we've commented out so much of the function, I instrument it so we can step through it and watch for any bad information. The plist we are working through looks like this:

```emacs-lisp
(foo (dict arglists ([x]) doc "I don't do a whole lot.") uses-import (dict arglists ([x])) i/bar (dict arglists ([x])))
```

If you visit the nrepl-dict files, you'll find out that a cider dictionary is a list with the first term of `dict`. Owing to grabbing this from string methosd, some quotation marks are missing but basically this is a plist of term to dictionary: `"foo"` has a dictionary of arglists and doc. Since we are looking for arglists, all three of these functions qualify. The `regex-opt` function will compile that down into this beauty:

```emacs-lisp
(("\\_<\\(foo\\|i/bar\\|uses-import\\)\\_>" 0
 (cider--unless-local-match font-lock-function-name-face)))
```
Earlier when I said that it was working out, this is my evidence. That regex includes `i/bar` in it. The best I can think of is that emacs internal stuff requires some extra escaping, but it doesn't really make sense. To make matters _more_ confusing, you can edit the separator and then it will work. For example, in `cider-resolve-ns-symbols`, you can see where the separator is introduced when you map over the ns cache aliases:

```emacs-lisp
(nrepl-dict-flat-map (lambda (sym meta)
                       (list (concat alias "/" sym) meta))
                     (cider-resolve--get-in namespace "interns"))
```

So if you turn that slash into `*` and change the separator you use in your code, your code won't compile but it _will_ font-lock. I have no idea and I'm just hoping that if anyone wants to pick up this thread this can help them along the way.
