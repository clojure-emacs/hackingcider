{:title "Basic setup of cider-nrepl for hacking"
 :layout :post
 :tags ["cider-nrepl" "setup" "clojure"]
 :toc true}

## Installation ##

Working with cider-nrepl is a little more difficult and a little less straightforward. The best way that I've found with it is to install it in your maven directory. When I'm developing on it, I run the shell command `rm -rf ~/.m2/repository/cider/ && lein install`.

Cider looks for the version that matches it's own:

```emacs-lisp
(defvar cider-jack-in-lein-plugins nil
  "List of Leiningen plugins where elements are lists of artifact name and version.")
(put 'cider-jack-in-lein-plugins 'risky-local-variable t)
(cider-add-to-alist 'cider-jack-in-lein-plugins
                    "cider/cider-nrepl" (upcase cider-version))

```

So we just make sure that the cider-nrepl version matches the CIDER version you are working against. If you keep both up to date with the source you should almost never have to worry about this. When CIDER asks for the dependency, it will find your local copy in the maven cache and not fetch a version from clojars.

```clojure
(def VERSION "0.15.0-SNAPSHOT")

(defproject cider/cider-nrepl VERSION
  :description "nREPL middlewares for CIDER"
  ... )

```

More information is found at the [cider docs](https://cider.readthedocs.io/en/latest/hacking_on_cider/#hacking-on-cider-nrepl).

## Broad Overview ##

The middleware lets you define operations on top of what nrepl itself handles. You can define an operation, say which middleware it requires, and then put its implementation. At the bottom of each middleware file is a macro `set-descriptor!` describing which operations the middleware handles and the handler function. So for example, here's the handler function that refreshes namespaces:

```clojure
(defn wrap-refresh
  "Middleware that provides code reloading."
  [handler]
  (fn [{:keys [op] :as msg}]
    (case op
      "refresh" (refresh-reply (assoc msg :scan-fn dir/scan))
      "refresh-all" (refresh-reply (assoc msg :scan-fn dir/scan-all))
      "refresh-clear" (clear-reply msg)
      (handler msg))))
```

As always, consult some documentation at the [nrepl github page.](https://github.com/clojure/tools.nrepl#middleware)

## Working In the Codebase ##

You can jack-in and work with things. The debugger works somewhat as well. Just be prepared to restart emacs or the repl as things get wonky easily.

I often resort to print statements in the codebase and then install in the maven repository to watch the statements bubble up in CIDER. I wish I had a better solution to this and if anyone does, please do a pull request or your own article about your process.

## Testing ##

In order to run all tests, you need to invoke the tests with profiles, as you can see from the `project.clj`:

```clojure
             :test-clj {:test-paths ["test/clj"]
                        :java-source-paths ["test/java"]
                        :resource-paths ["test/resources"]}
             :test-cljs {:test-paths ["test/cljs"]
                         :dependencies [[com.cemerick/piggieback "0.2.1"]
                                        [org.clojure/clojurescript "1.7.189"]]}
```
You can see that the testpaths for the CIDER test runner are extended with profiles. So when jacked-in, you can't easily just `C-c C-t p` to run all tests in project, nor does `lein test` run them all.

The best way to run the tests would be

```shell
cider-nrepl> lein with-profile +test-clj test
cider-nrepl> lein with-profile +test-cljs test

```
I'm not sure I see a benefit of excluding the test-clj test sources from the base profile and the [docs](https://cider.readthedocs.io/en/latest/hacking_on_cider/#testing-the-code_1) even hint that perhaps its time for this to change.

