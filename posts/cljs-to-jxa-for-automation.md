[Portal](https://github.com/djblue/portal) is a great dev helper for displaying REPL values.

Usually, I have more than one open, and it's a pain to find the one that belongs to the focused VSCode instance.

Instead of using the VSCode extension like a sane person, I decided to automate focusing the right portal window with a hotkey.

It turns out applescript can do what I want, but it's an *interesting* language with *great* documentation.
It turns out that everyting that applescript can do, can be used from javascript as well,
so after a few hours of frustration we get the following:

```javascript
function run(_input, _parameters) {
    var se = Application('System Events');
    var win = se.processes.whose({ frontmost: { '=': true } })[0].windows[0];
    var vsCodeName = win.name().split("—")[1]?.trim();

    const width = 800;
    const bounds = {
        "x": win.properties().position[0] + win.properties().size[0] - width,
        "y": 0,
        "width": width,
        "height": win.properties().size[1]
    };

    var browser = Application("Google Chrome");
    browser.activate();

    if (browser.running()) {
        browser.windows().some((window, _index) => {
            if (window.title().includes(vsCodeName)) {
                window.index = 1;
                window.bounds = bounds;
                return true;
            }
        });
    }

    return bounds;
}
```

This script looks at the title of the focused window, splits it by the "-"
and tries to find a chrome window whose title contains that. The found portal
window will then be placed and resized over the initial window.

You want to start your portal with your folder as a title:
`(p/open {:portal.launcher/window-title (System/getProperty "user.dir")})`

Usually I display it on a second montior just next to vscode, but that hard to
screencaputure and you can easily play with the position/size numbers to fit your
needs. You can also just remove the position/size assignment if you want to position
the window manually.

Here it is in action:

![alt text](/blog/assets/jxa-portal.gif)

Once I got that running I something drove me to say: well, that's just JS, I'd rather
write clojurescript.

So there is this old abandoned project [cljs-jxa-starter](https://github.com/blackgate/cljs-jxa-starter)
that would do the trick, right? It turns out it does not run anyway and uses
[lein-cljsbuild](https://github.com/emezeske/lein-cljsbuild) while I use deps.edn +
shadow-cljs. A few too many hours later, I present to you: [cljs-jxa-starter-shadow](https://github.com/Cyrik/cljs-jxa-starter-shadow).

Here's the above js as cljs:

```clojure
(ns cljs-jxa-starter.focus-resize-window
  (:require [clojure.string :as str]))

(def desired-width 800)
(def se (js/Application. "System Events"))

(defn main []
  (let [browser (js/Application. "Google Chrome")
        ^js win (aget ^js (.-windows (aget (.whose (.-processes se) #js{:frontmost #js{:= true}}) 0)) 0)
        vsCodeName (.name win)
        bounds #js{:x (- (+ (aget (.. win properties -position) 0) (aget (.. win properties -size) 0)) desired-width) 
                   :y 0 
                   :width desired-width 
                   :height (aget (.. win properties -size) 1)}]
    (when (.-running browser)
      (when-let [^js found (some #(when (str/includes? (.name %) vsCodeName) %) (.windows browser))]
        (.activate browser)
        (set! (.-index found) 1)
        (set! (.-bounds found) bounds)
        true))))

```

Yes, in jxa half the property access are functions the other half isn't, which is *great*.

Getting shadow-cljs to output a usable js file is pretty easy once you find the trick.
Just set :target as :browser and compile for release, otherwise jxa incompatible code
will be generated.

You can use it as a template to create your own cljs-jxa-shadow project and hopefully we
can use it as a springboard to more automation done the clojure way.
