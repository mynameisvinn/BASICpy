# parsing:
compared to lisp, whose internal representation closely matches lisp code, BASIC code is unwieldy. the BASIC parser will have to do a lot more work, particularly around operator precedence. 

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
the goal of `statement()` is to take a list of tokens and return a single namedtuple object `Stmt`. although `Stmt` is a single object (and obviously is syntatically different from `tokens`), the behavior/semantics remain the same.

## extracting line number
the first line, below, extracts line number from the expression.
```python
num  = linenumber()
```
`linenumber()` is stateful: it mutates `tokens` and return an integer.

## keyword for grammar rules
the second line extracts the reserved keyword from `tokens` and assigns it to `typ`. in this case, the keyword is `LET`. (other keywords include `READ`, `GOTO`, `FOR`, `STOP`, etc, there are 15 keywords known by the interpreter.)
```python
typ  = pop(is_stmt_type) or fail('unknown statement type')
```
`typ` will be used to extract the appropriate grammar rule. (`LET`'s grammar rule is `[variable, '=', expression]`.)

additionally, `tokens` is `['X', '=', 'X', '+', '1']`. remember `tokens` exists in the global environment and is therefore accessible by all functions.

## populating args
```python
args = []
grammar_rules = grammar[typ]
for p in grammar_rules:
    if callable(p):  # cpython assumes all variables bound to functions are callable
        args.append(p())
    else:
        _ = pop(p) or fail('expected ' + repr(p))
```
`typ` is used to extract the appropriate grammar rule into `grammar_rules`: `LET`'s grammar rule is `[variable, '=', expression]`.

we will now iterate through `grammar_rules`.

### step 1 of 3: evaluating `variable`
the first element is `variable`. `variable` is callable so we call and append `p()` to `args`. 
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
`V` is created with `varname()`:
```python
def varname():       
    return pop(is_varname) or fail('expected a variable name')
```
`pop()` returns the first token from `tokens` as long as the constraint `is_varname` is `True`. 

`is_varname` is defined as:
```python
def is_varname(x):
    return is_str(x) and len(x) in (1, 2) and x[0].isalpha() # A, A1, A2, B, ...
```

`pop()` is evaluated:
```python
def pop(constraint=None):
    top = peek()  # what is the next token in the list of tokens?
    if constraint is None or (top == constraint) or (callable(constraint) and constraint(top)):
        return tokens.pop(0)
```
from `peek()`, `top` is assigned to `"X"`, the first item in tokens. `is_varname` is passed as `constraint`: since `is_varname` is callable and `is_varname(top)` evaluates to `True`, `pop()` returns `"X"` to the calling function `varname()`. (after this pop operation, `tokens` is now `[=', 'X', '+', '1']`. in turn, `varname()` returns `"X"`, which is bound to the local variable `V`.

in recap, `variable()` was evaluated to `"X"`, which is then appended to `args`. `args` is currently `["X"]`.

### step 2 of 3: evaluating "="

moving on to the next element in `grammar_rules`. `=` is not callable so we pop from `tokens`. `=` is not bound to anything and lost. `tokens` now points to `['X', '+', '1']`.

### step 3 of 3: evaluating `expression`
moving on to the final element in `grammar_rules`, which is `expression`. 

`expression` is callable so we call `expression()`:
```python
def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    return exp
```
according to norvig, `expression` is the most complicated part of the grammar because it must handle operator precedence. [NEEDS EXAMINATION]. ultimately, `expression` returns a single namedtuple `Opcall(x='X', op='+', y=1.0)]`, which is appended to `args`. `args` is finally completed and points to `['X', Opcall(x='X', op='+', y=1.0)]`.

## `statement()` returns a `Stmt` object
finally, we have the following data objects: `num=20`, `typ='LET'`, and `args=['X', Opcall(x='X', op='+', y=1.0)]`. they are bundled into a single namedtuple `Stmt` and returned.
```python
def statement():
    ...
    return Stmt(num, typ, args)
```
