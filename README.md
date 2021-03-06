# Tentative `py.test` Tutorial






Testing for Exceptions
----------------------
You sometimes want to test not that your code suceeds, but that it fails
gracefully, or at least as expected.  You could write a more or less
cumbersome test using `try` that did something like:

```python
def test_verbose_raises():
    try:
        raise ValueError('I am a graceful failure')
    except ValueError:
        assert True, 'what a graceful failure!'
    except:
        assert False, 'this is not the graceful failure you were looking for'
    else:
        assert False, 'success is just another form of failure'
```

That is a lot of writing for a simple test, so pytest provides a context
manager, that encapsulates similar logic:

```python
import pytest

def test_succint_raises():
    with pytest.raises(ValueError):
        raise ValueError('I am a graceful failure')
```

If you care not only about the exception type, but want to also verify
e.g. the error message, you can get access to the exception object from
the context manager:

```python
def test_succint_raises_graceful():
    with pytest.raises(ValueError) as exception_info:
        raise ValueError('I am a graceful failure')
    assert 'graceful' in str(exception_info.value), 'the failure was graceful'
```

The relevant attributes of the exception object returned by the context
manager are `type`, `value` and `traceback`.

These examples are collected in file `test_1.py` in folder `test_raises`,
here is a sample run of those examples:

```
py.test -v test_raises/
============================= test session starts ==============================
platform darwin -- Python 2.7.10, pytest-2.8.0, py-1.4.30, pluggy-0.3.1 -- /Users/jaimefrio/miniconda/envs/numpydev/bin/python
cachedir: test_raises/.cache
rootdir: /Users/jaimefrio/open_source/pytest_sampler/test_raises, inifile:
collected 3 items

test_raises/test_1.py::test_verbose_raises PASSED
test_raises/test_1.py::test_succint_raises PASSED
test_raises/test_1.py::test_succint_raises_graceful PASSED

=========================== 3 passed in 0.01 seconds ===========================
```


Function Decorators
-------------------

In Python, a function decorator is a function that takes a function as
input, and returns another function.  They are typically used to expand
the functionality of other functions.  For instance:

```python
import time

def timer(func):
    def timed_func(*args, **kwargs):
        t = time.time()
        ret = func(*args, **kwargs)
        print('Function {} took {:.2e}s.'.format(func.__name__,
                                                 time.time() - t))
        return ret
    return timed_func

@timer
def count_to_n(n):
    for j in range(n):
        pass
```

The *decorated* `count_to_n` is equivalent to writing:

```python
def count_to_n(n):
    for j in range(n):
        pass

count_to_n = timer(count_to_n)
```

Either way, we can now run things like:

```
>>> count_to_n(1000)
Function count_to_n took 5.51e-05s.
>>> count_to_n(1000000)
Function count_to_n took 3.30e-02s.
```

Decorators are used in many places by py.test, to alter the behavior of
test functions in several ways, as we will soon see.


Skipping Tests
--------------
`pytest.mark.skipif` is a function decorator that can be used to mark
tests to be skipped:

```python
import pytest

@pytest.mark.skipif
def test_i_am_unconditionally_skipped_without_reason():
    print('Nothing to see here, we are being skipped')
```

Aside from a function to decorate, decorators can take extra parameters,
in the case of `skipif`, a boolean expression indicating whether the
test should be skipped or not, as well as a an optional `reason` keyword
argument taking a string that explains, well the reason for skipping the
test.  Both arguments are optional, with the default behavior being to
skip the test:

```python
@pytest.mark.skipif(True, reason='testing skip works')
def test_i_am_being_skipped():
    print('Nothing to see here, I am being skipped')

@pytest.mark.skipif(False, reason='testing not skipping works')
def test_i_am_not_being_skipped():
    print('Look at me, I am not being skipped')

@pytest.mark.skipif(reason='testing skip by default')
def test_yet_another_skipped_test():
    print('Nothing to see here, I am being skipped')
```

There is a chance you will not know if a test has to be skipped until
you have started running the tests, e.g. perhaps you need to check the
FW version of a device you only connect to during the test.  You can
also skip tests from within, a.k.a. imperative skipping, by calling the
function `pytest.skip`:

```python
def test_skipping_halfway_through():
    print('This code will be run...')
    pytest.skip('trying imperative skipping out')
    print('...but we will never make it to here.')
```

All of these examples are in the `test_skipif` folder, inside file
'test_1.py'.  Running `py.test` on that folder:

```
py.test -v test_skipif
============================= test session starts ==============================
platform darwin -- Python 2.7.10, pytest-2.8.0, py-1.4.30, pluggy-0.3.1 -- /Users/jaimefrio/miniconda/envs/numpydev/bin/python
cachedir: test_skipif/.cache
rootdir: /Users/jaimefrio/open_source/pytest_sampler/test_skipif, inifile:
collected 5 items

test_skipif/test_1.py::test_i_am_unconditionally_skipped_without_reason SKIPPED
test_skipif/test_1.py::test_i_am_being_skipped SKIPPED
test_skipif/test_1.py::test_i_am_not_being_skipped PASSED
test_skipif/test_1.py::test_yet_another_skipped_test SKIPPED
test_skipif/test_1.py::test_skipping_halfway_through SKIPPED

===================== 1 passed, 4 skipped in 0.01 seconds ======================
```


Marking Tests as Expected Failures
----------------------------------
Similarly to `pytest.mark.skipif` there also is `pytest.mark.xfail` to
identify tests know to be failing.  This is typically used to keep tests
for known bugs around without messing up reporting until you get around
to fix them.  Since we fix bugs before writing new functionality, this
should not be the most used of features.  Notice that, by default,
`xfail` marked tests still get run, but the error is not reported as a
test failure.

`xfail` can also be parametrized with:
 * a boolean expression, e.g. to only mark a test as expected failure on
   Windows but not Linux,
 * a `reason` keyword argument, just like skipif,
 * a `run` keyword argument, which if `False` will prevent the test from
   being run altogether, and
 * a `raises` keyword argument, which takes an exception type, and will
   only mark the test as expected failure if it raises that particular
   exception.

```python
import pytest

@pytest.mark.xfail
def test_i_will_fail():
    assert False, 'I told you I was going to fail'

@pytest.mark.xfail(run=False)
def test_i_will_break_things_big_time():
    print('Use this to not run e.g. tests that segfault')

@pytest.mark.xfail(raises=IndexError)
def test_moronic_list_access():
    a = []
    assert a[1] == 5

@pytest.mark.xfail(raises=IndexError)
def test_moronic_dict_access():
    a = {}
    # dict access raises KeyError, not IndexError
    assert a[1] == 5
```

The above example is in file `test_1.py` inside the `test_xfail` folder,
running those tests we get the following output:

```
py.test -v test_xfail
============================= test session starts ==============================
platform darwin -- Python 2.7.10, pytest-2.8.0, py-1.4.30, pluggy-0.3.1 -- /Users/jaimefrio/miniconda/envs/numpydev/bin/python
cachedir: test_xfail/.cache
rootdir: /Users/jaimefrio/open_source/pytest_sampler/test_xfail, inifile:
collected 4 items

test_xfail/test_1.py::test_i_will_fail xfail
test_xfail/test_1.py::test_i_will_break_things_big_time xfail
test_xfail/test_1.py::test_moronic_list_access xfail
test_xfail/test_1.py::test_moronic_dict_access FAILED

=================================== FAILURES ===================================
___________________________ test_moronic_dict_access ___________________________

    @pytest.mark.xfail(raises=IndexError)
    def test_moronic_dict_access():
        a = {}
        # dict access raises KeyError, not IndexError
>       assert a[1] == 5
E       KeyError: 1

test_xfail/test_1.py:20: KeyError
===================== 1 failed, 3 xfailed in 0.13 seconds ======================
```


Test Fixtures
-------------
Test fixtures create the proper state for tests to run under, e.g. by
establishing a connection to a database, or to a hardware device, and
performing any necessary setup.  Fixtures are implemented as functions
decorated with `pytest.fixture`.  The name of the function plays an
important role on how this work, so choose a unique, memorable one!

```python
import pytest

@pytext.fixture
def i_set_things_up():
    projector = {'status': 'doing fine',
                 'flashing': "dicts can't flash!"}
    return projector
```

To use this fixture within a test, we have to pass it as a parameter to
the test *with the exact same name as the fixture function*.

```
def test_fixture_contents(i_set_things_up):
    assert i_set_things_up['status'] == 'doing fine'
```

By default, the fixture gets run before every test function, so that
each test receives a fresh instance of the fixture, untainted by other
tests wrongdoings.

```python
def test_try_to_break_the_fixture_1(i_set_things_up):
    del i_set_things_up['flashing']

def test_try_to_break_the_fixture_2(i_set_things_up):
    assert i_set_things_up['flashing'] == "dicts can't flash!"
```

While the default fixture lifetime is a single test function call, they
can also be defined to be shared within a module (i.e. a *.py file), a
class (a pattern we are not using), or even a session (i.e. over all
*.py files discovered by py.test).  You can define it by passing a
`scope` keyword argument to the fixture decorator.

```python
@pytest.fixture(scope='module')
def i_also_set_things_up():
    projector = {'status': 'doing fine',
                 'flashing': "dicts can't flash!"}
    return projector
```

But you now have to be careful that your tests do not affect the state
of the shared fixture!  The second of this tests will fail as a result
of having run the first one:

```python
def test_try_to_break_the_module_fixture_1(i_also_set_things_up):
    del i_also_set_things_up['flashing']

def test_try_to_break_the_module_fixture_2(i_also_set_things_up):
    assert i_also_set_things_up['flashing'] == "dicts can't flash!"
```

All these examples are in the `test_1.py` file under the `test_fixtures`
folder.  Running them produces this output:

```
py.test -v test_fixtures/
============================= test session starts ==============================
platform darwin -- Python 2.7.10, pytest-2.8.0, py-1.4.30, pluggy-0.3.1 -- /Users/jaimefrio/miniconda/envs/numpydev/bin/python
cachedir: test_fixtures/.cache
rootdir: /Users/jaimefrio/open_source/pytest_sampler/test_fixtures, inifile:
collected 5 items

test_fixtures/test_1.py::test_fixture_contents PASSED
test_fixtures/test_1.py::test_try_to_break_the_fixture_1 PASSED
test_fixtures/test_1.py::test_try_to_break_the_fixture_2 PASSED
test_fixtures/test_1.py::test_try_to_break_the_module_fixture_1 PASSED
test_fixtures/test_1.py::test_try_to_break_the_module_fixture_2 FAILED

=================================== FAILURES ===================================
____________________ test_try_to_break_the_module_fixture_2 ____________________

i_also_set_things_up = {'status': 'doing fine'}

    def test_try_to_break_the_module_fixture_2(i_also_set_things_up):
>       assert i_also_set_things_up['flashing'] == "dicts can't flash!"
E       KeyError: 'flashing'

test_fixtures/test_1.py:28: KeyError
====================== 1 failed, 4 passed in 0.01 seconds ======================
```


Test Fixture Finalizers
-----------------------
Often times a test fixture will need some cleaning up after the test (or
tests, depending on the scope) completes.  You can do this by adding a
`request` parameter to the fixture, and calling the passed in object's
`addfinalizer` method.

```python
import pytest

@pytest.fixture
def i_set_things_up(request):
    projector = {'status': 'doing fine',
                 'flashing': "dicts can't flash!"}
    def fin():
        projector['status'] = 'torn down by finalizer!'
    request.addfinalizer(fin)
    return projector

def test_nothing(i_set_things_up):
    assert i_set_things_up['status'] == 'doing fine'
```
