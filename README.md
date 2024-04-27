```
    Y
  .-^-.
 /     \      .- ~ ~ -.    pssssss...
()     ()    /   _ _   `.                     _ _ _
 \_   _/    /  /     \   \                . ~  _ _  ~ .
   | |     /  /       \   \             .' .~       ~-. `.
   | |    /  /         )   )           /  /             `.`.
   \ \_ _/  /         /   /           /  /                `'
    \_ _ _.'         /   /           (  (
                    /   /             \  \
                   /   /               \  \    psssss.....
                  /   /                 )  )
                 (   (                 /  /
                  `.  `.             .'  /
                    `.   ~ - - - - ~   .'
                       ~ . _ _ _ _ . ~
```

the cpython interpreter is already pretty good at io-bound tasks through async/await.

but the GIL (global interpreter lock) hinders thread-parallelism in cpu-bound tasks.

you currently can only achieve parallelism in python through multiprocessing, which is not ideal for many data intensive use-cases.

the community is actively working on this by either trying to introduce multiple sub-interpreters [^subint1] [^subint2] or making the GIL optional [^nogil1] [^nogil2] [^nogil3].

until then, we can use some workarounds:

- superset programming languages [^superset1] [^superset2]
	- still in their infancy.
- different python implementations, like jit interpreters
	- usually ~4x faster when single-threaded [^PyPy] [^numba] but not parallel and very constrained.
- the `multiprocessing` standard library
	- high call overhead, (de)serialization overhead, resource overhead.
- mixing c/c++ code with python
	- a) extending cpython: fastest, but hard to implement. 🔥
	- b) ctypes: signifiantly slower, but a lot easier to implement.

<br><br>

## 1) multiprocessing

when using the `multiprocessing` library in python, we can call multiple system processes that each come with their own seperate python interpreter, GIL and memory space.

this is very simple, straightforward and the intended way to write parallel code in the latest python version. but it comes with all the pros and cons of using a processes for parallel programming:

- ✓ simple.
- ✓ higher isolation, security, robustness.
- 〜 context switching: actually doesn't matter, since the `threading` library threads are kernel-level as well.
- 𝙓 resource overhead: memory allocation, creation and management are slower for processes. additionally, having a unique copy of the interpreter for each process is really wasteful.
- 𝙓 serialization overhead: there is no shared memory, so data has to be serialized and deserialized for inter-process communication. also some objects are unserializeable: the `pickle` module is used to serialize objects. but some objects are not pickleable (i.e. lambdas, file handles, etc.).

links:

- https://docs.python.org/3/library/multiprocessing.html (same api as `threading`, so they're interchangable)
- https://docs.python.org/3/library/concurrent.futures.html (same functionality but more java-like)

<br><br>

## 2) extending cpython (c extension modules) 🔥

works by extending cpython with modules in which the gil is manually released. we can then call those modules in multithreaded python code.

the rust extension libraries are promising and used in some new popular projects [^rust1] [^rust2] but contain unsafe code [^rustunsafe] and are generally still too immature.

alternatively you can also use cython (not to be confused with cpython) for code generation. it's heavily optimized and used by big libraries like `numpy` and `lxml`.

- ✓ max performance: fastest possible interop because we're calling the external c functions from the cpython interpreter, written in c. we can easily share large chunks of memory with `mmap()`.
- 𝙓 very complex api: data isn't marshalled automatically, gil isn't freed automatically.
- 𝙓 not portable: the build step is very difficult but fortunately there are nice build tools to simplify this [^setuptools].

links:

- https://docs.python.org/3/extending/

<br><br>

## 3) ctypes (foreign function interface)

writing a shared library in c (or any other language providing a c interface [^nogolang]) and then calling it from multithreaded python code.

ctypes aren't meant to be used for high performance libraries that you use frequently but codebase-glue. you can still use them for that purpose and gain a significant amount of performance, but you have to move as much of the computation as possible into the c implementation. additionally you should: share as little data, pass as little data and call the foreign function as little as possible.

- ✓ very simple: no knowledge of extension api necessary. gil is released automatically on each foreign function call [^release].
- ✓ portable: also works with other python interpreters.
- 𝙓 massive serialization overhead: automatic type conversions done by the ffi-library are very expensive [^ctypebad]. this can be partially circumvented by passing pointers or using cffi but it will still be significantly slower than extending cpython [^edge].

links:

- https://docs.python.org/3/library/ctypes.html
- https://cffi.readthedocs.io/en/stable/overview.html#main-mode-of-usage

<br><br>

## reference

- inspiration: https://snarky.ca/programming-language-selection-is-a-form-of-premature-optimization/

- https://docs.python.org/3/contents.html

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
[^edge]: https://github.com/mattip/c_from_python/blob/master/c_from_python.ipynb
[^numba]: https://numba.pydata.org/
