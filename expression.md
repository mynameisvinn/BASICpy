# expression
imagine `tokens` points to `['X', '+', '1']`.
```python
def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    return exp
```
in the first line, `exp` is bound to `primary()`:
```python
def primary():
    "Parse a primary expression (no infix op except maybe within parens)."
    if is_number(peek()):                   # '1.23' => 1.23 
        return number()
    elif is_varname(peek()):                # X or A(I) or M(I+1, J)
        return variable()
    elif is_funcname(peek()):               # SIN(X) => Funcall('SIN', 'X')
        return Funcall(pop(), primary())
    elif pop('-'):                          # '-X' => Funcall('NEG', 'X')
        return Funcall('NEG', primary())
    elif pop('('):                          # '(X)' => 'X'
        exp = expression()
        pop(')') or fail('expected ")" to end expression')
        return exp
    else:
        return fail('unknown expression')
```
in frame 2, since `peek()` returns `'X'`, a valid variable name, we launch frame 3 for `variable()`.
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
`V` is created by `varname()` in frame 4.
```python
def varname():       
    return pop(is_varname) or fail('expected a variable name')
```
in frame 5, `pop()` returns the first token from `tokens` if the constraint `is_varname` is `True`. 
```python
def pop(constraint=None):
    top = peek()  # what is the next token in the list of tokens?
    if constraint is None or (top == constraint) or (callable(constraint) and constraint(top)):
        return tokens.pop(0)

def is_varname(x):
    return is_str(x) and len(x) in (1, 2) and x[0].isalpha() # A, A1, A2, B, ...
```
in this example, `'X'` is returned by `pop()` to frame 4 (`varname()`). frame 4 returns `'X'` to frame 3, which is then bound to `V` in frame 3. frame 3 (`variable()`) then returns `V` to frame 2. frame 2, which is `primary()` finally returns `X` to frame 1, which is the function `expression()`. `tokens` is now `'+', '1']`.

## while loop
`tokens` is now `'+', '1']`.
```python
def expression(prec=1): 
    ...
    while precedence(peek()) >= prec:
        ...
```
`tokens` is `'+', '1']`, `peek()` returns `'+'` so we evaluate `precedence(peek())` in frame 2.
```python
def precedence(op): 
    return (3 if op == '^' else 2 if op in ('*', '/', '%') else 1 if op in ('+', '-') else 0)
```
since `op` is `'+'`, `precedence(peek())` returns `1` to frame 1.

```python
def expression(prec=1): 
    ...
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)  
```
now we go through the body of the while loop.

`op` is equal to `+`, since `pop()` simply removes the first item from `tokens`. `tokens` is now `['1']`.

### rhs
lets evaluate the following, where `tokens` is `['1']`.
```python
rhs = expression(precedence(op) + associativity(op))
```
in frame 2, `precedence(op)` returns `1` and exits. 

in frame 2, `associativity(op)` returns `1` and exits:
```python
def associativity(op): 
    return (0 if op == '^' else 1)
```
in frame 1, we now evaluate:
```python
rhs = expression(precedence(op) + associativity(op))  # rhs = expression(2)
```
in frame 2, we evaluate `expression(2)`.
```python
def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    return exp

```
`exp` is equal to `primary()`, which hits the following condition:
```python
def primary():
    "Parse a primary expression (no infix op except maybe within parens)."
    if is_number(peek()):                   # '1.23' => 1.23 
        return number()
    ...
```