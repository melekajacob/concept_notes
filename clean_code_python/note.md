# Python Clean Code

## 2. Pythonic Code

* Python provides its own mechanisms for accomplishing common tasks
* Idiom: Type of way to write code in order to perform a specific task
  * Common and repeats the same structure every time
  * Way things should be written when we want to perform a particular task
  * Language dependent
  * Writing idiomatic code in the language usually helps performance
  * Allows entire team get used to the same patterns in the code
* Indexes and slices
  * First index is at `0`
  * With negative numbers can access indices in the array (negative number start counting from end of array, so `arr[-1]` is last element in list)
```python
>>> my_numbers = (4, 5, 3, 9)
>>> my_numbers[-1]
9
>>> my_numbers[-3]
5
```
  * Can use `slice` to obtain a subarray of a sequence 
  * Format is `[start:end:step]` (end is not inclusive) and they refer to array indices
  * Tuples and lists can both be sliced 
```python
>>> my_numbers = (1, 1, 2, 3, 5, 8, 13, 21)
>>> my_numbers[2:5]
(2, 3, 5)
```

  * Can also skip between elements 
```python
>>> my_numbers[:3]
(1, 1, 2)
>>> my_numbers[3:]
(3, 5, 8, 13, 21)
>>> my_numbers[::]  # also my_numbers[:], returns a copy
(1, 1, 2, 3, 5, 8, 13, 21)
>>> my_numbers[1:7:2]
(1, 3, 8)
```
  * returns a shallow copy of the subarray 
* Creating your Own Sequences
  * `magic methods` are those surrounded by `__`
  *  Need to implement `__getitem__` to create your own sequence
  *  This is the method that is called when   `myobject[key]` is called (key is the param)
  *  A sequence is an object which implements `__getitem__` and `__len__`
  *  Lists, tuples, and strings are example of built-in sequence objects
  *  If your custom sequence wraps a built-in sequence, than delegate to 
  *  Can inheret from `Sequence` ABC
```python
from collections.abc import Sequence
class Items(Sequence):
    def __init__(self, *values):
        self._values = list(values)
    def __len__(self):
        return len(self._values)
    def __getitem__(self, item):
        return self.__getitem__(item)
```
  * Any time we want our own implementation of a built-in object such as sequences and mappings we should inherit from ABC
  * In the above implementation we use composition of a list rather than inheritence
  * If not wrapping any built-in types keep in mind:
    * When indexing by range, result should be instance of same type of class
      * i.e. When you slice a range, you actually get a range, you don't get an array for example
```python
>>> range(1, 100)[25:50]
range(26, 51)
```
  * Consider developing classes using magic methods to follow the already existing Python built-in classes, which will make them easier to use
    * But make sure the use case agrees, don't try to force it to be fancy
* Context Managers
  * We use when we want some precondition and postcondition code to be run before performing an action (code to run before and after a certain main action)
  * Often occur around reasource management
    * i.e. when we open a file and then want to close after, when we open a connection and then want to close after
  * We need to have setup and cleanup code, but what if an exception occurs, we must have a way to run the cleanup code
    * Could do it `finally` clause, but more Pythonic to use Context Managers
```python
Before:
fd = open(filename)
try:
    process_file(fd)
finally:
    fd.close()

After: 
# open returns a file object with implementations of __enter__ and __exit__
with open(filename) as fd:
    process_file(fd)
```
  * `with` statement enters the context manager (`open` function implements the `context manager protocol`)
    * The file will be automatically closed when block is finished (even if exception occurs)
    * Two magic methods used, `__enter__` and `__exit__`
      * Whatever is returned from `__enter__` placed in variable defined with `as` 
      * When error occurs or last line of code is reached, `__exit__` method is called of current context manager
      * `__exit__` is even passed an exception if it occurs and you can handle it accordingly
  * We often want to use context managers when dealing with some limited resource which requires acquisition and cleanup
    * Notice we are isolating setup/cleanup from business logic, which is what we really want (`SOC`)
    * BP: Always return something on `__enter__`
```python
def stop_database():
    run("systemctl stop postgresql.service")
def start_database():
    run("systemctl start postgresql.service")
class DBHandler:
    def __enter__(self):
        stop_database()
        return self
    def __exit__(self, exc_type, ex_value, ex_traceback):
        start_database()
def db_backup():
    run("pg_dump database")
def main():
    with DBHandler():
        db_backup()
```
  * Even if error occurs, `__exit__` will still be called 
    * If no exception occurred, all params of `__exit__` are `None`
    * If you return `True` from `__exit__` it acts as catching the exception, but this is generally not the desired effect
    * BP: When possible, dont return `True` in `__exit__`, try to catch exceptions earlier or let them stop program
  * Implementing our Own Context Managers
    * Just need dto implement `__enter__` and `__exit__` magic methods and your object can act as a context manager
    * Can also use `contextlib` for non context managing objects into context managing objects
      * Note: Contains other useful methods around context mangement
    * `contextlib.contextmanager` can make a `generator` be used as context manager
      * Everything before the yield will act as `__enter__`, the yielded value is returned (and generator suspended) (this returned value can be stored with `with generator_cm() as <var_name>`), and then the code within the `with` block is run, and on error/on completion, everything after `yield` is run which acts as `__exit__` code
```python
import contextlib
@contextlib.contextmanager
def db_handler():
    try:
        stop_database()
        yield
    finally:
       start_database()
with db_handler():
    db_backup()
```
  * This is especially useful when you want a context manager that isn't associated with any particular object 
    * BP: Use context management for locks and semaphores, which ensures you don't forget to release them
  * This is usually a good idea when our context management code has little to with the class we are trying to make into a context manager/when we don't need the return value
  * But what if you want to turn a regular function into something that is context managed
    * Can use `contextlib.ContextDecorator` to create your own decorator class, and then use it to decorate your methods (or in your class hierarchy to allow other class to be context managers)
```python
class dbhandler_decorator(contextlib.ContextDecorator):
    def __enter__(self):
        stop_database()
        return self
    def __exit__(self, ext_type, ex_value, ex_traceback):
        start_database()
@dbhandler_decorator()
def offline_backup():
    run("pg_dump database")
```
  * Now you could call `offline_backup()` without any `with` and it will be called inside a context manager
```python
offline_backup()

# Is the same as...
try:
  __enter__()
  offline_backup()
except:
  __exit__()
finally:
  __exit__()

```
  * Especially useful for reusing logic
* Exception suppression
  * If you want to make it very obvious that certain exceptions are okay (remember, code should be obvious with little surprises)
```python
import contextlib
with contextlib.suppress(DataConversionException):
    parse_data(input_json_or_dict)
```