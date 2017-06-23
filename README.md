## Hacking CIDER ##

This is a site about hacking on CIDER. Anyone can contribute through pull requests with articles about working out bugs, adding features, or just documenting how some aspects of CIDER work. The point is to get more people comfortable in the codebase and willing to add changes. We are all working in Clojure so picking up lisp should come easy, and we want to provide as many guides as we can to that end here.

This site is a static site compiled with [Cryogen](http://cryogenweb.org/index.html). Blog posts are in markdown and placed in `resources/templates/md/` and must conform to the filename `yyyy-dd-mm-title-name.md`. The top of each file must contain some metadata in the form of a clojure map along the following structure:

```clojure
{:title "Basic setup of CIDER for hacking"
 :layout :post
 :tags ["CIDER" "setup" "emacs"]
 :toc true}
```

The site is located at [www.hackingcider.com](http://www.hackingcider.com).

## How to Add ##

Just clone the site and run `lein ring server`. This creates the webserver and serves the articles. Make your own article in the directory mentioned above and then send a pull request our way.

To compile the site, just run `lein run` which will create the `resources/public` directory.

If you would like to customize the look of the site, please feel free as it is very basic and could use some visual tweaks.
