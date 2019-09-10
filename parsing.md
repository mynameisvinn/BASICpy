# parsing
compared to lisp, whose internal representation closely matches lisp code, basic code is somewhat unwieldy. the basic interpreter will have to do a lot more work. 

### an execution trace
we have `tokens = ['20', 'LET', 'X', '=', 'X', '+', '1']` and want to parse it with `statement()`.

the first line, below, extracts line number from the expression.
```python
num  = linenumber()  # pops line number from line
```

### keyword
the second line, below, extracts the reserved keyword, in this case, the keyword is `LET`. (other keywords include `READ`, `GOTO`, `FOR`, `STOP`, etc, there are 15 keywords known by the interpreter.)
```python
typ  = pop(is_stmt_type) or fail('unknown statement type')
```
by now, `tokens` is the list `['X', '=', 'X', '+', '1']`. 

### populating args
```python
args = []
for p in grammar[typ]: # For each part of rule, call if callable or match if literal string
    if callable(p):  # cpython assumes all variables bound to functions are callable
        args.append(p())
    else:
        pop(p) or fail('expected ' + repr(p))
```
`typ` is `LET`, so we extract `LET`'s definition from the grammar book. `LET` is defined as:
```python
'LET': [variable, '=', expression]`
```
for the first element, `p` is `variable`. `variable` is callable, so we call `p()`, which will be appended to args. 

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
since the first element in `token` is not "(", we return "X" to frame 1. 

`=` is not callable so we pop `=` from the list thats being iterated (ie `for p in grammar[typ]`). 

at this point, `p` is `expression` and tokens is `['X', '=', 'X', '+', '1']`. 

finally, `p` is `expression`. `expression` is callable and is defined by the following:
```python
def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    return exp
```
`expression` returns an object `Opcall(x='X', op='+', y=1.0)]` this object is appended to `args`, bundled into another object `return Stmt(num, typ, args)`.