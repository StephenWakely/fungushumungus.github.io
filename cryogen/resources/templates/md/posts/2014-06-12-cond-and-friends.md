{:title "Cond and friends"
 :layout :post
 :tags  ["clojure"]}

There are a number of different cond's in Clojure.

###cond

The classic cond. This replaces the standard `if...else if....else` that you find in other languages. It takes a set of test and expression pairs. For the first test that evaluates to `true` it will evaluate and return its corresponing expression.

```
(cond (is-banana? me) "I am a banana"
      (is-slug? me) "I am a slug"
      :else "I am a turnip")

=> "I am a slug"      
```

###cond->

Now cond short circuits - it will stop at the first true expression. You may think "aaarg.. but more than one of my expressions may be true, I want all my true expressions to evaluate". In this case you probably want to use cond->. cond-> takes an initial form  and then a set of test and expression pairs. It will then thread that initial form through each expression where the test evaluates to true.

```
(cond-> "I am a " (is-slimey? me) (str "slimey ")
                  (is-banana? me) (str "banana")
                  (is-slug? me) (str "slug"))

=> "I am a slimey slug"                  
```

###cond->>

cond->> is pretty much the same as cond-> except cond-> threads first (puts the threaded param first in the argument list) and cond->> threads last (puns the threaded param last in the argument list).


```
(cond->> "I am a " (is-slimey? me) (str "slimey ")
                  (is-banana? me) (str "banana")
                  (is-slug? me) (str "slug"))

=> "slug slimey I am a"                  
```

###when
You might be thinking, "I don't want to thread an argument through all the true expressions - I just want to evaluate them". The only reason you would want to do this would be if those exressions had side effects and you wanted to ignore their results.

If you really want to ignore the results just use `when` :

```
(when (is-slimey? me) (send-slime))
(when (is-banana? me) (send-banana))
(when (is-slug? me) (send-slug))
```

###case
If you find you are writing this:

```
(cond (= me "banana") "I am a banana"
      (= me "slug") "I am a slug"
      :else "I am a turnip")
```

You should probably be using `case` instead. `case` takes an expression and thes a set of constant and expression pairs. For the first constant that is equal to the result of our expression the result of the corresponding expression is returned. There can be a single default expression at the end of our clauses that is evaluated and returned if none of the given constants are equal.

```
(case me
      "banana" "I am a banana"
      "slug" "I am a slug"
      "I am a turnip)
```

If none of the clauses match and there is no default you will get an IllegalArgumentException.

###condp

`case` is good and all, but that only uses = to determine a match. What if we wanted to use some other predicate instead - say a regex match - we could fall back to cond.

```
(cond (re-seq #"handsome" (looks me)) "Yes I am"
      (re-seq #"funky" (looks me)) "I'm ugly but at least I'm funky"
      :else "Oh dear")

=> "I'm ugly but at least I'm funky"      
```

However, here it would be neater to use `condp`. `condp` takes a predicate, a parameter (which is passed as the second parameter to the predicate) and a set of test expressions to result expressions. Then it runs through the clauses and passes each result of the test expression as the first parameter to the predicate. An example might make it clearer. The above can be rewritten as :

```
(condp re-seq (looks me)
       #"handsome" "Yes I am"
       #"funky" "I'm ugly but at least I'm funky"
       "Oh dear")

=> "I'm ugly but at least I'm funky"
```

`condp` is even better than that. If we separate our test clause and result clause with a `:>>` then the result of running the predicate with the test clause gets passed to the result clause - which must now be a function.

```
(condp re-find (looks me)
       #".*handsome.*" :>> #(str "Yes I am" %)
       #".*funky.*" :>> #(str "I'm ugly but at least I'm " %)
       "Oh dear")

=> "I'm ugly but at least I'm Big and funky"
```

