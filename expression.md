# expression

## evaluate `exp = primary()`
imagine `tokens` points to `['X', '+', '1']`.
```python
# def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    # while precedence(peek()) >= prec:
        # op = pop()
        # rhs = expression(precedence(op) + associativity(op))
        # exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    # return exp
```
`exp` is bound to `primary()`, which is evaluated in frame 2:
```python
def primary():
    # if is_number(peek()):                   # '1.23' => 1.23 
        # return number()
    elif is_varname(peek()):                # X or A(I) or M(I+1, J)
        return variable()
    # elif is_funcname(peek()):               # SIN(X) => Funcall('SIN', 'X')
        # return Funcall(pop(), primary())
    # elif pop('-'):                          # '-X' => Funcall('NEG', 'X')
        # return Funcall('NEG', primary())
    # elif pop('('):                          # '(X)' => 'X'
        # exp = expression()
        # pop(')') or fail('expected ")" to end expression')
        # return exp
    # else:
        # return fail('unknown expression')
```

`peek()` returns `'X'` (`tokens` points to `['X', '+', '1']`), a valid variable name, we launch frame 3 to evaluate `variable()`.

```python
# def variable(): 
    V = varname()
    # if pop('('):
        # indexes = list_of(expression)()
        # pop(')') or fail('expected ")" to close subscript')
        # return Subscript(V, indexes) # E.g. 'A(I)' => Subscript('A', ['I'])
    # else: 
        # return V  
```  
`V` is created by `varname()`, which is executed in frame 4.
```python
def varname():  # helper function to guarantee a valid token can be used as a variable       
    return pop(is_varname) or fail('expected a variable name')

# varname() calls the following two subroutines
def is_varname(x):
    return is_str(x) and len(x) in (1, 2) and x[0].isalpha() # A, A1, A2, B, ...

def pop(constraint=None):
    # top = peek()  # what is the next token in the list of tokens?
    if constraint is None or (top == constraint) or (callable(constraint) and constraint(top)):
        return tokens.pop(0)
```
in frame 5, `pop(is_varname)` is `True` and it returns `X` to frame 4, whose calling function is `varname()`. 

frame 4 returns `'X'` to frame 3 (whose calling function is `variable()`), which is then bound to `V`. 

frame 3 returns `V` to frame 2 (whose calling function is `primary()`), which finally returns `X` to frame 1 (whose calling function is `expression()`). inside frame 1, `exp` points to `X`.

`tokens` is now `['+', '1']`.

## enter into while loop
```python
def expression(prec=1): 
    # exp = primary()
    while precedence(peek()) >= prec:
        # op = pop()
        # rhs = expression(precedence(op) + associativity(op))
        # exp = Opcall(exp, op, rhs)
    # return exp
```
`tokens` is `['+', '1']`, 

`peek()` is executed in frame 2. it evaluates to `'+'`.

we evaluate `precedence('+')` in frame 2.
```python
def precedence(op): 
    return (3 if op == '^' else 2 if op in ('*', '/', '%') else 1 if op in ('+', '-') else 0)
```
`op` is `'+'` so `precedence(peek())` evaluates to `1`. it is greater than or equal to `prec=1` so we continue inside the while loop.

## evaluate `op = pop()`
now we go through the body of the while loop.
```python
# def expression(prec=1): 
    # exp = primary()
    # while precedence(peek()) >= prec:
        op = pop()
        # rhs = expression(precedence(op) + associativity(op))
        # exp = Opcall(exp, op, rhs)
    # return exp
```
`pop()` removes the first item from `tokens`, which is `['+', '1']`. `op` is equal to `'+'`. `tokens` is now `['1']`.

## evaluating `rhs = expression(precedence(op) + associativity(op))`
lets evaluate `rhs` in frame 1. `tokens` is `['1']`.
```python
# def expression(prec=1): 
    # exp = primary()
    # while precedence(peek()) >= prec:
        # op = pop()
        rhs = expression(precedence(op) + associativity(op))
        # exp = Opcall(exp, op, rhs)
    # return exp

def associativity(op): 
    return (0 if op == '^' else 1)
```
in frame 2, `precedence(op)` returns `1` to frame 1 and exits. then, in frame 2, `associativity(op)` returns `1` to frame 1 and exits. as result, we evaluate `expression(2)` in frame 1.

## evaluate `expression(2)`
we launch frame 2 so we can evaluate `expression(2)`.
```python
def expression(prec=1): 
    exp = primary()                         # 'A' => 'A'
    while precedence(peek()) >= prec:
        op = pop()
        rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)          # 'A + B' => Opcall('A', '+', 'B')
    return exp

```

### evaluate `exp` in frame 2
`exp` is equal to `primary()`, which is evaluated in frame 3 and hits the following condition:
```python
def primary():
    if is_number(peek()):  # is the next token a number?
        return number()
    ...

def is_number(x):
    return is_str(x) and x and x[0] in '.0123456789'

def number():
    return (-1 if pop('-') else +1) * float(pop())  # return next token as a float 
```
since `tokens` is currently `['1']`, `is_number(peek())` evaluates to `True` and returns `1` to frame 2. inside frame 2, `1` is bound to `exp`. `tokens` is empty now, having been popped by `number()`.

### evaluate `while precedence(peek()) >= prec:` in frame 2
since `tokens` is empty, `peek()` returns `None` and `precedence(None)` evaluates to 0:
```python
def precedence(op): 
    return (3 if op == '^' else 2 if op in ('*', '/', '%') else 1 if op in ('+', '-') else 0)
```
this causes us to break out of the while loop, since 0 is not greater or equal than 2.

### returning `exp` to frame 1
`expression(2)` finally returns `exp`, which is equal to `1` to frame 1 and binds it to `rhs` in frame 1.

## evaluating `exp = Opcall(exp, op, rhs)` in frame 1
```python
def expression(prec=1): 
    # exp = primary()
    # while precedence(peek()) >= prec:
        # op = pop()
        # rhs = expression(precedence(op) + associativity(op))
        exp = Opcall(exp, op, rhs)
    # return exp
```
we have `exp` bound to `1`, `op` equal to `+`, and `rhs` equal to `1`. we now wrap these three symbols into a namedtuple of type `Opcall` and name `exp`.

## return `exp`
```python
def expression(prec=1): 
    # exp = primary()
    # while precedence(peek()) >= prec:
        # op = pop()
        # rhs = expression(precedence(op) + associativity(op))
        # exp = Opcall(exp, op, rhs)
    return exp
```