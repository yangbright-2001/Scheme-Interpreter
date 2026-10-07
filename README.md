# Scheme Interpreter

A Scheme programming language interpreter project written in Python. It reads Scheme expressions, evaluates them, and prints the results. The repository also includes the project test suite and a browser-based editor.

## 1. Requirements

Python 3 is used to build this project. Turtle graphics use Tk, which ships with most Python installs. Install [Pillow](https://pypi.org/project/pillow/) only if you want headless rendering with `--pillow-turtle`.

## 2. Run the interpreter

Commands below are run from the repository root.

### 2.1 Interactive prompt

```bash
python3 scheme.py
```

```text
scm> (+ 1 2)
3
scm> (define (square x) (* x x))
square
scm> (square 4)
16
```

### 2.2 Run a file

Run a Scheme file, then exit:

```bash
python3 scheme.py tests.scm
```

Load a file and stay in the prompt:

```bash
python3 scheme.py -load questions.scm
```

### 2.3 Turtle graphics

Save a turtle drawing without opening a window:

```bash
python3 scheme.py --pillow-turtle --turtle-save-path drawing.png your-file.scm
```

### 2.4 Run the tests

```bash
python3 ok
```

Run one question:

```bash
python3 ok -q 01
```

Question names match the files in `tests/`, such as `eval_apply`, `01` through `16`, and `tests.scm`.

### 2.5 Web editor

```bash
python3 editor
```

This starts a local editor and opens it in the browser. The default port is `31415`.

```bash
python3 editor --nobrowser
python3 editor -p 8080
```

## 3. Background Knowledge: Scheme syntax

Scheme code is a tree of nested lists. The interpreter reads that tree and evaluates each expression in an environment. An environment is a chain of frames. Each frame maps names to values. Looking up a name starts in the current frame and continues through parent frames until the name is found.

Symbols are case-insensitive. `"Hello"` and `"hello"` are different strings, but `Square` and `square` are the same name. A semicolon starts a comment that runs to the end of the line.

```scheme
; this line is ignored
(+ 1 2)  ; so is the text after the semicolon
```

### 3.1 Literals

A literal evaluates to itself.

| Kind | Examples | Meaning |
| --- | --- | --- |
| Number | `3`, `-4`, `3.14` | An integer or a floating-point number |
| Boolean | `#t`, `#f`, `true`, `false` | True and false. Only `#f` is false; every other value counts as true |
| String | `"hello"` | An immutable string in double quotes |
| Empty list | `nil`, `()` | The empty list |
| Symbol | `x`, `square`, `+` | A name. Evaluation looks the name up in the environment |

### 3.2 Procedure calls

A call is a parenthesized list. The first element is the procedure. The rest are the arguments. The interpreter evaluates the procedure and the arguments, then applies the procedure.

```scheme
scm> (+ 1 (* 2 3))
7
scm> (print "hi")
hi
```

`(+ 1 (* 2 3))` is read as a list whose first element is the symbol `+`. That symbol looks up the addition procedure. The second argument is itself a call, so `(* 2 3)` runs first and produces `6`.

### 3.3 Definitions and procedures

`define` binds a name in the current frame. There are two forms.

```scheme
(define <name> <expression>)
(define (<name> <param> ...) <body> ...)
```

```scheme
scm> (define size 2)
size
scm> (define (square x) (* x x))
square
scm> (square 5)
25
```

The second form is shorthand for binding a `lambda`:

```scheme
(define square (lambda (x) (* x x)))
```

1. `lambda` creates a procedure and remembers the frame where it was created. A call evaluates the body in a new child of that frame, so free names use lexical scope.
2. `mu` is the same shape as `lambda`, but a call evaluates the body in a child of the caller's frame, so free names use dynamic scope.
3. `let` evaluates bindings in the current frame, then evaluates the body in a fresh child frame that holds those bindings.

```scheme
(lambda (<param> ...) <body> ...)
(mu (<param> ...) <body> ...)
(let ((<name> <expression>) ...) <body> ...)
```

```scheme
scm> (define y 1)
y
scm> (define f (mu (x) (+ x y)))
f
scm> (define g (lambda (x y) (f (+ x x))))
g
scm> (g 3 7)
13
scm> (let ((x 2) (y 3)) (+ x y))
5
```

In `(g 3 7)`, `f` is called from a frame where `y` is `7`, so the `mu` body sees that `y` and returns `6 + 7`.

### 3.4 Conditionals

`if` evaluates the test, then exactly one branch. The else branch may be omitted.

```scheme
(if <test> <consequent> <alternative>)
(if <test> <consequent>)
```

`cond` tries clauses in order. The first true test wins. A clause with only a test returns that test's value. `else` matches anything and must be the last clause.

```scheme
(cond (<test> <expression> ...)
      (<test> <expression> ...)
      (else <expression> ...))
```

`and` and `or` evaluate arguments from left to right and stop early. `(and)` is `#t`. `(or)` is `#f`. `and` returns the last value when every argument is true. `or` returns the first true value.

`begin` evaluates its expressions in order and returns the last value.

```scheme
scm> (if (> 3 1) 'yes 'no)
yes
scm> (cond ((> 2 3) 'a) ((= 1 1) 'b) (else 'c))
b
scm> (and 1 2 #f 4)
#f
scm> (or #f 0 5)
0
scm> (begin (define x 1) (+ x 2))
3
```

### 3.5 Quoting

`quote` returns its operand without evaluating it. A leading apostrophe is the same thing: `'(1 2)` is `(quote (1 2))`.

A backtick is `quasiquote`. A comma inside it is `unquote`, and that inner expression is evaluated. `unquote` outside a `quasiquote` is an error.

```scheme
scm> (quote (+ 1 2))
(+ 1 2)
scm> '(1 a)
(1 a)
scm> (define x 3)
x
scm> `(x is ,x)
(x is 3)
```

### 3.6 Lists

A list is either `nil` or a pair whose second half is a list. `cons` builds a pair. `car` is the first element. `cdr` is the rest. `list` builds a proper list ending in `nil`.

```scheme
scm> (cons 1 (cons 2 nil))
(1 2)
scm> (car '(1 2 3))
1
scm> (cdr '(1 2 3))
(2 3)
scm> (list 1 2 3)
(1 2 3)
```

`(1 . 2)` is a pair that is not a list, because its second half is the number `2`.

### 3.7 Built-in procedures

These names are already bound when the interpreter starts.

1. Arithmetic and comparison: `+`, `-`, `*`, `/`, `expt`, `abs`, `quotient`, `modulo`, `remainder`, `=`, `<`, `>`, `<=`, `>=`, `even?`, `odd?`, `zero?`, `not`.
2. Pairs and lists: `cons`, `car`, `cdr`, `list`, `append`, `length`, `map`, `filter`, `reduce`, `pair?`, `null?`, `set-car!`, `set-cdr!`.
3. Evaluation and files: `eval`, `apply`, `load`.
4. Output: `print`, `display`, `displayln`, `newline`, `error`.
5. Turtle drawing: `forward`, `left`, `right`, `circle`, `penup`, `pendown`, `color`, and related commands.

### 3.8 A short program

`questions.scm` defines `enumerate`, which pairs each element with its index:

```scheme
(define (enumerate s)
  (define (helper index lst)
    (if (null? lst)
        '()
        (cons (cons index (cons (car lst) nil))
              (helper (+ 1 index) (cdr lst)))))
  (helper 0 s))
```

```text
scm> (enumerate '(a b c))
((0 a) (1 b) (2 c))
```

## 4. Project Layout

| Path | Purpose |
| --- | --- |
| `scheme.py` | Read-eval-print loop |
| `scheme_eval_apply.py` | Expression evaluation and procedure application |
| `scheme_forms.py` | Special forms |
| `scheme_builtins.py` | Built-in procedures |
| `scheme_classes.py` | Pairs, frames, and procedures |
| `scheme_reader` | Tokenizer and parser |
| `questions.scm` | Scheme programs |
| `tests/` | Autograder tests |
| `editor/` | Browser editor |
| `abstract_turtle/` | Turtle drawing backend |
