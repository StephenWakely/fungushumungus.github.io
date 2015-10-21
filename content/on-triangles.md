+++
date="2015-10-20"
title="On Triangles"
tags=["clojure"]
+++
 
In [SICP 1.2.2 Tree Recursion](ttps://mitpress.mit.edu/sicp/full-text/book/book-Z-H-11.html#%_sec_1.2.2) we have Exercise 1.12 which asks us to code up a recursive solution to compute the elements of Pascals Triangle.

```
    1
   1 1
  1 2 1
 1 3 3 1
1 4 6 4 1
```

Pascals Triangle has two rules - the numbers on the edge are 1 and the numbers inside the triangle is the sum of the two numbers in the previous row.

A recursive solution is fairly simple :

(in Clojure)

```language-clojure
(defn triangle
  [col row]
  (if (or (= col 0) (= col row))
       1
       (+ (triangle (dec col) (dec row))
          (triangle col (dec row)))))
```

If the column is the first or the last one return 0 otherwise recurse up to get the two values in the prior row. These are then summed and returned. We can print out our triangle with some nested loops :

```
(doseq [row (range 0 5)]
  (doseq [col (range 0 (inc row))]
    (print (triangle col row) " "))
  (println))

1  
1  1  
1  2  1  
1  3  3  1  
1  4  6  4  1  
```

This is all good, but quite unsatisfying. It would be better if we could get the entire row in a single call:

```
(defn pad
  [row]
  (concat '(0) row '(0)))

(defn triangle2
  [row]
  (if (= 0 row)
    '(1)
    (let [previous (pad (triangle2 (dec row)))]
      (map +' previous (next previous)))))
```

Here we recurse down to the first row, which returns `'(1)`. Then for each successive row we pad the prior one with 0, so `'(1 2 1)` becomes `'(0 1 2 1 0)`. And then we add that row with itself offset by one using `map`:

```
  (0 1 2 1 0)
  (1 2 1 0)
  -----------
  (1 3 3 1)
```

And we can print out our triangle :

```
(doseq [row (range 5)] (println (triangle2 row)))

(1)
(1 1)
(1 2 1)
(1 3 3 1)
(1 4 6 4 1)
nil
```


However this is hopelessly inefficient. For each row we have to calculate the prior rows, every time. It would be much better and more idiomatic if we could create a lazy sequence of these rows so that each step builds on the prior ones.

```
(defn lazy-triangle
  ([] (lazy-triangle '(1)))
  ([previous]
   (lazy-seq
       (cons previous
          (lazy-triangle (let [padded (pad previous)]
                           (map +' padded (next padded))))))))
```

Here we cons the previous row to a lazy-seq to build up our lazy sequence:

```
(doseq [row (take 5 (lazy-triangle))] (println row))

(1)
(1 1)
(1 2 1)
(1 3 3 1)
(1 4 6 4 1)
nil
```

The above function can be rewritten much more succintly using [iterate](http://clojuredocs.org/clojure.core/iterate) :

```
(def lazy-triangle2 (iterate (fn [previous]
                               (let [padded (pad previous)]
                                 (map +' padded (next padded))))
                             '(1)))
```

```
(doseq [row (take 5 lazy-triangle2)] (println row))

(1)
(1 1)
(1 2 1)
(1 3 3 1)
(1 4 6 4 1)
nil
```

## Sierpinski Triangle

Now the interesting thing about Pascals Triangle is that if you keep the odd numbers and clear out the even numbers, you end up with [Sierpinski's triangle](https://en.wikipedia.org/wiki/Sierpinski_triangle). Lets try it. Now instead of adding the rows together, we will add and then Mod with 2 to ensure we only have 1 or 0 in our triangle.

```
(defn +mod2
  [a b]
  (mod (+ a b) 2))

(def sierpinski-triangle (iterate (fn [previous]
                                    (let [padded (pad previous)]
                                      (map +mod2 padded (next padded))))
                                  '(1)))
```

```
(doseq [row (take 10 sierpinski-triangle)] (println row))

(1)
(1 1)
(1 0 1)
(1 1 1 1)
(1 0 0 0 1)
(1 1 0 0 1 1)
(1 0 1 0 1 0 1)
(1 1 1 1 1 1 1 1)
(1 0 0 0 0 0 0 0 1)
(1 1 0 0 0 0 0 0 1 1)
nil
```

Now you do have to squint a little in order to see the triangle there - ideally the rows would be centralised, but the triangle is there.

## Sierpinski Pyramid

Lets expand our pyramid into the third dimension.

Adding a new dimension means instead of each row being represented by a vector, it will now be a vector of vectors. Something like :

```
[[1]]
[[1 1]
 [1 1]]
[[1 1 1]
 [1 0 1]
 [1 1 1]]]
```

Iterating over each row will look something like :

```

(iterate next-level [[1]])

```

As before we first need to pad each row. This time we need to do it in 3 dimensions.

```
(defn pad
  [previous]
  (let [len (-> previous count (+ 2))]
    `(~(repeat len 0)
      ~@(map (fn [a] `(0 ~@a 0)) previous)
      ~(repeat len 0))))
```

We get the length (note each level of the pyramid is a square, the length and width will always be equal) of the previous level and add two (one for each side). Then we add a new list of zeros at the start and end of the previous rows - each of which is also mapped to add zeros around each row.

We are taking advantage of Clojures list templating features to help us build up our list. Usually these are used for macros, but they can be very useful to easily build up lists as well. 

When creating a 2d triangle, we would add the numbers from the two columns above in the preceeding level. Now that we are in 3d we need to add the four numbers from the above rows and columns in both directions. To acheive this we need to pair each vector within the level above:

```
(defn pairs
  [[x & xs]]
  (lazy-seq
   (cons [x (first xs)]
         (if (seq (rest xs))
           (pairs xs)
           nil))))
```

The pairs function creates a sequence of vectors of each row in the list together with the ensuing row.

```

 (pairs [[0 0 0] [0 1 0] [0 0 0]])

([[0 0 0] [0 1 0]]
 [[0 1 0] [0 0 0]])

```

Now that we have each pair of padded rows, we can add them up.

```

(map (fn [[a b]]
     (map +mod2 a b (next a) (next b)))
     pairs)

```

We have a nested map so we can get each pair of rows and each pair of columns within those rows. This gives us the four values above which combine to make the value for this column.

We can then put this all together with the following function.

```

(defn next-level
  [previous]
  (->> previous pad pairs add-rows))

```

We can now create our 3D pyramid.

```

(take 4 (iterate next-level [[1]]))

([[1]] [[1 1] [1 1]] [[1 0 1] [0 0 0] [1 0 1]] [[1 1 1 1] [1 1 1 1] [1 1 1 1] [1 1 1 1]])

```

This isn't a particularly pleasant way to view our pyramid. Minecraft would be a much better way. Using [Bukkure](https://github.com/SevereOverfl0w/bukkure) we can create plugins for Minecraft in Clojure.

Here are some of our pyramids :

![Sierpinski Pyramid](/img/shot1.png)

![Sierpinski Pyramid](/img/shot2.png)

Flying around a pyramid can be particularly satisfying.

![Sierpinski Pyramid](/img/pyramid.gif)


The source code for the plugin can be found [here](https://github.com/FungusHumungus/bukkure-fractalz).