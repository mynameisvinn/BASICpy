# parsing
compared to lisp, whose internal representation closely matches lisp code, basic code is somewhat unwieldy. the basic interpreter will have to do a lot more work. 

## an execution trace of a simple expression
we have `tokens = ['20', 'LET', 'X', '=', 'X', '+', '1']` and want to parse it with `statement()`, as defined:
```python
def statement():
    num  = linenumber()
    typ  = pop(is_stmt_type) or fail('unknown statement type')
    args = []
    grammar_rules = grammar[typ]
    for p in grammar_rules:
        if callable(p):
            args.append(p())
        else:
            pop(p) or fail('expected ' + repr(p))
    return Stmt(num, typ, args)
```
the goal of `statement()` is to take a list of tokens and return a single `Stmt` object. although `Stmt` is a single object, it retains the semantics of the list of tokens.

### extracting line number
the first line, below, extracts line number from the expression.
```python
num  = linenumber()
```
`linenumber()` is stateful: it will mutate the `tokens` and return an integer.

### keyword for grammar rules
the second line extracts the reserved keyword from `tokens` and assigns it to `typ`. in this case, the keyword is `LET`. (other keywords include `READ`, `GOTO`, `FOR`, `STOP`, etc, there are 15 keywords known by the interpreter.)
```python
typ  = pop(is_stmt_type) or fail('unknown statement type')
```
`typ` will be used to extract the appropriate grammar rule. (`LET`'s grammar rule is `[variable, '=', expression]`.)

additionally, `tokens` is `['X', '=', 'X', '+', '1']`. 

### populating args
```python
args = []
grammar_rules = grammar[typ]
for p in grammar_rules:
    if callable(p):  # cpython assumes all variables bound to functions are callable
        args.append(p())
    else:
        pop(p) or fail('expected ' + repr(p))
```
`typ` is used to extract the appropriate grammar rule into `grammar_rules`: `LET`'s grammar rule is `[variable, '=', expression]`.

we then iterate through `grammar_rules`.

the first element is `variable`. `variable` is callable (since it's a function previously), so we call `p()`. 

this kicks off frame 2 to execute `variable`.
```python
def variable(): 
    V = varname()
    if pop('('):
        indexes = list_of(expression)()
        pop(')') or fail('expected ")" to close subscript')
        return Subscript(V, indexes) # E.g. 'A(I)' => Subscript('A', ['I'])
    else: 
        return V  
```  
`V` is created by popping off the first element in `tokens`, which was `['X', '=', 'X', '+', '1']` and `[=', 'X', '+', '1']` afterwards. `V` points to the value `"X"`. since the first element in `token` is now `"="` and not `"("`, we return `V` to frame 1 and appended to `args`.

`args` is currently `["X"]`.

moving on to the next element in `grammar_rules`. `=` is not callable so we pop from `tokens`, which is currently `['=', 'X', '+', '1']`. `=` is not bound to anything and lost. `tokens` now points to `['X', '+', '1']`.

moving on to the final element in `grammar_rules`, which is `expression`. `expression` is callable so we call `expression()`:
```python
def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    return exp
```
according to norvig, `expression` is the most complicated part of the grammar because it must handle operator precedence. [NEEDS EXAMINATION]. ultimately, `expression` returns a single namedtuple `Opcall(x='X', op='+', y=1.0)]`, which is appended to `args`. 

`args` is finally completed and points to `['X', Opcall(x='X', op='+', y=1.0)]`.

### `statement()` returns a `Stmt` object
at the end of `statement()`, we have a few data objects: `num=20`, `typ='LET'`, and `args=['X', Opcall(x='X', op='+', y=1.0)]`. they are bundled into a single namedtuple `Stmt`.