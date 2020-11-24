# TensorAnnotations

TensorAnnotations is an experimental library enabling annotation of semantic
tensor shape information using type annotations - for example:
```python
def calculate_loss(frames: array4[Time, Batch, Height, Width]):
  ...
```

This annotation states that the dimensions of `frames` are time-like,
batch-like, etc. (while saying nothing about the actual _values_ - e.g. the
actual batch size).

Why? Two reasons:

*   Shape annotations can be checked _statically_. This can catch a range of
    bugs caused by e.g. wrong selection or reduction of axes before you run your
    code - even when the errors would not necessarily throw a runtime exception!
*   Interface documentation (also enabling shape autocompletion in IDEs).

To do this, the library provides three things:

*   A set of custom tensor types for TensorFlow and JAX, supporting the above
    kinds of annotations
*   A collection of common semantic labels (e.g. `Time`, `Batch`, etc.)
*   Type stubs for common library functions that preserve semantic shape
    information (e.g. `reduce_sum(Tensor[Time, Batch], axis=0) ->
    Tensor[Batch]`)

TensorAnnotations is being developed for JAX and TensorFlow.

## Example

Here is some code that takes advantage of static shape checking:

```python
import tensorflow as tf

from tensor_annotations import axes
import tensor_annotations.tensorflow as ttf

Batch, Time = axes.Batch, axes.Time

def sample_batch() -> ttf.Tensor2[Time, Batch]:
  return tf.zeros((3, 5))

def train_batch(batch: ttf.Tensor2[Batch, Time]):
  m: ttf.Tensor1[Batch] = tf.reduce_max(batch, axis=1)  # B
  # do something useful

def main():
  batch = sample_batch()
  batch = tf.transpose(batch)  # A
  train_batch(batch)
```

This code contains shape annotations in the signatures of `sample_batch` and
`train_batch`, and in the line marked with `# B`. It is otherwise the same code
you would have written in an unchecked program.

You can check these annotations for inconsistencies by running a static type
checker on your code (see 'General usage' below). For example, deleting the
`tf.transpose` statement in the line marked with `# A` will result in the
following error from pytype:

```
File "example.py", line 10: Function train_batch was called with the wrong arguments [wrong-arg-types]
         Expected: (batch: Tensor2[Batch, Time])
  Actually passed: (batch: Tensor2[Time, Batch])
```

Similarly, changing the axis argument in the line marked with `# B` to
`tf.reduce_max(batch, axis=0)` results in:

```
File "example.py", line 15: Type annotation for m does not match type of assignment [annotation-type-mismatch]
  Annotation: Tensor1[Batch]
  Assignment: Tensor1[Time]
```

(These messages were shortened for readability. The actual errors will be more
verbose because fully qualified type names will be displayed. We are looking
into improving this.)

See `examples/tf_time_batch.py` for a complete example.

## Installation



To install TensorAnnotations itself:

```bash
pip install git+https://github.com/deepmind/tensor_annotations
```

You'll also need to take a few extra steps in order to let pytype (currently
the only type checker we support) to take advantage of these stubs. First, make a
copy of typeshed in e.g. your home directory:

```bash
git clone https://github.com/python/typeshed
```

Next, symlink `tensor_annotations` into your copy of typeshed:

```bash
# Path to installed tensor_annotations/__init__.pyi
d=$(python -c 'import tensor_annotations; print(tensor_annotations.__file__)')
# Path to tensor_annotations/
d=$(dirname "$d")

ln -s "$d/library_stubs/third_party/py/tensorflow" typeshed/third_party/3/tensorflow
ln -s "$d/library_stubs/third_party/py/jax" typeshed/third_party/3/jax
mkdir typeshed/third_party/3/tensor_annotations
ln -s "$d/__init__.py" typeshed/third_party/3/tensor_annotations/__init__.pyi
ln -s "$d/jax.pyi" typeshed/third_party/3/tensor_annotations/jax.pyi
ln -s "$d/tensorflow.pyi" typeshed/third_party/3/tensor_annotations/tensorflow.pyi
ln -s "$d/axes.py" typeshed/third_party/3/tensor_annotations/axes.pyi
```

## General usage

First, import `tensor_annotations` and start annotating function signatures
and variable assignments. This can be done gradually.

Next, run a static type checker on your code. At the moment we only support
pytype, but we're working on support for mypy and Pyre. However, even pytype
must be invoked in a special way, in order to let it know about the custom
typeshed installation as described above:

```
TYPESHED_HOME=$HOME/typeshed pytype your_code.py
```



We recommend you deliberately introduce a shape error and then confirm that
your type checker gives you an error to be sure you're set up correctly.

### Annotated tensor classes

TensorAnnotations provides tensor classes for JAX and TensorFlow:

```python
# JAX
import tensor_annotations.jax as tjax
tjax.arrayN  # Where N is the rank of the tensor

# TensorFlow
import tensor_annotations.tensorflow as ttf
ttf.TensorN  # Where N is the rank of the tensor
```

These classes can be parameterized by semantic axis labels (below) using
generics, similar to `List[int]`. (Different classes are needed for each rank
because Python currently does not support variadic generics, but we're working
on it.)

### Axis labels

Axis labels are used to indicate the semantic meaning of each dimension in a
tensor - whether the dimension is batch-like, features-like, etc. Note that no
connection is made between the symbol, e.g. `Batch`, and the actual _value_ of
that dimension (e.g. the batch size) - the symbol really does only describe the
semantic meaning of the dimension.

See `axes.py` for the list of axis labels we provide out of the box. To define a
custom axis label, simply subclass `tensor_annotations.axes.Axis`. You can also
use `typing.NewType` to do this using a single line:

```python
CustomAxis = typing.NewType('CustomAxis', axes.Axis)
```

In the future we intend to support axis types that are tied to the actual size
of that axis. Currently, however, we don't have a good way of doing this. If you
nonetheless want to annotate certain dimensions with a literal size, e.g. for
documentation of interfaces which are hardcoded for specific sizes, we recommend
you just use a custom axis for this purpose. (Just to be clear, though: these
sizes will _not_ be checked - neither statically, nor at runtime!)

```python
L64 = typing.NewType('L64', axes.Axis)
```

### Stubs

By default, TensorFlow and JAX are not aware of our annotations. For example, if
you have a tensor `x: array2[Time, Batch]` and you call `jnp.sum(x, axis=0)`,
you won't get a `array1[Batch]`, you'll just get an `Any`. We therefore provide
a set of custom type annotations for TensorFlow and JAX packaged in 'stub'
(`.pyi`) files.

Our stubs currently cover the following parts of the API. All operations are
supported for rank 1, 2, 3 and 4 tensors, unless otherwise noted.

#### TensorFlow

**Unary operators**: `tf.abs`, `tf.acos`, `tf.acosh`, `tf.asin`, `tf.asinh`,
`tf.atanh`, `tf.cos`, `tf.cosh`, `tf.exp`, `tf.floor`, `tf.logical_not`,
`tf.negative`, `tf.round`, `tf.sigmoid`, `tf.sign`, `tf.sin`, `tf.sinh`,
`tf.sqrt`, `tf.square`, `tf.tan`, `tf.tanh`, `tf.math.erf`, `tf.math.erfc`,
`tf.math.erfinv`, `tf.math.expm1`, `tf.math.is_inite`, `tf.math.is_inf`,
`tf.math.is_nan`, `tf.math.lbeta`, `tf.math.lgamma`, `tf.math.log`,
`tf.math.log1p`, `tf.math.log_sigmoid`, `tf.math.ndtri`, `tf.math.reciprocal`,
`tf.math.reciprocal_no_nan`, `tf.math.rint`, `tf.math.rsqrt`,
`tf.math.softplus`,`tf.math.softsign`

**Tensor creation**: `tf.ones`, `tf.ones_like`, `tf.zeros`, `tf.zeros_like`

**Axis manipulation**: `tf.reduce_all`, `tf.reduce_any`, `tf.reduce_logsumexp`,
`tf.reduce_max`, `tf.reduce_mean`, `tf.reduce_min`, `tf.reduce_prod`,
`tf.reduce_sum`, `tf.transpose` (up to rank 3). Yet to be typed: multi-axis reductions
(we support `reduce(x, axis=0)`, but not `reduce(x, axis=(0, 1))`),
`keepdims=True`

**Tensor unary operators**: For tensor `x`: `abs(x)`, `-x`, `+x`

**Tensor binary operators**: For tensors `a` and `b`: `a + b`, `a / b`, `a //
b`, `a ** b`, `a < b`, `a > b`, `a <= b`, `a >= b`, `a * b`. Yet to be typed:
`a ? float`, `a ? int` for `Tensor0`, broadcasting where one axis is 1

#### JAX

**Unary operators**: `jnp.abs`, `jnp.acos`, `jnp.acosh`, `jnp.asin`,
`jnp.asinh`, `jnp.atan`, `jnp.atanh`, `jnp.cos`, `jnp.cosh`, `jnp.exp`,
`jnp.floor`, `jnp.logical_not`, `jnp.negative`, `jnp.round`, `jnp.sigmoid`,
`jnp.sign`, `jnp.sin`, `jnp.sinh`, `jnp.sqrt`, `jnp.square`, `jnp.tan`,
`jnp.tanh`, `jnp.sqrt`

**Tensor creation**: `jnp.ones`, `jnp.ones_like`, `jnp.zeros`, `jnp.zeros_like`

**Axis manipulation**: `jnp.sum`, `jnp.mean` `jnp.transpose`. Yet to be typed:
`keepdims=True`

**Tensor unary operators**: For tensor `x`, `abs(x)`, `-x`, `+x`

**Tensor binary operators**: For tensors `a` and `b`, `a + b`, `a / b`, `a //
b`, `a ** b`, `a < b`, `a > b`, `a <= b`, `a >= b`, `a * b`. Yet to be typed:
`a ? float`, `a ? int` for `Tensor0`, broadcasting where one axis is 1

### Casting

Some of your code might be already typed with existing library tensor types:

```python
def sample_batch() -> jnp.array:
 ...
```

If this is the case, and you don't want to change these types globally in your
code, you can cast to TensorAnnotations classes with `typing.cast`:

```python
from typing import cast

x = cast(tjax.array2[Batch, Time], x)
```

Note that this is only a hint to the type checker - at runtime, it's a no-op. An
alternative syntax emphasising this fact is:

```python
x: tjax.array2[Batch, Time] = x  # type: ignore
```

## Gotchas

**Use tuples for shape/axis specifications**

For type inference with TensorFlow and JAX API functions we often have to match
additional arguments. I.e., the rank of a `tf.zeros(...)` tensor depends on the
length of the shape argument. This only works with tuples, and not with lists:

```python
a = tf.zeros((10, 10))  # Correctly infers type Tensor2[Any, Any]

b: ttf.Tensor2[Time, Batch] = get_batch()
c = tf.transpose(b, perm=(0, 1))  # Tracks and infers the axes-types of b
```

while

```python
a = tf.zeros([10, 10])  # Returns Any

b: ttf.Tensor2[Time, Batch] = get_batch()
c = tf.transpose(b, perm=[0, 1]))  # Does not track permutations and returns Any
```

**Runtime vs static checks**

Note that we do not verify that the rank of a tensor at runtime matches the one
specified in the annotations. If you were in an evil mood, you could create an
untyped (Any) tensor, and statically type it as something completely wrong. This
is in line with the rest of the python type-checking approach, which does not
*enforce* consistency with the annotated types at runtime.

**Value consistency**. Not only do we not verify the rank, we don't verify
anything about the actual shape value either. The following will _not_ raise an
error:

```python
x: tjax.array1[Batch] = jnp.zeros((3,))
y: tjax.array1[Batch] = jnp.zeros((5,))
```

Note that _this is by design_! Shape symbols such as `Batch` are _not_
placeholders for actual values like 3 or 5. Symbols only refer to the _semantic
meaning_ of a dimension. In the above example, say, `x` might be a train batch,
and `y` might be a test batch, and therefore they have different sizes, even
though both of their dimensions are batch-like. This means that even
element-wise operations like `z = x + y` would in this case not raise a
type-check error.

## See also

This library is one approach of many to checking tensor shapes. We don't expect
it to be the final solution; we create it to explore one point in the space of
possibilities.

Other libraries for checking tensor shapes include:

* [tsanley](https://github.com/ofnote/tsanley), which uses string annotations
  combines with runtime verification
* [Shape Guard](https://github.com/Qwlouse/shapeguard), another runtime
  verification tool using concise helper methods
* [swift-tfp](https://github.com/google-research/swift-tfp), a static analyzer
  for tensor shapes in Swift
  
To learn more about tensor shape checking in general, see:

* Stephan Hoyer's [Ideas for array shape typing in Python] doc
* The
  [Typing for multi-dimensional arrays](https://github.com/python/typing/issues/513)
  GitHub issue in `python/typing`
* Our
  [Shape annotation feature scoping](https://docs.google.com/document/d/1t-j1MJ9M0f0KMAnM22J97tCHSfVoFjAy9k4Lexi75c4/edit)
  and our [Shape annotation syntax proposal](https://docs.google.com/document/d/1But-hjet8-djv519HEKvBN6Ik2lW3yu0ojZo6pG9osY/edit) doc (a synthesis of the most promising ideas
  from the full doc)
* The Python
  [typing-sig](https://mail.python.org/archives/list/typing-sig@python.org/)
  mailing list (in particular,
  [this thread](https://mail.python.org/archives/list/typing-sig@python.org/thread/IOBJGI5SJCUHJAUE4BOULGFBBEO5DCVG/)
  )
* [Notes and recordsings](https://docs.google.com/document/d/1oaG0V2ZE5BRDjd9N-Tr1N0IKGwZQcraIlZ0N8ayqVg8/edit)
  from the Tensor Typing Open Design Meetings

## Repository structure

The `tensor_annotations` package contains four types of things:

* **Custom tensor classes**. We provide our own versions of e.g. TensorFlow's
  `Tensor` class and JAX's `Array` class in order to support shape
  parameterisation. These are stored in **`tensorflow.py`** and **`jax.py`**.
  (Note that these are only used in the context of type annotations - they are
  never instantiated - hence no implementation being present.)
* **Type stubs for custom tensor classes**. We also need to provide type
  annotations specifying what the shape of, say, `x: Tensor[A, B] + y:
  Tensor[B]` is. These are **`tensorflow.pyi`** and **`jax.pyi`**.
  * These are generated from `templates/tensors.pyi` using
    `tools/render_tensor_template.py`.
* **Type stubs for library functions**. Finally, we need to specify what the
  shape of, say, `tf.reduce_sum(x: Tensor[A, B], axis=0)` is. This information
  is stored in type stubs in **`library_stubs`**. (The `third_party/py`
  directory structure is necessary to indicate to pytype exactly which packages
  these stubs are for.) Ideally, these will eventually live in the libraries
  themselves.
  * JAX stubs are auto-generated from `templates/jax.pyi` using
    `tools/render_jax_library_template.pyi`. Note that we currently specify the
    signature of the library members we don't generate automatically as `Any`.
    Ideally, we'd like to automatically generate complete type stubs and then
    tweak them to include shape information, but we haven't gotten around to
    this yet.
  * For TensorFlow stubs, we start from stubs generated by a Google-internal
    TensorFlow stub generator
    and then hand-edit those stubs to include shape stubs. The edits we've made
    are demarcated by `BEGIN/END tensor_annotations annotations for ...` blocks.
    Again, we'll make this more automated in the future.
* **Common axis types**. Finally, we also provide a canonical set of common axis
  labels such as 'time' and 'batch'. These are stored in **`axes.py`**.