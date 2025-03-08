#ocaml #fp

Dune supports a [`diff` action](https://dune.readthedocs.io/en/stable/concepts.html#diffing-and-promotion) that compares two files aa and bb and fails if aa ≠ bb. The magic of this action is that it allows the user to set a:=ba:=b if aa is a source file and bb is a generated file. This is used under the hood for code formatting with `dune build @fmt`:

1. for each file aa, generate a formatted file bb;
2. assert that a=ba=b;
3. if not, the user may run `dune promote` to set a:=ba:=b.

The diff action can also be used to write 'self-correcting' tests: write a series of tests with some expected output; run each test and `diff` the output with an expected output; if any of the errors are expected, run `dune promote` to auto-correct the test.

One particularly useful type of 'self-correcting' test is an assertion of the output of the `--help` option to a binary. Snapshot the `--help` output in a `help.txt` file and `diff` it against the true `--help` output each time you run your tests. This has two advantages:

- you are certain of how any PR will change your program's CLI;
    
- the `help.txt` file serves as documentation that is guaranteed to be up-to-date.
    

For example, here's how it's being done in [CraigFe/oskel](https://github.com/CraigFe/oskel):

```dune
(rule
 (with-stdout-to
  oskel-help.txt.gen
  (run oskel --help=plain)))

(rule
 (alias runtest)
 (action
  (diff oskel-help.txt oskel-help.txt.gen)))
```