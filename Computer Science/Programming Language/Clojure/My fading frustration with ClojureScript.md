#clojure #fp #web

I’ve talked about [at another post](https://mauricio.szabo.link/blog/2018/04/05/my-frustration-with-clojurescript/) on how ClojureScript frustrates me, mostly because I was doing some Node.JS work and Figwheel simply wasn’t working correctly. Now, it’s time to revisit these points:

_A little update_: I talked a little with Thomas Heller, Shadow-CLJS creator, and he pointed me some issues with this article, so I’ll update it acordingly

## Tooling

Figwheel and Lein are not the best tools to work with ClojureScript. Since I discovered shadow-cljs, things are working way better than before: I can reload ClojureScript code from any target, and I’m even experimenting with Hubot (and it works really fine too). The only thing I’m missing is my `profiles.clj` file, but I can live with that (and I can always use Shadow with Lein if I need profiles.clj).

Also, I’m working on a new package for Atom (and in the future, for another editors too) called [Chlorine](http://github.com/mauricioszabo/atom-chlorine/). One of the ideas is to offer better ClojureScript support (we have Autocomplete now!), using Socket REPL for solutions (even self-hosted REPLs like Lumo and Plank) and even wrap UNREPL protocol in Clojure. So far, is still in the very beginning but things are looking promising!

## The stack

Forget Figwheel _at all_: Shadow-CLJS is probably the best tooling for ClojureScript **ever**. It auto-reloads ClojureScript code for the browser, for node.js, for node modules, and it integrates with almost everything that you want. It controls release optimizations, have sensible defaults, and even have post-compile hooks (so you can hook Clojure code to do something after some compilation phases). Also, it integrates with node-modules (no more maven-wrappers for JS libraries!) and have some warnings when you use some kind of ClojureScript code that would break `:advanced` compilation. And, let’s not forget that you can control the refresh reload phase, it adds a userful `:include-macros` in `ns` form (that will include all macros from the namespace being required), and controls exports in a sane manner. But first let’s begin with the feature that I found most useful: `:before-load-async`.  
  
When you’re developing ClojureScript code, one of the killer features is the abilty to reload your code after save, so at any moment you have the most recent version of your code running. Let’s see how it works in practice: imagine you’re developing a simple static website (no react, no vue.js, nothing – just a simple site that’ll calculate the sum of two input boxes). You already have the HTML file with two input boxes (IDs “n1” and “n2”) and a “results” div. You’ll want to add Shadow-CLJS in this project.

Simple: just `npm install shadow-cljs`, then `npx shadow-cljs init`, and edit the `shadow-cljs.edn` file. You’ll want to add a build targeting browser, so the file will look like this:

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8|`{``:source-paths` `[``"src"``]`<br><br> `:dependencies` `[``]`<br><br> `:builds` `{``:app` `{``:target` `:browser`<br><br>                `:modules` `{``:main` `{``:init-fn` `calc.core/main}}`<br><br>                `:output-dir` `"js"`<br><br>                `:asset-path` `"js"`<br><br>                `:devtools`<br><br>                `{``:before-load-async` `calc.core/reload}}}}`|

Most of the keys should be self-explanatory. The ones that are not are:

- `:asset-path` mostly says that, relative to the root of your webserver, where assets will be put – for instance, suppose that your webserver will load static assets from the `public` directory. Then, your config will be `:output-dir "public/js"`, so that the compiler will generate code in a folder that your webserver will be able to serve, and `:asset-path "js"`, so the compiler will generate “requires” relative to “js” folder, not “public/js”
- `:devtools` registers a “hook function” that’ll be called when Shadow-CLJS have compiled the source paths, and it is ready to reload our code. This “reload function” will receive a parameter (let’s call it `done`) that, when called, it informs shadow-cljs that it’s time to use the new versions of our functions. So, let’s first look at the `main` function:

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10|`(``ns` `calc.core)`<br><br>`(``defn``- calculate-res` `[``]`<br><br>  `(``let` `[``n1 (-> (js/jQuery` `"#n1"``) .val (js/parseInt))`<br><br>        `n2 (-> (js/jQuery` `"#n2"``) .val (js/parseInt))``]`<br><br>    `(prn` `[``n1 n2``]``)`<br><br>    `(. (js/jQuery` `"#result"``) (text (+ n1 n2)))))`<br><br>`(``defn` `main` `[``]`<br><br>  `(. (js/jQuery` `"input"``) (on` `"input"` `""` `calculate-res)))`|

You can start our compiler with `npx shadow-cljs watch app`, and yes, we’re using jQuery. Also, there’s a `prn` function that’ll be used to debug. If you include `js/main.js` as a script in your webpage, this code will work – but it’ll not reload. So, after these functions, we can add our reload function that’ll just call our `main` function after informing shadow that we did our cleanup:

|   |   |
|---|---|
|1<br><br>2<br><br>3|`(``defn` `reload` `[``done``]`<br><br>  `(done)`<br><br>  `(main))`|

If you change your code in any way – maybe change the math operation to multiplication – you’ll see that our page changes correctly. You’ll also see, in the console, that we’re printing our numbers twice per change. This is because jQuery’s `.on` method stacks callbacks: this means that we’re still listening to our old code. What we have to do is to clean our callbacks _before_ reloading our code, and this is quite simple with Shadow:

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4|`(``defn` `reload` `[``done``]`<br><br>  `(.off (js/jQuery` `"*"``)` `"input"` `""``)`<br><br>  `(done)`<br><br>  `(main))`|

UPDATE: As Thomas Heller told me, `:before-load-async` is used do signal my application that a reload is due, so I can control what to happen before the reload, and also to inform shadow that it can continue reloading our code. If we do any async stuff to reload code – let’s say, resolve a promise (something like `(-> clear-things (.then done))` – or if the reload uses some async code, this will not work. In this case, it’s better to register another callback, `:after-load`, and use it to re-activate our code, something like:

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12|`; On shadow-cljs.edn, under` `:devtools` `key:`<br><br>`{``:before-load-async` `calc.core/reload`<br><br> `:after-load` `calc.core/reactivate}`<br><br>`; On calc.core.cljs`<br><br>`(``defn` `reload` `[``done``]`<br><br>  `(done))`<br><br>`(``defn` `reactivate` `[``]`<br><br>  `; other code to re-activate things`<br><br>  `; that``'ll` `be run after reloaded`<br><br>  `(main))`|

## More about reloads

Now, this probably misses some points about why this is completely awesome. So, let’s imagine a different use-case: a `reload` function for an Atom editor’s package. In Atom, almost every side-effect function that you can call will return an `Disposable` object: one that you can dispose and nullify the effect (for instance, if you listen to changes in an editor, you can `.dispose()` those changes and un-listen to changes).

What you _can do_ is to create a `CompositeDisposable`, and when you refresh, you `.dispose` everything, re-creates the `CompositeDisposable`, then re-adds your effects. This means that while you’re developing your plug-in, you can stop subscriptions, delete your commands, add new commands to editor, re-load subscriptions, all just saving your code. If this is not **awesome**, I don’t really know what it is. The code to do it is surprisingly simple:

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16|`(``def` `CompositeDisposable`<br><br>  `(->` `"atom"`<br><br>      `js/``require`<br><br>      `.-CompositeDisposable))`<br><br>`(``def` `subscriptions (atom nil))`<br><br>`(``defn` `activate` `[``]`<br><br>  `(reset! subscriptions (CompositeDisposable.))`<br><br>  `(.add @subscriptions`<br><br>        `(.. js/atom -workspace`<br><br>            `(observeTextEditors #(prn` `"An event have happened!"``)))))`<br><br>`(``defn` `reload` `[``done``]`<br><br>  `(.dispose @subscriptions)`<br><br>  `(done)`<br><br>  `(activate))`|

For projects like Atom or other editors’ plug-ins (like VSCode, NeoVim, Oni) this give a temendous power. Also, this can be used while developing extensions for browsers, or projects when it’s tedious to unload everything, reload everything, just to see you have called a function with the arguments swapped…

## More Goodies

With a simple `npx shadow-cljs release app`, Shadow will try to compile your code with `:advanced` optimizations. But sometimes, this won’t work: imagine that in our jQuery example, `:advanced` will rename `jQuery` to something else, so things will not work. There are two options you can use: inside your build id (in our case, `:app`) you can add the key `:compiler-options {:infer-externs :auto}`. This will throw an warning when you’re using code that Shadow-CLJS can’t infer. Also, you can use `npx shadow-cljs release --debug app` to compile with `:advanced` features, but will pretty-print files and will make meaningful names (so you can debug where things went wrong).

This is not exclusive of Shadow-CLJS, but the compiler just make easier to access these options. ~~Now, what **is exclusive** from Shadow-CLJS is an extension to `ns` form~~ (UPDATE: it is not. As Thomas told me, it uses a different implementation and have more checks, but it works in CLJS too):

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4|`(``ns` `my.lib`<br><br>  `(``:require` `[``some-lib.core` `:as` `lib` `:include-macros` `true``]``))`<br><br>`(lib/some-macro ....)`|

This means no more `:refer-macros` or `:require-macros`. This magically finds the macros in that required namespace, and allows then to be used in the current ns. Simpler code, less things to remember, and cleaner. Also, I never had the compiler problems that I had with figwheel – that it compiles a version, then sometime later you clean things and it didn’t compile anymore. Or, that it compiles a version, and there’s a runtime exception with a compiler error.

Yet, for now, there are still problems with `.cljc` files that have macros, and some toolings that I was unable to make it work (cljs-complete being one of the most critical for me). Even with these limitations, developing with ClojureScript is now my primary choice for JS projects (when I can choose), even if there’s already JS code in production (because it integrates with existing code like a breeze too). It wasn’t before, and this was all thanks to a better tooling.

So yes, tools make all the difference!