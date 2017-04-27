{:title "Basic setup of CIDER for hacking"
 :layout :post
 :tags ["CIDER" "setup" "emacs"]
 :toc true}

## Installation ##

Emacs normally downloads packages into a directory in `~/.emacs.d/` or somewhere else not super accessible. The first step to setting up a way to start hacking on CIDER is to have your own copy that you own. [Fork](https://github.com/clojure-emacs/cider) it, and then add the following to your your init.el or equivalent, but substituting wherever you cloned it for `~/projects/cider`:

```emacs-lisp
;; load local version of cider
(add-to-list 'load-path "~/projects/cider")
(require 'cider)
```

Delete any residual CIDER code left over in your melpa directory so that you know there's only one copy of CIDER running and its a permanent one. (This, in [prelude](https://github.com/bbatsov/prelude) is located at `~/.emacs.d/elpa`).

If you don't have CIDER's dependencies from running it previously from melpa, ensure that the following packages are installed:

- clojure-mode
- pkg-info
- queue
- spinner
- seq

See the [installation docs](https://cider.readthedocs.io/en/latest/installation/#manual-installation) for more info.

## Begin Introspecting  ##

### Looking at traffic ###

CIDER at a highlevel is an elisp client that sends messages back and forth with the cider-nrepl clojure server. The best place to start--whether gathering information for bug reports, development or prototyping--is by toggling the recording of these messages. Toggle this by `M-x nrepl-toggle-message-loggin`. After any interaction, be it autocompelete, code evaluation, etc, there will be a messages buffer with the back and forth of your code.

```clojure
(-->
  id         "55"
  op         "eldoc"
  session    "d36ae5a8-16e5-4454-985f-aa028767f33f"
  time-stamp "2017-04-26 22:52:29.466403690"
  ns         "fizzbuzz.core"
  symbol     "div-yo"
)
(<--
  id         "55"
  session    "d36ae5a8-16e5-4454-985f-aa028767f33f"
  time-stamp "2017-04-26 22:52:29.471443612"
  docstring  nil
  eldoc      (("x" "y"))
  name       "div-yo"
  ns         "fizzbuzz.core"
  status     ("done")
  type       "function"
)
(-->
  id         "56"
  op         "eval"
  session    "d36ae5a8-16e5-4454-985f-aa028767f33f"
  time-stamp "2017-04-26 22:52:31.746495637"
  code       "(div-yo 4 3)
"
  column     16
  file       "*cider-repl fizzbuzz*"
  line       68
  ns         "fizzbuzz.core"
)
(<--
  id         "56"
  session    "d36ae5a8-16e5-4454-985f-aa028767f33f"
  time-stamp "2017-04-26 22:52:31.755104708"
  ns         "fizzbuzz.core"
  value      "4/3"
)
(<--
  id         "56"
  session    "d36ae5a8-16e5-4454-985f-aa028767f33f"
  time-stamp "2017-04-26 22:52:31.766114712"
  status     ("done")
)
(<--
  id                 "56"
  session            "d36ae5a8-16e5-4454-985f-aa028767f33f"
  time-stamp         "2017-04-26 22:52:31.767151265"
  changed-namespaces (dict)
  repl-type          "clj"
  status             ("state")
)

```

These are actual messages from a test project, fizzbuzz, that I keep up for prototyping and testing. Broadly, each message is an id, session, timestamp, and then whatever information happens to be going across. Here, the first of these messages, message 55, is a request for eldoc information about the function `div-yo`. Message 56 is requesting evaluation of a simple invocation of the function and its return value, followed by a done message and a state message.

Each of these values could probably be a good topic of a post. For example, the session dictates which repl you're talking to, sorta. Each repl (really client buffer) has a process attached with actually two sessions. One session is for standard evaluation like above `(div-yo 3 4)` and the other is the tooling session, meant for things like eldoc and compeletion requests. The fact that these two are in the same session is a bug actually. The separation comes so that when you ask for the last result, `*1`, you'll never get a completition result or eldoc signature but the `4/3` that the function returned.

### Debugging Elisp Code ###

Emacs is quite a beast; self-documenting, several vm's, it is a lisp interpreter at heart. Getting familiar with the debugging tooling and navigation will become quickly second nature due to shared keybindings with Clojure navigation and debugging shortcuts.

Navigating to functions is trivial in emacs with `M-x find-function`, bound to `C-h f` in vanilla emacsen. If you're running emacs 25 you have a really nice new feature with the `Xref` [package](https://www.gnu.org/software/emacs/manual/html_node/emacs/Xref.html) for navigation. Its a general purpose go to definition bound by default to `M-.` and popping with `M-,`, just like CIDER navigation.

Once you've found the code you'd like to look at, instrumenting for debugging is trivial as well. Again, it is the same keybinding as CIDER, `C-u C-M-x`. This looks like a mouthful but my hands do this chord out of instinct: it's just the [prefix argument](https://www.gnu.org/software/emacs/manual/html_node/elisp/Prefix-Command-Arguments.html) along with the `eval-defun` command. Just like in CIDER, to de-instrument the function simply re-eval it with `C-M-x` or `eval-defun`.

Instrumenting code gives you a debugger that's quite extensive. But just remember `n` for next and `c` for continue and you've got perhaps 50% of what you need to get stepping and introspecting and hit `?` for some help.

## Running Tests ##

The testing situation in emacs is not, uhh, the best. But there are a few frameworks in use in emacs and CIDER:

- [buttercup](https://github.com/jorgenschaefer/emacs-buttercup)
- [cask](https://github.com/cask/cask)

You'll need to install cask yourself to run the tests reliably. Cask will run emacs headless, make sure that all the dependencies are met, byte compile files and then run your tests for you. The others are assertion libraries for actual tests. Once you have this met, testing is simple as:

```shell
cider|master⚡ ⇒ cask install
cider|master⚡ ⇒ make test

...
...

nrepl--merge
  preserves id and session keys of dict1
  appends all other keys
  dict1 is updated destructively

Ran 248 specs, 0 failed, in 2.3 seconds.
cider|master⚡ ⇒

```

## Contributing ##

The above should be enough to get started hacking in CIDER. The issues list always welcomes discussions and questions from new contributors. There's even a bug label [low-hanging fruit](https://github.com/clojure-emacs/cider/labels/low%20hanging%20fruit) to label things as approachable to those who don't know the code-base very well. Feel free to take issues, fix what bugs you, or contribute to discussions about open issues. The chatroom #cider on [slack](https://clojurians.slack.com/) is a place for help and discussion of feature development.

For pull requests, make sure to follow the following [guidelines](https://github.com/clojure-emacs/cider/blob/master/.github/CONTRIBUTING.md)
