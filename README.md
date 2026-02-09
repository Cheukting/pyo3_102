# PyO3 102 - Building a Python library with PyO3

## Preflight checklist

- Install/ Update Rust (Requires Rust 1.83 or greater.)
- Install [uv](https://docs.astral.sh/uv/)

## Part 1 - Starting a project using PyO3 

### What is PyO3?

PyO3 is a Rust library that brings Python interoperability to Rust and function as Rust bindings for Python, including tools for creating native Python extension modules.

It supports CPython, PyPy, and GraalPy.

### Setting up

1. Create a new working directory

```bash
mkdir pyo3_102
cd pyo3_102
```

2. Download Python 3.14

```bash
uv python install 3.14
```

3. Install **maturin** to uv

```bash
uv tool install maturin
```

4. Start a project

Maturin provides a command to initialize a new project.

```bash
maturin init
```

select `pyo3` when asked which binding to use.

5. Create a Python evniroment

```bash
uv venv
```

### What is maturin?

Formally know as `pyo3-pack`, maturin is a tool for building and publishing Python packages with Rust. Maturin automatically detects PyO3 bindings when it's added as a dependency in `Cargo.toml` and builds a rust extension module alongside the python package.

It simplifies the process of creating Python extensions using Rust by building and publishing crates with PyO3, cffi and uniffi bindings as well as rust binaries as python packages with minimal configuration. 

It supports building wheels for python 3.8+ on Windows, Linux, macOS and FreeBSD, can upload them to pypi and has basic PyPy and GraalPy support.

### Build a Python library

In `src/lib.rs` an example code creating a simple Python function in a Python module has been added. Before we dive into coding, let's first try building it and see what's inside a Python module.

To see what command `maturin` offer:

```bash
maturin --help
```

During development, the best command to use is `maturin develop` which the module will be installed locally in the current virtualenv:

```bash
maturin develop
```

### What's in a Python package?

After running the command, a Python wheel is built. You can inspect the `/target` folder.

Python packages come in two formats: A built form called wheel and source distributions (sdist), both of which are archives. A wheel can be compatible with any python version, interpreter, operating system and hardware architecture, or it can be limited to a specific platform and architecture or to a specific python interpreter and version on a specific architecture and operating system.

With PyO3, the wheel we built will be specific to a python interpreter and version on a specific architecture and operating system. When the user install a Python package, the tool (e.g. pip) will tries to find a matching wheel and install that. If it doesn't find one, it downloads the source distribution and builds a wheel for the current platform, which requires the right compilers to be installed. 

### Publishing Python packages

Installing a wheel is much faster than installing a source distribution as building wheels is generally slow. It is advised that the publisher of the package publishing multiple wheels that cover most common Python versions and operating system supported.

`maturin` provides a command `maturin generate-ci` which makes publishing for cross platform using GitHub Actions simple.

For more about publishing Python packages using `maturin`, see [the documentation](https://www.maturin.rs/distribution.html).

## Part 2 - Creating Python module and function with PyO3

Let's now look at the `src/lib.rs`, we see there are two macros, `pymodule` and `pyfunction`,from `pyo3::prelude` being used. 

That's it, it is super easy to create Python packages using PyO3 bindings. Let's now create our own, don't do anything complicated yet, may be a function to creat a sum of squares?

```rust
fn sum_of_squares(input: Vec<i32>) -> PyResult<i32> {
    Ok(input.iter().map(|&i| i * i).sum())
}
```

Just replace the original function that we had.

Now, let's give the Python module a name, we can do it using this macro after `#[pymodule]`:

```rust
#[pyo3(name = "pycool")]
```

I named it "pycool" but you can change it to anything.

### Mixing a Python and a Rust project

In most cases, you would like to write some Python code to acommpany or test the Python module you just created in Rust. In that case, it is advise to have a project structure like this:

```text
.
├── Cargo.toml
├── pyproject.toml
├── README.md
├── src
│   └── lib.rs
└── python
    └── pyo3_102
        ├── __init__.py
        └── test1.py
```

Next we need to do is to provide this extra information at `pyproject.toml` to specific where the Python source is:

```toml
[tool.maturin]
python-source = "python"
module-name = "pyo3_102.pycool"
```

Now if we run `maturin develop`, we will see a `pycool.cpython-314-darwin.so` created in the `python/pyo3_102/` folder.

To test, we will write the simple `text1.py`:

```python
from pycool import sum_of_squares

print(sum_of_squares([1,2,3]))
```

Then we can run and test it:

```bash
uv run python/pyo3_102/test1.py
```

### Throwing Python exceptions

You may notice, we are returning `PyResult` instead of `Result` in our previous examples. This bring us into looking at how errors are handled in Python. In Python, errors are handled using exceptions. When an error occurs during the execution of a program, an exception is raised, which can then be caught and handled using `try...except` blocks.

To solve the difference between Rust and Python, PyO3 uses the `PyResult<T>` type (which is an alias for `Result<T, PyErr>`) to indicate that a function can return either a successful value or a Python exception. When you return `Ok(value)`, PyO3 converts the value to its Python equivalent. When you return `Err(PyErr)`, PyO3 raises the corresponding exception in the Python interpreter.

For example, if our `sum_of_square` function is designed for calculating sum of areas and should returns an error when a negative number is encountered, since it doesn't make sense.

```rust
use pyo3::exceptions::PyValueError;
use pyo3::prelude::*;

#[pyfunction]
fn sum_of_squares(input: Vec<i32>) -> PyResult<i32> {
    Ok(input.iter()
        .map(|&i| {
            if i < 0 {
                Err(PyValueError::new_err("Input contains negative numbers"))
            } else {
                Ok(i * i)
            }
        })
        .collect::<PyResult<Vec<i32>>>()?
        .into_iter()
        .sum::<i32>())
}
```

If we now build again and run the `test1.py` test script with a negative number"

```python
print(sum_of_squares([-1,2,3]))
```
We will get a `ValureError`.

The list of all the exceptions that PyO3 provides can be found [here](https://docs.rs/pyo3/latest/pyo3/exceptions/index.html).

### Creating documentation and function signatures

Another nice feature provided by PyO3 is that it automatically convert the documentation from the Rust code to the Python module. Python provides a built-in `help` function that can be used to inspect the documentation of a module or a function.

If we create another test script `test2.py`:

```python
import pycool
help(pycool)
```

and run it, we see that our documentation for the module `A Python module implemented in Rust.` and that for the  function `Formats the sum of squares.` is there. 

We can also see the signature of the function is currently `sum_of_squares(input)`, which is not very informative.

In PyO3, we can use `#[pyo3(text_signature = "...")]` to specify the signature of the function. Let's add this to our function after `#[pyfunction]`:

```rust
#[pyo3(text_signature = "(input: list[int])")]
```

Now build and run `test2.py` again, we see that now overwrite the signature of the function in the documnetation with our own.

Other than changing how the function looks like in documentation, we can actually change the signature of the function in Python as well. To do that, we will use the `#[pyo3(signature = (...))]` option. This is handy if there's a default value to be added for any argument for the function. For example, if we add a default for our function:

```rust
#[pyo3(signature = (input = Vec::new()))]
```

Now build and run `test1.py` without any argument:

```python
print(sum_of_squares())
```

we will get a 0 instead of an error.

To see more examples of usage of `#[pyo3(signature = (...))]` and `#[pyo3(text_signature = "...")]`, see [the documentation](https://pyo3.rs/v0.28.0/function/signature.html).

### Exercise - Creating a D&D Python module

Task: In a seperate directory, create a new PyO3 project called `game_with_pyo3` and a module called `dnd`. The module should have a function called `roll_dice` that takes a number of dice and number of sides as arguments and returns a list of the rolled dice.

## Part 3 - Type Conversions between Rust and Python

### Using Rust types or Python-native types as arguments

When accepting a function argument, it is possible to either use Rust library types instead of PyO3’s Python-native types. For example, if we want to accept a list of integers as an argument, we can use `Vec<i32>` instead of `PyList`. The table of Rust types to their corrisponding Python types is available [here](https://pyo3.rs/v0.28.0/conversions/tables.html).

Using Rust library types as function arguments will incur a conversion cost compared to using the Python-native types. You can instead use Python-native types with almost zero-cost, however a type check would be required.

### Returning Rust values to Python

You have already seen that we can return `PyResult` for fallible functions. When returning a Rust value to Python, a lot of Rust types will be converted to their Python equivalent automatically. For the list, check [here](https://pyo3.rs/v0.28.0/conversions/tables.html#returning-rust-values-to-python). Another option is to return [PyO3’s smart pointers](https://pyo3.rs/v0.28.0/types#pyo3s-smart-pointers): `Py<T>`, `Bound<'py, T>`, or `Borrowed<'a, 'py, T>` with zero-cost.

### Creating Custom Python classes

In some cases, a more complex type than a primitive type is needed. For example, if we want to create a custom class that represents a point in the 2-D Cartesian coordinate, we can use PyO3’s `#[pyclass]` macro to define a Python class that wraps a Rust struct. The struct can have fields that are Rust types, and the class can have methods that operate on those fields. PyO3 will automatically generate the Python class and its methods, and will handle the conversion between Rust and Python types:

```rust
use pyo3::prelude::*;

#[pyclass]
struct Point {
    #[pyo3(get, set)]
    x: f64,
    #[pyo3(get, set)]
    y: f64,
}

#[pymethods]
impl Point {
    #[new]
    fn new(x: f64, y: f64) -> Self {
        Point { x, y }
    }

    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

Note that the `#[pymethods]` macro is used to define the methods of the class and `#[pyo3(get, set)]` is used to define the getter and setter methods for the fields. The `#[new]` macro is used to define the constructor method for the class.

Then, you can use it in Python script `test3.py` like this:

```python
from pycool import Point

p = Point(3.0, 4.0)
print(f"Point: ({p.x}, {p.y})")
print(f"Distance from origin: {p.distance_from_origin()}")

p.x = 5.0
print(f"New distance: {p.distance_from_origin()}")
```

Python objects created from `#[pyclass]` (`pyclass` types) are used in both Python and Rust code and this may cause issue with the `&mut` references borrow checking. To overcome this, PyO3 does borrow checking at runtime using a interior mutability scheme which is very similar to `std::cell::RefCell<T>`. To opt out of this borrow checking, a class can declaring itself frozen by adding the `#[pyclass(frozen)]` attribute.

For example, to borrow a `Point` object from Rust code:

```rust
use pyo3::prelude::*;

#[pyfunction]
fn move_point(p: PyRefMut<'_, Point>, dx: f64, dy: f64) {
    let mut p = p;
    p.x += dx;
    p.y += dy;
}
```

In the example above, `PyRefMut` is used to mutably borrow the `Point` object. If the object is already borrowed (either mutably or immutably) elsewhere, PyO3 will raise a `RuntimeError` in Python.

For more about creating custom Python classes, check [here](https://pyo3.rs/v0.28.0/class/).

### Exercise - Expanding your D&D Python module

Task: Create a new Python class in the `dnd` module called `Character` that has a `name`, `health` and `strength` field. You can also optionally add more fields to this class to represent other attributes of a character, such as `dexterity`, `constitution`, `intelligence`, `wisdom`, and `charisma`. The attribute of the character will be set upon creation. 

Additionally, implement methods for the class to handle character actions, such as `attack`, `defend`, and `heal`. For example, `attack` will take another `Character` as target and can be used to calculate the damage dealt by the character based on its strength, and the dexterity of the target.

## Part 4 - Considering the Python Global Interpreter Lock (GIL)

### What is a Global Interpreter Lock (GIL) in Python?

The Global Interpreter Lock (GIL) is a mechanism in Python that allows only one thread to execute Python bytecodes at a time. This means that even if you have multiple CPU cores, only one thread can execute Python code at any given time. The GIL is necessary to ensure that Python's memory management is thread-safe, but it can limit the performance of CPU-bound tasks that could benefit from parallel execution.

In recent versions of Python (3.14), a "free-threaded" build of CPython without the GIL has been introduced. Since version 0.23, PyO3 supports building Rust extensions for the free-threaded Python build and calling into free-threaded Python from Rust. Using PyO3 can provide Rust's "fearless concurrency", which can provide stronger safety guarantees than C extensions to "free-threaded" Python.

### How to use PyO3 with the free-threaded Python build?

Unless using `unsafe` code in Rust, PyO3 defaults to assuming Python modules created with it are thread-safe. However, if you are supporting unsafe code which was written with the historical assumption that Python was single-threaded due to the GIL, you can turn on the assumption of the GIL with `#[pymodule(gil_used = true)]` until all the `unsafe` code and inspected. By opting-out of supporting free-threaded Python, the Python interpreter will re-enable the GIL at runtime while importing your module and print a `RuntimeWarning` with a message containing the name of the module causing it to re-enable the GIL.

Let's see that in action:

```bash
uv python install 3.14t
uv python pin 3.14t
```

Now add the `gil_used = true` option to the `#[pymodule]` macro and build again. Run any test script and you will see a warning message printed.

### Attaching and detaching to the runtime

In the GIL-enabled build, CPython C API is only legal when an OS thread is explicitly attached to the interpreter runtime, that is, the GIL is acquired. In this case the, using the `Python<'py>` type the `'py` lifetime signify that the global interpreter lock is held. 

In the free-threaded build, there can be many threads simultaneously attached to the Python runtime. Holding a `'py` lifetime means only that the thread is currently attached to the Python interpreter – other threads can be simultaneously interacting with the interpreter. You still need to obtain a `'py` lifetime to interact with Python objects, you can register a thread using the `Python::attach` function unless you are using a thread that is created via the Python `threading` module, which PyO3 will handle setting up the `Python<'py>` token when CPython calls into your extension

When doing long-running tasks that do not require the CPython runtime or when doing any task that needs to re-attach to the runtime in free-threaded Python, you need to using `Python::detach` to avoid hangs and deadlocks.

### Making mutable PyClass threadsafe

In free-threaded Python, Python objects are freely shared between threads by the Python interpreter, that means `Send` and `Sync` are required for the `#[pyclass]` object. In some cases, the data structures stored within a `#[pyclass]` may themselves not be thread-safe and no `Send` and `Sync` on the `#[pyclass]` type. In this case, careful inspections and a manual `Send` and `Sync` implementation is required if the soundness of the implementation can be garenteed.

When using the Python `threading` module to simultaneously call a rust function that mutably borrows a `pyclass` in multiple threads, PyO3 will raise exceptions or panic to enforce exclusive access for mutable borrows. To avoid the possibility of having overlapping `&self` and `&mut self` references produce runtime errors, it is suggested to use `#[pyclass(frozen)]` and atomic data structures to control modifications directly, for example:

```rust
use std::sync::atomic::{AtomicU32, Ordering};

#[pyclass(frozen)]
struct FrozenCounter {
    count: AtomicU32,
}

#[pymethods]
impl FrozenCounter {
    #[new]
    fn new() -> Self {
        Self {
            count: AtomicU32::new(0),
        }
    }

    fn increment(&self) {
        self.count.fetch_add(1, Ordering::Relaxed);
    }

    fn get(&self) -> u32 {
        self.count.load(Ordering::Relaxed)
    }
}
```

or to use `locks` to make threads wait for access to shared data, like this:

```rust
use std::sync::Mutex;

#[pyclass]
struct LockCounter {
    count: Mutex<u32>,
}

#[pymethods]
impl LockCounter {
    #[new]
    fn new() -> Self {
        Self {
            count: Mutex::new(0),
        }
    }

    fn increment(&self) {
        let mut count = self.count.lock().unwrap();
        *count += 1;
    }

    fn get(&self) -> u32 {
        *self.count.lock().unwrap()
    }
}
```

Another new challenge is thread-safe single initialization. To initialize data exactly once, PyO3 provides `PyOnceLock` type, which is a close equivalent to `std::sync::OnceLock`. Using `PyOnceLock`can also help avoid deadlocks by detaching from the Python interpreter when threads are blocking while waiting for another thread to complete initialization. For example:

```rust
use pyo3::sync::PyOnceLock;

#[pyclass]
struct LazyConfig {
    data: PyOnceLock<String>,
}

#[pymethods]
impl LazyConfig {
    #[new]
    fn new() -> Self {
        Self {
            data: PyOnceLock::new(),
        }
    }

    fn get_data(&self, py: Python<'_>) -> PyResult<&str> {
        self.data.get_or_init(py, || {
            "Initialized Data".to_string()
        }).map(|s| s.as_str())
    }
}
```

### Exercise - Making your D&D Python module free-threaded compatible

Task: Check all the functions in the `dnd` module and make sure they are thread-safe. Using one or more of the strategies mentioned above to make the objects in the `Character` class are able to be shared between threads with no issues.

---

## Reference

This is the end of the workshop, there are much more in the usage of PyO3, however, we only have enough time to scratch the surface. Here is a list of resources that you can use to learn more about PyO3:

- [The PyO3 user guide](https://pyo3.rs/)
- [Crate pyo3_async_runtimes documentation](https://docs.rs/pyo3-async-runtimes/latest/pyo3_async_runtimes/index.html)
- [Python documentation - C API Extension Support for Free Threading](https://docs.python.org/3/howto/free-threading-extensions.html#freethreading-extensions-howto)

---

## Support this workshop

This workshop is created by Cheuk and is open source for everyone to use (under MIT license). Please consider sponsoring Cheuk's work via [GitHub Sponsor](https://github.com/sponsors/Cheukting).
