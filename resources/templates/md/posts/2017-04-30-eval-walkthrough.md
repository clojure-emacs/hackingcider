{:title "Lifecycle Of a Evaluation"
 :layout :post
 :tags ["cider" "evaluation" "walkthrough"]
 :toc true}

## Lifecycle of a request ##

I wanted to give a broad overview of the lifecycle of an eval request, from invocation to marking completed. As with any code walkthrough, ultimately every guide will elide details as the only true witness of what will happen is the code itself. That being said, I'm trying to thread a fine line of not drowning in details while still giving a fairly technical overview of the eval mechanism of CIDER.

This is a good system to serve as a walkthrough as this is largely what CIDER is: emacs lisp sending messages to your project code running in clojure code. There are lots of other features but at its core, any repl interaction will be this functionality. A good place to start is the message logging that happens when you invoke  `M-x nrepl-toggle-message-logging`.

```clojure
(-->
  id         "16"
  op         "eval"
  session    "42da1513-54f2-4a29-b4b1-603d07a18434"
  time-stamp "2017-04-30 11:20:56.916993414"
  code       "(+ 1 2)
"
  column     12
  file       "*cider-repl CLJS cljc-bug*"
  line       49
  ns         "cljs.user"
)
(<--
  id         "16"
  session    "42da1513-54f2-4a29-b4b1-603d07a18434"
  time-stamp "2017-04-30 11:20:56.975191155"
  ns         "cljs.user"
  value      "3"
)
(<--
  id         "16"
  session    "42da1513-54f2-4a29-b4b1-603d07a18434"
  time-stamp "2017-04-30 11:20:56.986582000"
  status     ("done")
)
(<--
  id                 "16"
  session            "42da1513-54f2-4a29-b4b1-603d07a18434"
  time-stamp         "2017-04-30 11:20:56.987795668"
  changed-namespaces (dict)
  repl-type          "cljs"
  status             ("state")
)
```

This post tries to present a fairly technical explanation of these messages and the code that drives evaluation in CIDER.

### Outgoing ###

#### cider-interactive-eval ####

The code quickly hits [cider-interactive-eval](https://github.com/clojure-emacs/cider/blob/master/cider-interaction.el#L1098). Since we might be eval-ing code in a namespace that has not been loaded yet, [cider--prep-interactive-eval](https://github.com/clojure-emacs/cider/blob/master/cider-interaction.el#L1076)) will make sure that the namespace has been found and evaluated.

#### asynchronous setup ####

The communication channel with nrepl is asynchronous using registered callbacks to handle the results of interaction with nrepl. To setup the handler defaults for interactive evaluation,  `cider-interactive-eval-handler`. But the real important stuff happens in the [nrepl-make-response-handler](https://github.com/clojure-emacs/cider/blob/master/nrepl-client.el#L737).

Owing to the asynchronous manner of calling, this makes sure that response handlers clean up after themselves. In particular, from nrepl-make-response-handler:

```emacs-lisp
(when (member "done" status)
  (nrepl--mark-id-completed id)
  (when done-handler
    (funcall done-handler buffer)))
```

With this callback constructed, it heads into last legs of the outgoing side where the spinner is started up and the callback modified to stop it as well.

#### Nrepl encoding and operation ####

The penultimate step on the outgoing side is to set which operation is to be performed, `("op" "eval")`, and then finally sent it "across". This last bit of code is fairly straightforward in [nrepl-send-request](https://github.com/clojure-emacs/cider/blob/master/nrepl-client.el#L805):

```emacs-lisp
(defun nrepl-send-request (request callback connection &optional tooling)
  "Send REQUEST and register response handler CALLBACK using CONNECTION.
REQUEST is a pair list of the form (\"op\" \"operation\" \"par1-name\"
\"par1\" ... ). See the code of `nrepl-request:clone',
`nrepl-request:stdin', etc. This expects that the REQUEST does not have a
session already in it. This code will add it as appropriate to prevent
connection/session drift.
Return the ID of the sent message.
Optional argument TOOLING Set to t if desiring the tooling session rather than the standard session."
  (with-current-buffer connection
    (when-let ((session (if tooling nrepl-tooling-session nrepl-session)))
      (setq request (append request `("session" ,session))))
    (let* ((id (nrepl-next-request-id connection))
           (request (cons 'dict (lax-plist-put request "id" id)))
           (message (nrepl-bencode request)))
      (nrepl-log-message request 'request)
      (puthash id callback nrepl-pending-requests)
      (process-send-string nil message)
      id)))
```

Here the session id is extracted from the connection. Connections keep buffer-local variables for the two sessions, tooling and standard, which is now put into the request. Previously, this session was put into the request at earlier stages, leading to some subtle bugs. An id is generated from the buffer-local `nrepl-request-counter`, and the message is bencoded, the transport format used for communication. The message is logged (if toggled, ie, the first `(-->` form at the top of this post). The callback is registered in `nrepl-pending-requests` and the actual transmission is accomplished with `(process-send-string nil message)`.

### Incoming ###

Emacs runs the jvm as a process. And the communication is by the above `process-send-string` and by [filter functions](https://www.gnu.org/software/emacs/manual/html_node/elisp/Filter-Functions.html) that read the resulting output written to standard out.

When creating the client process in [nrepl-start-client-process](https://github.com/clojure-emacs/cider/blob/master/nrepl-client.el#L637), several things happen:

- `:response-q` is created `(process-put client-proc :response-q (nrepl-response-queue))`
- `:string-q` is created
- project-dir is set
- endpoints are set
- hash-table for pending and completed requests are set
- the filter is set on outcoming text from nrepl to handle responses.

The [nrepl-client-filter](https://github.com/clojure-emacs/cider/blob/master/nrepl-client.el#L467) watches the output and keeps storing it in a variable associated with the process called :string-q (think string queue) to gather incoming strings. This gets moved over into the response queue when the following failsafe test is true:

```emacs-lisp
;; Start decoding only if the last letter is 'e'
(when (eq ?e (aref string (1- (length string))))
```
The letter `e` is a fine marker for the end of encoded input. Once the string has been decoded and put into the response queue, the callbacks are called. `nrepl-response-handler-functions`, which is something set globally at repl creation, and the meat: `(nrepl--dispatch-response response)`.

```emacs-lisp
(while (queue-head response-q)
  (with-current-buffer (process-buffer proc)
    (let ((response (queue-dequeue response-q)))
      (with-demoted-errors "Error in one of the `nrepl-response-handler-functions': %s"
        (run-hook-with-args 'nrepl-response-handler-functions response))
      (nrepl--dispatch-response response))))
```

The dispatch response function logs the message, gets the callback and invokes it. The importance of the id is shown here, as this is the key logged into the `nrepl-pending-requests` hashmap and used to invoke the callback later after as the response is received. In our example here, this would write the to the repl, but in general this is just a big case statement:

```emacs-lisp
(cond (value
       (when value-handler
         (funcall value-handler buffer value)))
      (out
       (when stdout-handler
         (funcall stdout-handler buffer out)))
      (pprint-out
       (cond (pprint-out-handler (funcall pprint-out-handler buffer pprint-out))
             (stdout-handler (funcall stdout-handler buffer pprint-out))))
      (err
       (when stderr-handler
         (funcall stderr-handler buffer err)))
      (status
       (when (member "interrupted" status)
         (message "Evaluation interrupted."))
       (when (member "eval-error" status)
         (funcall (or eval-error-handler nrepl-err-handler)))
       (when (member "namespace-not-found" status)
         (message "Namespace not found."))
       (when (member "need-input" status)
         (cider-need-input buffer))
       (when (member "done" status)
         (nrepl--mark-id-completed id)
         (when done-handler
           (funcall done-handler buffer))))))))
```

We can again see when the id is marked complete `(nrepl--mark-id-completed id)`.

The last bit that happens in the lifecycle of a request is the hook that runs from the client-filter:

```emacs-lisp
(run-hook-with-args 'nrepl-response-handler-functions response)
```

This serves to invoke the following state handler:

```emacs-lisp
(defun cider-repl--state-handler (response)
  "Handle the server state contained in RESPONSE.
Currently, this is only used to keep `cider-repl-type' updated."
  (with-demoted-errors "Error in `cider-repl--state-handler': %s"
    (when (member "state" (nrepl-dict-get response "status"))
      (nrepl-dbind-response response (repl-type changed-namespaces)
        (when repl-type
          (setq cider-repl-type repl-type))
        (unless (nrepl-dict-empty-p changed-namespaces)
          (setq cider-repl-ns-cache (nrepl-dict-merge cider-repl-ns-cache changed-namespaces))
          (dolist (b (buffer-list))
            (with-current-buffer b
              ;; Metadata changed, so signatures may have changed too.
              (setq cider-eldoc-last-symbol nil)
              (when (or cider-mode (derived-mode-p 'cider-repl-mode))
                (when-let ((ns-dict (or (nrepl-dict-get changed-namespaces (cider-current-ns))
                                        (let ((ns-dict (cider-resolve--get-in (cider-current-ns))))
                                          (when (seq-find (lambda (ns) (nrepl-dict-get changed-namespaces ns))
                                                          (nrepl-dict-get ns-dict "aliases"))
                                            ns-dict)))))
                  (cider-refresh-dynamic-font-lock ns-dict))))))))))
```

This is some not very nice code. This watches for the following status messages:

```clojure
(<--
  id                 "16"
  session            "42da1513-54f2-4a29-b4b1-603d07a18434"
  time-stamp         "2017-04-30 11:20:56.987795668"
  changed-namespaces (dict)
  repl-type          "cljs"
  status             ("state")
)
```

In particular, note the looping over all open buffers **not just clojure buffers** and sets buffer local variables (it hopes) to nil. Further, it's not smart enough to remember its important buffers but uses `cider-mode` and `cider-repl-mode` as markers for important dictionaries of namespaces. These are used to font-lock the relevant buffers with known clojure and project function names, macros, etc.

The main point of it is to record what the repl type is, `'clj` or `'cljs` as well as font-lock the buffers.

## Navigation ##

Throughout all of this, `xref-find-defintion` and `M-x rgrep` have been invaluable. Getting used to these tools makes navigating CIDER quite easy.

## Wrap-up ##

While its possible that there are some minor mistakes, this cuts quite a swatch across the codebase. the hope is that knowing these mechanics, idioms, and variables gives aid in bug reporting, debugging, and general confidence for newcomers to jump into the codebase. Take a few minutes and navigate through the whole lifecycle.

