---
title: Clojure State and Identity
date: 2023-12-31
---

***Clojure Values*** are different from what you know about values we're used to and the term is used a lot by Clojurists.
1. Values are atomic, they form a single irreducible unit or component in a larger system
2. They're indivisible, unchanging, stable entities
3. Numbers for example are values, number 15 is number 15 and doesn't tend to mutate to another number 16.
4. Clojure's data structures are Values since they are immutable
5. Values do not change, but you can apply a process/function to it to produce a new Value.


**In OOP world**: we translate an identity as inherent to a changing object.

**In Clojure**: Idenitty is a ***succession*** of ***unchanging values*** produced by a ***process*** over ***time***


> We use the names to designate identities, for example, the name Ahmed refers to a series of individual states of Ahmed A1 A2 A3, and so on over time, and from that there's
> nothing such thing as mutable state, instead **state** means the **value** of an identity at a **point in time**.

## Clojure and Change
in Clojure, change only occurs when
1. a process generates a new value
2. we choose to associate the identity with the new value
and Clojure handles this sort of change using *reference types* that let you name identities and retrieve their state.

### Atoms
> *Atoms* are ideal for managing the state of independent identities, when we want to express that even should update the state of more than one identity simultaneously, *Refs* are the tool.
Clojure's atom reference type allows you to give a succession of related values with an identity.
```clojure
(def ahmed (atom {:level 0 :percent 0}))
```
the above ahmed atom refers to a value `{:level 0 :percent 0}` and that's its current state.

```clojure
@ahmed
; => {:level 0 :percent 0}
```
to update an Atom so that it refers to a new state we use `swap!`
> `swap!` implements *compare-and-set* semantics to ensure concurrency safety and two threads can't update an atom at the same time
to increase Ahmed's level by 1
```clojure
(swap! ahmed
       (fn [current-state]
         (merge-with + current-state {:level 1})))

@ahmed
; => {:level 1, :percent 0}
```
you can use the Clojure built-in function `update-in` to call swap! with extra arguments
```clojure
(swap! ahmed update-in [:level] + 10)

@ahmed
; => {:level 11, :percent 0}
```
Sometimes we might not need to check the current value, in this case, we can `reset!`
```clojure
(reset! ahmed {:level 0, :percent 0})
```

#### Atom Watch
From their name, *watches* allows you to check in for your reference types every change happens
the coming code example is self-explanatory
```clojure
(defn shuffle-speed
  [zombie]
  (* (:level zombie)
     (- 100 (:percent zombie))))

(defn shuffle-alert
  [key watched old-state new-state]
  (let [sph (shuffle-speed new-state)]
    (if (> sph 5000)
      (do
        (println "Run, you fool!")
        (println "The zombie's SPH is now " sph)
        (println "This message brought to your courtesy of " key))
      (do
        (println "All's well with " key)
        (println "Cuddle hunger: " (:level new-state))
        (println "Percent deteriorated: " (:percent new-state))
        (println "SPH: " sph)))))

(reset! ahmed {:level 22
               :percent 2})
(add-watch ahmed :ahmed-shuffle-alert shuffle-alert)
(swap! ahmed update-in [:percent] + 1)
; => All's well with  :ahmed-shuffle-alert
; => Cuddle hunger:  22
; => Percent deteriorated:  3
; => SPH:  2134

(swap! fred update-in [:cuddle-hunger-level] + 30)
; => Run, you fool!
; => The zombie's SPH is now 5044
; => This message brought to your courtesy of :fred-shuffle-alert
```

#### Atom Validator
Validators allow you to restrict what states are allowable
```clojure
(defn percent-deteriorated-validator
  [{:keys [percent-deteriorated]}]
  (or (and (>= percent-deteriorated 0)
           (<= percent-deteriorated 100))
      (throw (IllegalStateException. "That's not mathy!"))))
(def bobby
  (atom
   {:cuddle-hunger-level 0 :percent-deteriorated 0}
    :validator percent-deteriorated-validator))
(swap! bobby update-in [:percent-deteriorated] + 200)
; This throws "IllegalStateException: That's not mathy!"
```

### Update multiple identities using Refs & Transactions
We can update multiple identities using *refs & transactions* and these transactions have three features
1. They are *atomic*, all ***refs*** are updated or none of them are.
2. They are *consistent*, all the ***refs*** appear to have valid states.
3. They are *isolated*, the transactions behave as if they are executed serially. (when two threads are running two different transactions simultaneously that alter the same ref, one transaction will retry).
**These are the A, C, and I in ACID properties of a database, but in-memory with the same concurrency safety of a database.**
> Clojure uses *Software Transactional Memory* to implement the above features.

**Problem Statement**: We have a dryer with some socks and we need to express that a dryer has lost a sock and a gnome has gained a sock ***simultaneously***
> The sock should never appear to belong to both the dryer and the gnome, nor should it appear to belong to neither

```clojure
(def sock-varieties
  #{"darned" "argyle" "wool" "horsehair" "mulleted"
    "passive-aggressive" "striped" "polka-dotted"
    "athletic" "business" "power" "invisible" "gollumed"})

(defn sock-count
  [sock-variety count]
  {:variety sock-variety
   :count count})

(defn generate-sock-gnome
  "Create an initial sock gnome state with no socks"
  [name]
  {:name name
   :socks #{}})
```
Now let's define our refs
```clojure
(def sock-gnome (ref (generate-sock-gnome "Barumpharumph")))
(def dryer (ref {:name "LG 1337"
                 :socks (set (map #(sock-count % 2) sock-varieties))}))
```
Now let's solve our problem using a transaction, to begin a transaction we use `dosync` and to modify a ref we use `alter`
#### Alter

```clojure
(defn steal-sock
  [gnome dryer]
  (dosync
   (when-let [pair (some #(if (= (:count %) 2) %) (:socks @dryer))]
     (let [updated-count (sock-count (:variety pair) 1)]
       (alter gnome update-in [:socks] conj updated-count)
       (alter dryer update-in [:socks] disj pair)
       (alter dryer update-in [:socks] conj updated-count)))))
(steal-sock sock-gnome dryer)

(:socks @sock-gnome)
; => #{{:variety "passive-aggressive", :count 1}}
```


Here's a tricky example of alter
```clojure
(def counter (ref 0))
(future
  (dosync
   (alter counter inc)
   (println @counter)
   (Thread/sleep 500)
   (alter counter inc)
   (println @counter)))
(Thread/sleep 250)
(println @counter)
```
the above prints 1 then 0 then 2.

We can also modify the *refs* using *commute*
Here's how *alter* behaves inside a transaction (compare-and-set)
1. Read the *ref's* current state outside the transaction
2. Compare the current state to the state the ref started with within the transaction
3. If the two differ, make the transaction retry
4. If not, Commit

#### Commute

Commute however tries to be clever and make improve performance by not forcing a transaction to retry.
Here's how *commute* behave
1. Read the *ref's* current state outside the transaction
2. Run the commute function again using the current state
3. Commit the result
> *It’s important that you only use commute when you’re sure that it’s not possible for your refs to end up in an invalid state. Let’s look at examples of safe and unsafe uses of commute*
