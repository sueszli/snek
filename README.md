the cpython interpreter is already pretty good at io-bound tasks through async/await.

but the GIL (global interpreter lock) hinders thread-parallelism in cpu-bound tasks.

you currently can only achieve parallelism in python through multiprocessing, which is not ideal for many use-cases.

the community is actively working on this by either trying to introduce multiple sub-interpreters [^subint1] [^subint2] or making the GIL optional [^nogil1] [^nogil2] [^nogil3].

until then, we can:

- use superset programming languages [^superset1] [^superset2]
- use different python implementations, like jit interpreters [^PyPy]
- use the `multiprocessing` standard library
- use c-python interop

<br>

## multiprocessing

when using the `multiprocessing` library in python, we are calling multiple system processes that each come with their own seperate python interpreter, GIL and memeory space.

this is very simple and straightforward, and the intended way to write parallel code in the latest python version. but it comes with all the pros and cons of using a processes for parallel programming:

- ✓ simple.
- ✓ higher isolation, security, robustness.
- 〜 context switching: actually doesn't matter, since the `threading` library uses kernel-level threads.
- 𝙓 resource overhead: memory allocation, creation and management.
- 𝙓 serialization overhead: there is no shared memory, so data has to be serialized and deserialized for inter-process communication. also some objects are unserializeable: the `pickle` module is used to serialize objects. but some objects are not pickleable (i.e. lambdas, file handles, etc.).

links:

- https://docs.python.org/3/library/multiprocessing.html (same api as `threading`, so they're interchangable)
- https://docs.python.org/3/library/concurrent.futures.html (same functionality but more java-like)

<br>

## c extension modules

works by extending cpython with modules in which the gil is manually released. we can then call those modules in multithreaded python code.

the rust extension libraries are promising and used in some new popular projects [^rust1] [^rust2] but contain unsafe code [^rustunsafe] and are generally still too immature.

alternatively you can also use cython (not to be confused with cpython) for code generation. it's heavily optimized and used by `numpy` and `lxml` but a lot more complicated than writing the extension modules by hand in c. 

- ✓ staying within c environment by calling c functions inside the cpython interpreter.
- 𝙓 very complex api. data isn't marshalled automatically, gil isn't freed automatically.
- 𝙓 not portable. constrained to c. we must link cpython during the build step to extend it. but fortunately there are nice build tools to simplify this [^setuptools].

links:

- https://docs.python.org/3/extending/extending.html

<br>

## ctypes (foreign function interface)

writing a shared library in c (or any other language providing a c interface [^nogolang]) and then calling it from multithreaded python code.

- ✓ very simple. no knowledge of extension api necessary. gil is released automatically on each foreign function call [^release].
- ✓ portable. also works with other python interpreters.
- 𝙓 significantly higher python prep overhead, serialization costs and function call overhead. automatic type conversions done by the ffi-library are very expensive [^ctypebad]. → this can be partially circumvented by passing pointers.

links:

- https://docs.python.org/3/library/ctypes.html
- https://cffi.readthedocs.io/en/stable/overview.html#main-mode-of-usage

<br>

## references

- https://realpython.com/python-parallel-processing/#make-python-threads-run-in-parallel
- https://github.com/realpython/materials/tree/master/python-parallel-processing/
- https://github.com/mattip/c_from_python/blob/master/c_from_python.ipynb
- https://pythonspeed.com/articles/python-extension-performance/

[^subint1]: https://peps.python.org/pep-0554/
[^subint2]: https://peps.python.org/pep-0683/
[^nogil1]: https://peps.python.org/pep-0703/
[^nogil2]: https://discuss.python.org/t/a-steering-council-notice-about-pep-703-making-the-global-interpreter-lock-optional-in-cpython/30474
[^nogil3]: https://engineering.fb.com/2023/10/05/developer-tools/python-312-meta-new-features/
[^superset1]: https://www.taichi-lang.org/
[^superset2]: https://docs.modular.com/mojo/stdlib/python/python.html
[^rust1]: https://github.com/PyO3/pyo3/blob/main/guide/src/parallelism.md#parallelism
[^rust2]: https://github.com/pola-rs/polars
[^rustunsafe]: https://users.rust-lang.org/t/python-rust-interop/30243/12
[^release]: https://docs.python.org/3/library/ctypes.html#:~:text=released%20before%20calling
[^ctypebad]: https://stackoverflow.com/a/8069179/13045051
[^nogolang]: https://stackoverflow.com/questions/70349271/ctypes-calling-go-dll-with-arguments-c-string
[^setuptools]: https://setuptools.pypa.io/
[^PyPy]: https://www.pypy.org/index.html
