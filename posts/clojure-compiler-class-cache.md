A follow-up to the last post when I still thought it would be easy to clear JVM soft
references. Spoiler alert, it's not.

## A quick background on JVM references:

The reason the JVM has more than one reference type is garbage collection. They are all
handled differently when the GC runs, so if you don't care about that, there is no reason
to create any of the non-default types.

### Strong references

These are the default references if you don't do anything special. As long as there is a
strong reference somewhere, the referenced object will not be garbage collected.

### Weak references

These exist so you can hold on to some object without owning it. This is useful for a
publish/subscribe or caching scenario where you want the objects to be collected if no
other code uses them anymore.

### Soft references

These work like strong references until the JVM needs the used memory.
This is the case when an OutOfMemory Exception would be thrown.
Right before that happens the GC runs and soft reference gets collected to hopefully
make enough space for the execution to continue.

### Phantom references

These work like weak references, except that they don't allow you to actually get
the object they reference. The use-case is a custom *finalize* workflow.

## Clojure class cache

The Clojure *DynamicClassLoader* has a map of all the classes it has currently loaded.
It holds those classes in soft references so that they do not get collected unless
there is no more memory to hold on to them.

This makes total sense since recompiling those classes is expensive but possible.

This does pose a problem for though if you want to build a tool that determines which
classes are actually reachable. As I mentioned above, the only way soft references get
cleaned up is when the memory fills up. This means any anonymous function that's not in
use anymore still stays around mostly forever.

## Solutions

### Blow up your memory

If the only way to get rid of objects in soft references is to fill up the memory, why not
just do that?

```clojure
(defn oom []
    (try (let [memKiller (java.util.ArrayList.)]
            (loop [free 10000000]
                (.add memKiller (object-array free))
                (.get memKiller 0)
                (recur (if (< (Math/abs (.. Runtime (getRuntime) (freeMemory))) Integer/MAX_VALUE)
                                (Math/abs (.. Runtime (getRuntime) (freeMemory)))
                                Integer/MAX_VALUE))))
            (catch OutOfMemoryError _
                (println "freed"))))
```

This function will produce empty arrays until there is no more space left. Sadly the
JVM garbage collector is pretty smart and will sometimes optimize this away when the array
is never accessed, so the *.get* is needed.

But even this does not always work since the GC is being smart again. It will not clean
up the soft references if they are too small to help with the OOM exception. This can
be fixed by creating even smaller arrays, but this will get pretty slow at some point.

The bigger problem is that your JVM instance will now expand its memory usage until your
repl hit's the default 4gig. This is not a very good user experience.

### JVM Tools Interface

The [JVM TI](https://docs.oracle.com/javase/8/docs/technotes/guides/jvmti/) allows you to get
all strong references to an object. So this would be an easy solution, right?

Sadly not every JVM has support for the Tools Interface and even worse, it's hard to use
since you have to write native code and connect that to the running JVM.

## Conclusion

At this point, there is no good way to find out which functions in the running VM are still
in use.

I'd love to be wrong about this, so if anyone has any more ideas I'm very open to suggestions.
