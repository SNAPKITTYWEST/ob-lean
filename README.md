<div align="center">

<h1>ob-lean4</h1>

<p><b>Run Lean 4 inside Org mode source blocks.</b></p>

[Installation](#installation) • [Quick start](#quick-start) • [Header arguments](#header-arguments) • [Contributing](#contributing) • [Author](#author)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](./LICENSE)
[![Emacs](https://img.shields.io/badge/Emacs-27.1%2B-7F5AB6?logo=gnuemacs&logoColor=white)](https://www.gnu.org/software/emacs/)
[![Lean 4](https://img.shields.io/badge/Lean-4-2F2F2F)](https://lean-lang.org/)
[![Org mode](https://img.shields.io/badge/Org-babel-77AA99)](https://orgmode.org/worg/org-contrib/babel/)
<br/>
[![Patreon](https://img.shields.io/badge/Patreon-f96854?style=flat-square&logo=patreon&logoColor=white)](https://www.patreon.com/0xAX)
[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-FFDD00?style=flat-square&logo=buymeacoffee&logoColor=black)](https://www.buymeacoffee.com/0xAX)

<img src="assets/demo.gif" alt="Lean 4 blocks evaluated in an Org document" width="900">

</div>

## Introduction

[Lean 4](https://lean-lang.org/) is a programming language and interactive theorem prover.

ob-lean4 allows Lean code to be executed directly within embedded code blocks in [Org-Mode](https://orgmode.org/) documents. These code blocks and their results can be included in an exported document.

The following covers details on how to implement Lean source code blocks in Org Mode documents. 

## Contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [Header arguments](#header-arguments)
  - [`:imports`](#imports)
  - [`:lake`](#lake)
  - [`:flags`](#flags)
  - [`:messages`](#messages)
  - [`:positions`](#positions)
  - [`:var`](#var)
  - [`:prologue` and `:epilogue`](#prologue-and-epilogue)
  - [`:session`](#session)
- [Errors](#errors)
- [Interactive editing](#interactive-editing)
- [Tangling and export](#tangling-and-export)
- [Customization](#customization)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## Installation

`ob-lean4` is a single file with no dependencies beyond Org itself. Clone it anywhere you like:

```sh
git clone https://github.com/0xAX/ob-lean.git ~/.emacs.d/site-lisp/ob-lean
rm -rf ~/.emacs.d/site-lisp/ob-lean/.git
```

Then put that directory on your `load-path` and require the file:

```elisp
(add-to-list 'load-path "~/.emacs.d/site-lisp/ob-lean")
(require 'ob-lean4)
```

The `load-path` entry is the *directory* holding `ob-lean4.el`, not the file itself. If you would rather not clone anything, dropping `ob-lean4.el` into a directory that is already on your `load-path` works just as well.

To load it lazily, so that Emacs reads the file only when a Lean block is first evaluated:

```elisp
(add-to-list 'load-path "~/.emacs.d/site-lisp/ob-lean")

(with-eval-after-load 'org
  (require 'ob-lean4))
```

With `use-package` and a local checkout:

```elisp
(use-package ob-lean4
  :load-path "~/src/ob-lean"
  :after org)
```

Or straight from the repository:

```elisp
(use-package ob-lean4
  :after org
  :vc (:url "https://github.com/0xAX/ob-lean" :rev :newest))
```

### Finding the toolchain

Blocks are run with the same `lean` that [`lean4-mode`](https://github.com/leanprover-community/lean4-mode) is configured to use, so that your blocks and your language server never disagree about which toolchain is in play. 

If `lean4-mode` is not loaded, `lean4-rootdir` is consulted, and failing that `lean` is looked up on `PATH`.

If you install Lean with [elan](https://github.com/leanprover/elan) and `lean` is on your `PATH`, there is nothing to configure.

## Quick start

Evaluate and inspect:

```org
#+begin_src lean4
#eval 2 + 2
#+end_src

#+RESULTS:
: 4
```

Imports can go at the top of the block, exactly as they would in a `.lean` file:

```org
#+begin_src lean4
import Std.Data.HashMap

#eval (Std.HashMap.emptyWithCapacity : Std.HashMap Nat Nat).size
#+end_src

#+RESULTS:
: 0
```

## Header arguments

In addition to the standard babel arguments, `ob-lean4` understands the following:

| Argument     | Values                     | Default | Purpose                                       |
|--------------|----------------------------|---------|-----------------------------------------------|
| `:imports`   | module names               | -       | Turn module names into `import` lines         |
| `:lake`      | `yes` `no`                 | `no`    | Run under `lake env lean`                     |
| `:flags`     | string                     | -       | Extra flags for `lean`                        |
| `:messages`  | `all` `info` `diag` `none` | `all`   | Which messages reach the result               |
| `:positions` | `yes` `diag` `no`          | `diag`  | Prefix messages with line and column          |
| `:session`   | name                       | -       | Recompile earlier blocks in front of this one |

`:results output` and `:exports both` are the defaults. Everything Lean has to say about a block arrives on its standard output, so `output` is the only meaningful `:results` type.

You can find information about each argument in the sections below.

### `:imports`

Space or comma-separated module names, turned into `import` lines:

```org
#+begin_src lean4 :imports Std.Data.HashMap
#eval (Std.HashMap.emptyWithCapacity : Std.HashMap Nat Nat).size
#+end_src

#+begin_src lean4 :imports "Std.Data.HashMap, Std.Data.HashSet"
#eval 1
#+end_src
```

Writing the `import` inside the block does the same thing. `:imports` is useful when many blocks need the same modules: set it once in a property line instead of repeating it in every block:

```org
#+PROPERTY: header-args:lean4 :imports Mathlib :lake yes :dir ~/src/myproject
```

### `:lake`

`:lake yes` runs the block as `lake env lean` instead of `lean`, which makes the dependencies of a [Lake](https://github.com/leanprover/lean4/tree/master/src/lake) project importable. 

Combine it with the standard `:dir` to point at the project root:

```org
#+begin_src lean4 :lake yes :dir ~/src/my-mathlib-project
import Mathlib.Analysis.SpecialFunctions.Log.Basic

#check Real.log
#+end_src

#+RESULTS:
: Real.log : ℝ → ℝ
```

This is how you get Mathlib into a document. The project must already be built (`lake build`), the same as for any other Lean tooling.

### `:flags`

Extra command line flags, split on whitespace and passed straight to `lean`:

```org
#+begin_src lean4 :flags "-DmaxHeartbeats=1000000"
#eval (List.range 100000).sum
#+end_src

#+begin_src lean4 :flags "-DwarningAsError=true"
#eval 1
#+end_src
```

### `:messages`

Which of Lean's messages end up in the result:

| Value  | Keeps                                                  |
|--------|--------------------------------------------------------|
| `all`  | everything (default)                                   |
| `info` | only `#eval` and `#check` output                       |
| `diag` | only warnings and errors                               |
| `none` | nothing - the block is compiled purely to typecheck it |

```org
#+begin_src lean4 :messages diag
#eval "swallowed"

theorem bad : False := by sorry
#+end_src

#+RESULTS:
: L3:8 warning: declaration uses `sorry`
```

`:messages none` is the way to say "check this, but do not talk about it" - the block is still compiled, and a failure is still reported.

### `:positions`

Whether a message is prefixed with where it came from:

| Value  | Effect                                          |
|--------|-------------------------------------------------|
| `diag` | only warnings and errors are prefixed (default) |
| `yes`  | everything is prefixed                          |
| `no`   | nothing is prefixed                             |

```org
#+begin_src lean4 :positions yes
#eval 41 + 1
#+end_src

#+RESULTS:
: L1:0 info: 42
```

```org
#+begin_src lean4 :positions no
theorem bad : False := by sorry
#+end_src

#+RESULTS:
: declaration uses `sorry`
```

Line numbers are relative to the block body, not to the file that was actually compiled, so whatever `:imports`, `:var`, `:prologue` or a session put in front of your code does not shift them. Three prefixes can appear:

| Prefix     | Means                                         |
|------------|-----------------------------------------------|
| `L3:12`    | line 3, column 12 of the block body           |
| `preamble` | from `:imports`, `:var` or `:prologue`        |
| `session`  | from one of the earlier blocks of the session |

### `:var`

Every `:var` becomes a `def` above the body:

```org
#+begin_src lean4 :var n=10 name="world" flag='t xs='(1 2 3)
#eval n * 2
#eval s!"hello, {name}"
#eval flag
#eval xs.length
#+end_src

#+RESULTS:
: 20
: "hello, world"
: true
: 3
```

Values come from anywhere Org can supply them - a table, a named block, a call line:

```org
#+name: primes
| 2 | 3 | 5 | 7 |

#+begin_src lean4 :var ps=primes
#eval ps.map List.sum
#+end_src

#+RESULTS:
: [17]
```

Emacs Lisp values are rendered as Lean literals like so:

| Elisp                 | Lean                           |
|-----------------------|--------------------------------|
| `t`                   | `true`                         |
| `nil`                 | `[]`                           |
| `42`, `3.5`           | `42`, `3.5`                    |
| `"text"`              | `"text"`                       |
| `sym`                 | `sym`                          |
| `'(1 2 3)`, `[1 2 3]` | `[1, 2, 3]` (nested lists too) |

Quote anything that is not a number or a string. Org reads a bare word in a header as the name of another block or table, so `:var flag=t` fails with `Reference 't' not found in this buffer`, while `:var flag='t` gives you `true`. The same goes for `'nil` and for symbols such as `'foo`.

### `:prologue` and `:epilogue`

The standard babel arguments work, and are slotted in underneath any `import` lines - Lean insists on seeing imports before anything else:

```org
#+begin_src lean4 :prologue "set_option pp.numericTypes true" :epilogue "#print axioms foo"
def foo : Nat := 1

#eval foo
#+end_src
```

### `:session`

Lean 4 has no REPL, so a session cannot be a live process. `:session` is supported anyway, in the only way it can be: **the blocks already run in a named session are compiled again in front of the block being run**, so that what they defined is in scope.

```org
#+begin_src lean4 :session work
def base : Nat := 21
#+end_src

#+begin_src lean4 :session work
def doubled : Nat := base * 2
#+end_src

#+begin_src lean4 :session work
#eval doubled
#+end_src

#+RESULTS:
: 42
```

Only the messages the current block produced become its result. The `#eval` output of the blocks replayed in front of it is dropped, since it was already reported when they were run - but anything that *goes wrong* in them still surfaces, tagged `session`.

Two things follow from the design and are worth knowing:

- **A block joins its session only once it compiles without errors.** A typo does not poison the session, so you do not have to undo anything by hand before the next block can run.
- **Running a block again replaces what it contributed**, rather than adding a second copy that Lean would reject as a duplicate definition. Blocks are told apart by their `#+name:` when they have one, and otherwise by their position among the src blocks of the buffer. Naming them is the robust option if you intend to move them around:

```org
#+name: definitions
#+begin_src lean4 :session work
def base : Nat := 21
#+end_src
```

`:session none` means no session, as elsewhere in babel.

#### Clearing a session

Running a block again replaces what it gave the session: it is compiled after the blocks that joined before it, not after its own earlier version. Deleting a block, or renaming it, does not retract what it already gave the session, and Lean will refuse a definition that arrives twice. Forget the session and re-run:

```
M-x org-babel-lean4-clear-session RET work RET    ; one session
M-x org-babel-lean4-clear-session RET RET         ; all sessions
```

or from Lisp:

```elisp
(org-babel-lean4-clear-session "work")
(org-babel-lean4-clear-session "")
```

`C-c C-v C-z` (`org-babel-switch-to-session`) deliberately signals an error: there is no process to switch to.

## Errors

Errors and warnings arrive like any other message, pointing at the line of the block they came from:

```org
#+begin_src lean4
#eval Nat.succ "not a nat"
#+end_src

#+RESULTS:
: L1:15 error: Application type mismatch: The argument
:   "not a nat"
: has type
:   String
: but is expected to have type
:   Nat
: in the application
:   Nat.succ "not a nat"
```

Output that is not valid JSON - Lake noise, a compiler panic - is kept verbatim, so nothing Lean printed can go missing. If Lean fails and says nothing that can be parsed at all, you get the status rather than a block that pretends to have succeeded:

```
: Lean exited with status 1
```

## Interactive editing

`C-c '` edits a block in a buffer that is not visiting a file, and the Lean language server has nothing to work with there: no goals, no infoview, no completion. `M-x org-babel-lean4-edit` opens the block in a real `.lean` file instead, so with `lean4-mode` the server starts and the block can be developed the way any Lean file is:

```org
#+begin_src lean4 :imports Lean
theorem mul_zero (a : Nat) : a * 0 = 0 := by
  skip  -- point here, M-x org-babel-lean4-edit, then C-c C-i for the goals
#+end_src
```

What the block is compiled with is written around it, so the server sees it the way running it would: the blocks of its `:session` that ran before it, `:imports`, `:var`, `:prologue` and `:epilogue`. Those lines sit in regions marked as generated, are read-only, and never make it back into the block:

```lean
-- ob-lean4: generated, not part of the block {{{
import Lean
-- ob-lean4: }}}
theorem mul_zero (a : Nat) : a * 0 = 0 := by
  skip  -- point here, M-x org-babel-lean4-edit, then C-c C-i for the goals
```

| Key       | Command                      | Effect                                       |
|-----------|------------------------------|----------------------------------------------|
| `C-c '`   | `org-babel-lean4-edit-exit`  | Write the buffer back into the block, return |
| `C-c C-q` | `org-babel-lean4-edit-abort` | Return without writing anything back         |
| `C-x C-s` | `org-babel-lean4-edit-save`  | Write the buffer back and save the Org file  |

Abort is not on `C-c C-k`, as it is in an Org src buffer, because `lean4-mode` uses that key to show how to type the symbol at point.

The file is written to `org-babel-lean4-edit-directory`, along with a `lean-toolchain` naming the toolchain `lean` runs, which is what makes the server treat that directory as a project. A block with `:lake yes` is edited inside its project instead, under `.lake/ob-lean4` of its `:dir`, so that the dependencies of the project are importable. The file is deleted when you leave the buffer.

The blocks of a session only exist once they have been run in the current Emacs, so run them before editing a block that needs them.

To have `C-c '` do this for Lean blocks and stay `org-edit-special` everywhere else:

```elisp
(defun my-org-edit-special ()
  "Edit Lean blocks with the language server, anything else as usual."
  (interactive)
  (if (member (car (org-babel-get-src-block-info 'light)) '("lean4" "lean"))
      (org-babel-lean4-edit)
    (call-interactively #'org-edit-special)))

(with-eval-after-load 'org
  (define-key org-mode-map (kbd "C-c '") #'my-org-edit-special))
```

## Tangling and export

`.lean` is registered as the extension for both `lean4` and `lean`:

```org
#+begin_src lean4 :tangle Main.lean
def main : IO Unit := IO.println "hello"
#+end_src
```

What `:imports`, `:var`, `:prologue` and `:epilogue` generate is part of the expansion and is tangled with the block. The other blocks of a `:session` are not: they are put in front of the block only when it is run, and copying them into every file the block is tangled into would be wrong.

The standard export knobs all work as usual:

```org
#+begin_src lean4 :exports results :wrap example
#eval "shown in the export without the source"
#+end_src
```

`:exports both` is the default, along with `:results output`. `:eval no`,
`:noweb yes`, `:cache`, `:dir`, `:wrap` and the rest behave the way they do for any other language.

## Customization

| Variable                              | Default                                       | Meaning                              |
|---------------------------------------|-----------------------------------------------|--------------------------------------|
| `org-babel-lean4-command-name`        | `"lean"`                                      | The Lean executable                  |
| `org-babel-lean4-lake-name`           | `"lake"`                                      | The Lake executable, for `:lake yes` |
| `org-babel-default-header-args:lean4` | `((:results . "output") (:exports . "both"))` | Default header arguments             |
| `org-babel-lean4-edit-directory`      | `ob-lean4` in `temporary-file-directory`      | Where `org-babel-lean4-edit` works   |

```elisp
(setq org-babel-lean4-command-name "lean"
      org-babel-lean4-lake-name "lake")

;; Quieter by default: only warnings and errors, everywhere.
(setq org-babel-default-header-args:lean4
      '((:results . "output") (:exports . "both") (:messages . "diag")))
```

Both executables are resolved through `lean4-rootdir` when `lean4-mode` is around, so pointing that at your elan root is usually all the configuration a
pinned toolchain needs.

## Contributing

Bug reports and pull requests are welcome - see
[CONTRIBUTING.md](./CONTRIBUTING.md).

## License

GPL-3.0-or-later. See [LICENSE](./LICENSE).

## Author

[0xAX](https://x.com/0xAX)
