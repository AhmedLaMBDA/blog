---
title: Concurrent and Parallel Programming - in Clojure
date: 2023-12-31
---

* auto-gen TOC:
{:toc}

## Overview
### Elementary Concepts
Concurrent and parallel programming involves a lot of details from all levels of program execution, from the hardware to the operating system to programming
language libraries to the code.
#### Managing Multiple Tasks vs. Executing Tasks Simultaneously
**Concurrency**: Managing more than one task at the same time.

> *Note that this doesn't mean we're executing these tasks at the same time,
> but we can manage and switch context between more than one task*

**Parallelism**: Executing more than one task at the same time.

> *Note that Parallelism is a subclass of concurrency,
>  as before you can execute multiple tasks simultaneously, you have to manage them first*

> *You need multiple processors to be able to achieve parallelism*

**Distributed Systems**: They are a special version of parallelism, where processors are distributed across different machines and they communicate throw Network

#### Blocking vs Non-blocking
They're mostly referred to in I/O operations (reading a file/waiting for an HTTP request)

**Blocking**: Tasks are executed one after another, they are called *executed synchronously*

**Non-blocking**: Tasks don't wait for other tasks to be executed, they are called "executed asynchronously*

> *Concurrent and Parallel programming are the techniques for splitting a task into multiple subtasks,
> which can executed in parallel and manage the risks that arise from doing it as well*


### Threads
Serial code is a sequence of tasks, you indicate that tasks can be performed ***concurrently*** by putting them on a JVM Thread.

**A thread** is a sub-program, that executes its instructions while sharing access to the whole program's state.

Since Clojure is built on the JVM, The JVM provides its platform-independent thread management functionality.

**Example:**

***Thread A*** consists of multiple instructions **A1 A2 A3 A4** that needs to be executed sequentially

***Thread B*** consists of multiple instructions **B1 B2 B3 B4**

***In a single-core processor executing a single-threaded program A***

The order of execution will be: *A1 -> A2 -> A3 -> A4* - deterministic

***In a single-core processor executing two threads A-B***

The order of execution becomes *non-deterministic*, as you can't know when the processor will switch context from A to be or from B to A
It could be:
- A1 -> A2 --> B1 -> B2 -> B3 --> A3 -> A4 --> B4
- A1 --> B1 -> B2 --> A2 -> A3 -> A4 --> B3 -> B4
- And so on...

***In two-core processor executing a two-threaded program A-B***

The process becomes deterministic again since the two processors will execute the two threads ***simultaneously***


### The Three Challenges
1. Reference Cells
2. Mutual Exclusion
3. Deadlock (Dwarven Berserkers) - The dining philosophers problem

#### Instructions for a Program with a Nondeterministic Outcome

| ID |	Instruction        |
| -- |  -----------------  |
| A1 |	WRITE X = 0        |
| A2 |	READ X             |
| A3 |	WRITE X = X + 1    |
| B1 |	READ X             |
| B2 |	WRITE X = X + 1    |

If the processor follows *the order A1, A2, A3, B1, B2, then X will have a value of 2*, as you’d expect. 
But if it follows *the order A1, A2, B1, A3, B2, X’s value will be 1*.

> *The reference cell problem happens when two threads can read and write to the same location*

#### Mutual Exclusion
> *Two threads fight for access to some resource without any way to claim exclusive access to that resource.*
For example, a write to a file, where both threads write for some time, the end result of the file will be corrupted with the function of the two threads

#### The dining philosophers problem - Clear. Right?!!!!

## Clojure Concurrency Model

In a serial code execution, we bind these three events together
1. Task definition
2. Task execution
3. Requiring the task's result

```clojure
(web-api/get :some-api-call)
```
When Clojure sees the above, it executes it and waits for the result, *blocking* until the API call finishes.

> **Clojure tries to decouple these chronological bindings through *futures*, *delays* and *promises*, which allow us to separate task definition, task execution, and requiring the result**

### Futures
*To **define** a task and place it on another thread without requiring the result immediately, you use future*

```clojure
(future (Thread/sleep 4000)
        (println "I'll print after 4 seconds"))
(println "I'll print immediately")
```

***Concepts***

1. Future function returns a *reference value*
2. Use *reference value* to request a *future*'s result
3. If the *future* isn't done computing the result while requesting, you'll have to wait
4. You can pass `deref` a number of milliseconds to wait along with the value to return if the `deref` times out
```clojure
(deref (future (Thread/sleep 1000) 0) 10 5)
; => 5
```
5. Dereferencing the *future* can be done using `deref` function or the `@` reader macro
6. A *future*'s result is the last expression evaluated in its body
7. *Future*'s body executes only once, and its value gets cached

```clojure
(let [result (future (println "this prints once")
                     (+ 1 1))]
  (println "deref: " (deref result))
  (println "@: " @result))
; => "this prints once"
; => deref: 2
; => @: 2
```

```clojure
(let [result (future (Thread/sleep 3000)
                     (+ 1 1))]
  (println "The result is: " @result)
  (println "It will be at least 3 seconds before I print"))
; => The result is: 2
; => It will be at least 3 seconds before I print
```
8. You can interrogate a future using `realized?` to see if its done running

```clojure
(realized? (future (Thread/sleep 1000)))
; => false

(let [f (future)]
  @f
  (realized? f))
; => true
```

### Delays
*Allow you to **define** a task **without** having to **execute** it or **require** the result immediately*
```clojure
(def jackson-5-delay
  (delay (let [message "Whatever, just call me"]
           (println "First deref:" message)
           message)))
```
Nothing is printed!!! as expected

***Concepts***

1. You can evaluate the delay and get its result by dereferencing it or by using force
```clojure
(force jackson-5-delay)
; => First deref: Whatever just call me
; => "Whatever, just call me"
```
2. Run only once, and it's result is cached
```clojure
@jackson-5-delay
; => "Whatever, just call me"
```

#### Using Future & Delay to Prevent Mutual Exclusion
Suppose you have a list of images you want to upload
```clojure
(def images ["serious.jpg" "fun.jpg" "playful.jpg"])
```
and you want to notify the owner as soon as the first one is up through a mail server that sends emails
```clojure
(defn email-user
  [email-address]
  (println "Sending image notification to" email-address))
(defn upload-document
  "Needs to be implemented"
  [image]
  true)
```
Now let's combine *future* so we can upload the documents and use *delay* to guard the email server resource from mutual exclusion that might happen from these threads created by the *future*s
```clojure
(let [notify (delay (email-user "and-my-axe@gmail.com"))]
  (doseq [image images]
    (future (upload-document image)
            (force notify))))
```
The body of the *delay* gets evaluated the first time one of the *future*s created by the doseq form evaluates (force notify). Even though (force notify) will be evaluated three times, the delay body is evaluated only once

Because the body of the *delay* gets evaluated only once, the mail server is protected from the *future*s threads trying to send the same email.

### Promises
*The ultimate **abstraction**, **promises** allow you to express that you **expect** a result **without** having to **define the task** that should produce it or when the task should run. Create using **promise** and deliver results using **deliver*** 
```clojure
(def my-promise (promise))
(deliver my-promise (+ 1 2))
@my-promise
; => 3
```
**Concepts**
1. You can deliver a result to a *promise* only once
2. A program will block if you try to deref a promise until you deliver a result to it
3. For the promise deref not to block forever, it is better supplied with a timeout and a default return value
```clojure
(let [p (promise)]
  (deref p 100 "timed out"))
```

#### Using Future and Promise to Prevent Reference Cell Concurrency
Suppose you have a list of API calls, and you need to update some value once any of these calls succeed.
You can view  these calls as independent threads each trying to access some cell by updating it, we can prevent this from happening by using *Promise* since we can only write once to a promise
```clojure
(def yak-butter-international
  {:store "Yak Butter International"
    :price 90
    :smoothness 90})
(def butter-than-nothing
  {:store "Butter Than Nothing"
   :price 150
   :smoothness 83})
;; This is the butter that meets our requirements
(def baby-got-yak
  {:store "Baby Got Yak"
   :price 94
   :smoothness 99})

(defn mock-api-call
  [result]
  (Thread/sleep 1000)
  result)

(defn satisfactory?
  "If the butter meets our criteria, return the butter, else return false"
  [butter]
  (and (<= (:price butter) 100)
       (>= (:smoothness butter) 97)
       butter))
```
If we run the above synchronously
```clojure
(time (some (comp satisfactory? mock-api-call)
            [yak-butter-international butter-than-nothing baby-got-yak]))
; => "Elapsed time: 3002.132 msecs"
; => {:store "Baby Got Yak", :smoothness 99, :price 94}
```
If we used Promise and future
```clojure
(time
 (let [butter-promise (promise)]
   (doseq [butter [yak-butter-international butter-than-nothing baby-got-yak]]
     (future (if-let [satisfactory-butter (satisfactory? (mock-api-call butter))]
               (deliver butter-promise satisfactory-butter))))
   (println "And the winner is:" @butter-promise)))
; => "Elapsed time: 1002.652 msecs"
; => And the winner is: {:store Baby Got Yak, :smoothness 99, :price 94}
```

#### Using Future and Promise to register Callbacks

```clojure
(let [ferengi-wisdom-promise (promise)]
  (future (println "Here's some Ferengi wisdom:" @ferengi-wisdom-promise))
  (Thread/sleep 100)
  (deliver ferengi-wisdom-promise "Whisper your way to success."))
; => Here's some Ferengi wisdom: Whisper your way to success.
```
This example creates a future that begins executing immediately. However, the future’s thread is blocking because it’s waiting for a value to be delivered to ferengi-wisdom-promise. After 100 milliseconds, you deliver the value and the println statement in the future runs.

### Queue Time, heeeeh!
One characteristic The Three Concurrency Goblins have in common is that they all involve tasks concurrently accessing a shared resource—a variable, a printer in an uncoordinated way. If you want to ensure that only one task will access a resource at a time, you can place the resource access portion of a task on a queue that’s executed serially.

Let's implement that

First, we will create a macro wait, that just waits for some milliseconds and then executes the body
```clojure
(defmacro wait
  "Sleep `timeout` seconds before evaluating body"
  [timeout & body]
  `(do (Thread/sleep ~timeout) ~@body))
```
Now let's say we have three tasks, each task part of them needs to occur sequentially and another part can be executed concurrently.
```clojure
(let [saying3 (promise)]
  (future (deliver saying3 (wait 100 "Cheerio!")))
  @(let [saying2 (promise)]
     (future (deliver saying2 (wait 400 "Pip pip!")))
     @(let [saying1 (promise)]
        (future (deliver saying1 (wait 200 "'Ello, gov'na!")))
        (println @saying1)
        saying1)
     (println @saying2)
     saying2)
  (println @saying3)
  saying3)
```
In the above, we need to make sure that the printing of saying1 happens before the printing of saying2 and so on while we also wait for the part of the task that can occur concurrently (which is waiting for what to be printed).

We ensure that by creating a promise for each task (the printing part) to create a corresponding future that will deliver a concurrently computed value to the promise.

let's create a macro that does exactly the above
and work as follows:
```clojure
(-> (enqueue saying (wait 200 "'Ello, gov'na!") (println @saying))
    (enqueue saying (wait 400 "Pip pip!") (println @saying))
    (enqueue saying (wait 100 "Cheerio!") (println @saying)))
(defmacro enqueue
   ([q concurrent-promise-name concurrent serialized]
    `(let [~concurrent-promise-name (promise)]
      (future (deliver ~concurrent-promise-name ~concurrent))
       (deref ~q)
      ~serialized
      ~concurrent-promise-name))
   ([concurrent-promise-name concurrent serialized]
   `(enqueue (future) ~concurrent-promise-name ~concurrent ~serialized)))

```
