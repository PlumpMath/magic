MAGIC
=====
Morgan And Grand Iron Clojure

A functional Clojure compiler library and the start of a standalone Clojure implementation for the CLR.

Status
------
Experimental, unfinished, way pre-alpha. Here be dragons.

Overview
--------
MAGIC is a compiler for Clojure written in Clojure targeting the Common Language Runtime. Its goals are to:

1. Take full advantage of the CLR's features to produce better byte code
2. Leverage Clojure's functional programming and data structures to be more tunable, composeable, and maintainable

Strategy
--------
MAGIC consumes AST nodes from [`clojure.tools.analyzer.clr`](https://github.com/nasser/tools.analyzer.clr). It turns those AST nodes into [MAGE](https://github.com/nasser/mage) byte code to be compiled into executable MSIL. By using the existing ClojureCLR reader and building on [`clojure.tools.analyzer`](https://github.com/clojure/tools.analyzer), we avoid rewriting most of what is already a high performance, high quality code. By using MAGE, we are granted the full power of Clojure to reason about generating byte code without withholding any functionality of the CLR.

### Symbolizers
Clojure forms are turned into MAGE byte code using *symbolizers*. A symbolizer is a function that transforms a single AST node into MAGE byte code. For example, a static property like [`DateTime/Now`](https://msdn.microsoft.com/en-us/library/system.datetime.now(v=vs.110).aspx) would be analyzed by [`clojure.tools.analyzer.clr`](https://github.com/nasser/tools.analyzer.clr) into a hash map with a `:property` containing the correct [`PropertyInfo`](https://msdn.microsoft.com/en-us/library/system.reflection.propertyinfo(v=vs.110).aspx) object. The symbolizer looks like this:

```clojure
(defn static-property-symbolizer
  "Symbolic bytecode for static properties"
  [{:keys [property] :as ast} symbolizers]
  (il/call (.GetGetMethod property)))
```

It extracts the `PropertyInfo` from the `:property` key in the AST, computes the [getter method](https://msdn.microsoft.com/en-us/library/e17dw503(v=vs.110).aspx), and returns the [MAGE byte code for a method invocation](https://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.call(v=vs.110).aspx) of that method.

Note that this is not a side-effecting function, i.e. it does no actual byte code emission. It merely returns the *symbolic byte code* to implement the semantics of static property retrieval as pure Clojure data, and MAGE will perform the actual generation of runnable code as a final step. This makes symbolizers easier to write and test interactively in a REPL.

Note also that the symbolizer takes an additional argument `symbolizers`, though it makes no use of it. `symbolizers` is a map of keywords identifying AST node types (the `:op` key in the map `tools.analyzer` produces) to symbolizer functions. The basic one built into MAGIC looks like 

```clojure

(def base-symbolizers
  {:const               #'const-symbolizer
   :do                  #'do-symbolizer
   :fn                  #'fn-symbolizer
   :let                 #'let-symbolizer
   :local               #'local-symbolizer
   :binding             #'binding-symbolizer
   ...
```

Every symbolizer is passed such a map, and is expected it pass it down when recursively symbolizing. The symbolizer for `(do ...)` expressions does exactly this

```clojure
(defn do-symbolizer
  [{:keys [statements ret]} symbolizers]
  [(map #(symbolize % symbolizers) statements)
   (symbolize ret symbolizers)])
```

`do` expressions analyze to hash maps containing `:statements` and `:ret` keys referring to all expressions except the last, and the last expression respectively. The `do` symbolizer recursively symbolizes all of these expression, passing its `symbolizers` argument to them.

Early versions of MAGIC used a multi method in place of this symbolizer map, but the map has several advantages. Emission can be controlled from the top level by passing in a different map. For example, symbolizers can be replaced:

```clojure
(binding [magic/*initial-symbolizers*
          (merge magic/base-symbolizers
                :let #'my-namespace/other-let-symbolizer)]
(magic/compile-fn '(fn [a] (map inc a))))
```

or updated

```clojure
(binding [magic/*initial-symbolizers*
          (update magic/base-symbolizers
                :let (fn [old-let-symbolizer]
                       (fn [ast symbolizers]
                         (if-not (condition? ast)
                           (old-let-symbolizer ast symbolizers)
                           (my-namespace/other-let-symbolizer ast symbolizers)))))]
(magic/compile-fn '(fn [a] (map inc a))))
```

Additionally, symbolizers can change this map *before they pass it to their children* if they need to. This can be used to tersely implement optimizations, and some Clojure semantics depend on it. `magic.core/let-symbolizer` implements symbol binding using this mechanism.

Rationale and History
---------------------
During the development of [Arcadia](https://github.com/arcadia-unity/Arcadia), it was found that binaries produced by the ClojureCLR compiler did not survive Unity's AOT compilation process to its more restrictive export targets, particularly iOS, WebGL, and the PlayStation. While it is understood that certain more 'dynamic' features of C# are generally not supported on these platforms, the exact cause of the failures is difficult to pinpoint. Additionally, the byte code the standard compiler generates is not as good as it can be in situations where the JVM and CLR semantics do not match up, namely value types and generics. MAGIC was built primarily to support better control over byte code, and a well reasoned approach to Arcadia export.

Name
----
MAGIC stands for "Morgan And Grand Iron Clojure" (originally "Morgan And Grand IL Compiler"). It is named after the [Morgan Avenue and Grand Street intersection](https://www.google.com/maps/place/Grand+St+%26+Morgan+Ave,+Brooklyn,+NY+11237/@40.7133714,-73.9348001,17z/data=!3m1!4b1!4m2!3m1!1s0x89c25eab5ea3b021:0x77aaab63f0e3d135) in Brooklyn, the location of the [Kitchen Table Coders](http://kitchentablecoders.com/) studio where the library was developed. "Iron" is the prefix used for dynamic languages ported to the CLR (e.g. [IronPython](https://en.wikipedia.org/wiki/IronPython), [IronRuby](https://en.wikipedia.org/wiki/IronRuby)).

Legal
-----
Copyright © 2015-2017 Ramsey Nasser and contributers

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

```
http://www.apache.org/licenses/LICENSE-2.0
```

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
