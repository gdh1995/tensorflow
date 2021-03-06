# AutoGraph reference

[Index](index.md)

## Limitations

When AutoGraph is applied to normal Python code, you should expect no change
in functionality.
However, when applied to TensorFlow control flow (for example, an if statement
with a `tf.Tensor` condition), there are certain limitations. This section
describes these limitations and practices that will allow you to avoid them.

### Indirect modifications and hidden side effects in TensorFlow control flow

<!-- TODO(mdan) Refine this paragraph well - it's important -->
Key Point: We recommend using functional style and immutable Python collections.

#### AutoGraph analyzes code to detect modifications

One of the most important functions of AutoGraph is to rewrite Python control
flow statements into equivalent TensorFlow ops. This process requires "wiring"
variables in the Python code whose values are affected these statements control
flow into the respective ops.

Note: Python variables should not be confused with TensorFlow variables.

The examples below use a `while` loop, but the same notions extend to all
control flow: `if` and `for` statements.

In the example below, `x` needs to become a _loop variable_ of the
corresponding `tf.while_loop':

```
while x > 0:
  x = x - 1
```
```
x = tf.while_loop(..., loop_vars=(x,)
```

TF control ops support only a limited set of types for loop variable. At the
same time, the efficiency of TensorFlow graphs is influenced by the number of
loop variables, so we don't want to create them unnecessarily. For this reason,
AutoGraph only pulls symbols through loop variables if necessary.

Note: If a symbol refers to a nested structure, such as a `dict` of `dict`s,
then when that symbol is added to the loop variables the entire structure
becomes part of the loop variables - TensorFlow automatically unpacks it.

For example, the symbol 'y' below is not wired through the `tf.while_loop`'s
`loop_vars` because it is not affected by the while loop:

```
y = 0
while x > 0:
  x = x - 1
print(y)
```
```
x = tf.while_loop(..., loop_vars=(x,)  # y does not need to be a loop variable
```

AutoGraph uses static analysis to determine which symbols are modified by the
code, in order to transform them into control flow variables. Static analysis
is generally performed on single functions - Python's dynamic nature limits its
effectiveness across functions.

#### Modifications are not detected across functions

Because static analysis is limited to single functions, modifications that are
performed in other functions are not visible to AutoGraph:

```
def change_y():
  global y
  y = y + 1

while x > 0:
  change_y()  # Problem -- change made to y is not visible here!
```

This can be easily remedied using functional style - writing functions that take
their inputs as arguments, and return everything they calculate as return
values:

```
def change(y):
  y = y + 1
  return y

while x > 0:
  y = change(y)  # Okay -- y can now be properly tracked!
```

#### Modifications are not detected in methods

A special case of hidden side effects are methods, which are commonly used
to change the value of objects:

```
def MyClass(object):
  def change(self):
    self.y += 1

c = MyClass()
while x > 0:
  c.change()  # Problem -- modification to c.y is not visible here!
```

This can be addressed in a number of ways.

One possibility is to operate directly on the object properties:

```
c = MyClass()
while x > 0:
  c.y += 1  # Okay -- c.y can now be properly tracked!
```

Another possibility is to rely on immutable objects. This may lead to many
temporary objects when executing eagerly, but their number is greatly reduced
in `@tf.function`:

```
def MyClass(object):
  def change(self):
    self.y += 1
    return self

c = MyClass()
while x > 0:
  c = c.change()  # Okay -- c is now a loop var.
```

Note: TensorFlow control flow does not currently support arbitrary Python
objects, but it does support basic collection objects such as `list`, `dict`,
`tuple`, `namedtuple` and their subclasses. Design your objects as subclasses
of [namedtuple](https://docs.python.org/3/library/collections.html#collections.namedtuple).

### Python collections in TensorFlow control flow

Key Point: Use TensorFlow collection classes instead of Python collections.
Python collections are okay to use when they represent a fixed structure (that
is, `list`s don't change length, `dict`s don't add or remove keys).

#### Modifying Python collections in TensorFlow control flow is not allowed

One of the advantages of eager execution is that you may use the usual Python
collections, like `list` or `dict` to hold `tf.Tensor` values. However, these
are generally not compatible with TensorFlow control flow. Specialized
collections like `tf.TensorArray` are required.

Consider the following example:

```
def fn():
  l = []

  def loop_cond(i):
    return i < 10

  def loop_body(i):
    i = i + 1
    l.append(i)
    return i,

  tf.while_loop(
      cond=loop_cond,
      body=loop_body,
      loop_vars=(0,))

  return l
```

This code works in eager execution, which does not use the TensorFlow runtime
for the `tf.while_loop`:

```
fn()
```

However, it does not work in graph execution, because TensorFlow uses special
mechanisms to ensure the computations are correctly sequenced in the dataflow
graph:

```
tf.function(fn)()  # Error -- illegal tensor capture!
```

The equivalent AutoGraph code raises the same error:

```
l = []
for i in tf.range(10):
  l.append(i)  # Error -- illegal tensor capture!
```

Instead, use the specialized `tf.TensorArray` class:

```
l = tf.TensorArray(tf.int32, size=0, dynamic_size=True)
for i in tf.range(10):
  l = l.write(l.size(), i)  # Okay
```

#### Python collections of fixed structure are allowed TensorFlow control flow

An exception from the previous rule is made by Python collections that are
static, that is, they don't grow in size for the duration of the computation.

Caution: Use functional style when manipulating static collections.

Examples:

```
static_list = [tf.constant(3)]
while d.prop > 0:
  static_list[0] -= 1  # Okay -- static_list does not change structure
```
```
static_object = MyClass()
static_object.field = tf.constant(3)
while static_object.field > 0:
  static_object.field -= 1  # Okay -- static_object does not change structure
```
```
static_dict = {'field': tf.constant(3)}
while static_dict['field'] > 0:
  static_dict['field'] -= 1  # Okay -- static_dict does not change structure
```

However, remember to use functional style when these collections are used
inside control flow.

#### Python collections of fixed structure with dynamic index

A more subtle error occurs when the collection is static, but is accessed in a
dynamic way, that is with a key that is not constant.

For example:

```
d = {'a': tf.constant(3)}
for i in tf.range(10):
  for key in d:
    d[key] += i  # Problem -- accessing `dict` using non-constant key
```

The code above will raises an "illegal capture" error. To remedy it, write it
in functional style:

```
d = {'a': tf.constant(3)}
for i in tf.range(10):
  d = {key: value + i for key, value in d.items()}  # Okay
```

### Access to source code

Key point: AutoGraph can only handle functions whose source code can be
accessed at runtime.

Almost all Python functions allow access to their source code. However, a few
exceptions exist:

 * functions created in the Python interactive shell
 * functions with native bindings (these do not have Python source code)
 * functions created dynamically, using `exec` or `eval`

Use
[inspect.getsource](https://docs.python.org/3/library/inspect.html#inspect.getsource)
to quickly diagnose whether the source code is available for a function.
