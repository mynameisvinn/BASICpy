# parsing
compared to lisp, whose internal representation closely matches lisp code, BASIC code is unwieldy. the BASIC parser will have to do a lot more work, particularly around operator precedence. 

we have `tokens = ['20', 'LET', 'X', '=', 'X', '+', '1']` and want to parse it with `statement()`:
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
            _ = pop(p) or fail('expected ' + repr(p))
    return Stmt(num, typ, args)
```
`statement()` consumes `tokens`, a list of tokens that exists as a global variable, and returns a single namedtuple `Stmt`. although `Stmt` preserves semantics despite being syntactically different.

## extracting line numbers with `linenumber()`
we extract the line number.
```python
num  = linenumber()
...
def linenumber():    
    return (int(pop()) if peek().isnumeric() else fail('missing line number'))
```
`linenumber()` returns the line number as an integer. it also mutates `tokens`, just like any function that calls `pop()`. `tokens` is now `['LET', 'X', '=', 'X', '+', '1']`.

## extracting types for grammar rules
the second line in `statement()` extracts the reserved keyword from `tokens` and assigns it to `typ`:
```python
def statement():
    ...
    typ  = pop(is_stmt_type)
    ...
```
`pop()` takes the `is_stmt_type` as an argument:
```python
def is_stmt_type(x):  
    return is_str(x) and x in grammar  # LET, READ, ... (other keywords include `READ`, `GOTO`, `FOR`, `STOP`, etc, there are 15 keywords known by the interpreter.)

def pop(constraint=None):
    top = peek()  # in this case, top = 'LET'
    if constraint is None or (top == constraint) or (callable(constraint) and constraint(top)):
        return tokens.pop(0)
```
since `is_stmt_type` is callable and `is_stmt_type("LET")` evaluates to `True`, `pop(is_stmt_type)` returns `LET`, which is then bound to `typ`. `pop()` modifies `tokens` to `['X', '=', 'X', '+', '1']`.

## using `typ` to extract grammar rules
`typ` is bound to `'LET'` and then used to extract the corresponding grammar (ie `[variable, '=', expression]`).

```python
def statement():
    ...
    grammar_rules = grammar[typ]
    ...
```
`tokens` is still `['X', '=', 'X', '+', '1']`. 

## populating `args` according to grammar rules
```python
def statement():
    ...
    args = []
    ...
    for p in grammar_rules:
        if callable(p):
            args.append(p())
        else:
            _ = pop(p) or fail('expected ' + repr(p))
```
`grammar_rules` is `[variable, '=', expression]`, which is the rule corresponding to the `'LET'` keyword.


### step 1 of 3: evaluating `variable`
the first element in `grammar_rules` is `variable`. `callable(variable)` is `True` so it is evaluated in frame 2.
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
`V` is created by `varname()` in frame 3.
```python
def varname():       
    return pop(is_varname) or fail('expected a variable name')
```
`pop()` takes us to frame 4 and returns the first token from `tokens` if the constraint `is_varname` is `True`. 
```python
def is_varname(x):
    return is_str(x) and len(x) in (1, 2) and x[0].isalpha() # A, A1, A2, B, ...

def pop(constraint=None):
    top = peek()  # what is the next token in the list of tokens?
    if constraint is None or (top == constraint) or (callable(constraint) and constraint(top)):
        return tokens.pop(0)
```
`top` is assigned to `"X"`, the first item in `tokens`. 

since the function `is_varname` is callable and `is_varname(top)` evaluates to `True`, `pop()` returns `"X"` to `varname()`. (after this pop operation, `tokens` is now `[=', 'X', '+', '1']`.) finally, `varname()` returns `"X"`, which is bound to the local variable `V`.

in recap, the `variable()` evaluated to `"X"` and appended to `args`. `args` is currently `["X"]`.

### step 2 of 3: evaluating `"="`

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
