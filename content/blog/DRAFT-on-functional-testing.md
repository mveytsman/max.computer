---
title: "On Functional Testing"
date: 2014-06-29
comments: true
categories:
draft: true
---

## Next Steps
### Refactor
The core of my emulator is a parser that parses a 16-bit MSP430 instruction, and then dispatches to the correct function. The meat is then an implementation of the instruction set, with each instruction being a function that takes a `computer` state along with some arguments and retruns a `computer` state after applying the function. I use a multimethod to dispatch on the OP name.

For instance below is the implemtation of push.
```clojure
;; unary-op is a multimethod that performs a single operand OP. OP is parameterized
;; by the first argument (a symbol)
;; These always puts the result in the register directly, which is a bug I believe?
(defmulti unary-op (fn [op _ _ _ _] op))
;; ...
(defmethod unary-op :PUSH
  [_ computer byte? source-mode register]
  (let [[val computer] (get-value computer source-mode byte? register)]
  (stack-push computer (make-word val))))
```
As it turns out, this architecture means that I have a lot of repeating code (identical let bindings inside of every OP function) and am passing around quote a lot of state on every call. The `source-mode` represents register access mode. There are called direct, indirect (pointer), inexed (pointer with an index), and post-increment. Each instruction can be byte or word addressed, tracked with the `byte?` variable.

I paired with a couple of hacker-schoolers and talked about strategies for refactoring.

The code I would like to write looks like this
```clojure
(def-unary-op PUSH :opcode "100"
  [computer register value]
  (stack-push computer (make-word val)))
```
The most effective way is to use a macro to take care of pulling the value of a given register (depending on addressing mode and `byte` flag).

### The Web
I can add a x on the end of my `clj` file names, and this code will run on the web, right?

I really like the idea of making a nice web front-end to a debugger that runs in a browser (unlike the hosted one used for microcorruption). Woud be cool to try Om/React.

### Push-button exploits
I've spent the past few days reading about SMT solveers 


