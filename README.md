# dpython

[![Passing Tests](https://github.com/CuckooEXE/dpython/actions/workflows/build.yml/badge.svg)](https://github.com/CuckooEXE/dpython/actions/workflows/build.yml)

This is (dumb) Python, with a number of (dumb) enhancements that will make your life ~more confusing~ easier. This is a fork of `Python 3.12.0 alpha 0` (which was the `main` branch at time of forking, no real reason for this version). 

> Please don't actually use this, this is just me messing around with Python to learn its internals, and to learn more about programming languages.

## Features

### Implicit `str` + `int` concatenation

A former professor/employer/landlord and good friend of mine was frustrated that Python does not have the ability to implicitly cast integer types to string types when printing them out. For example, in JavaScript we can do:

```javascript
> '5 + 5 = ' + (5 + 5)
< '5 + 5 = 10'
```

(and yes, I'm aware of that chart of various implicit casting that JavaScript does). However, Python is a little more strict giving us this:

```python
>>> '5 + 5 = ' + (5 + 5)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can only concatenate str (not "int") to str
```

Let's see if we can fix this. To start out, I should mention I've done this previously, but it's been years, and I no longer have the code; I'm fairly familiar with Python internals (more than the average developer using Python, much less than a Python developer). But to start, let's see if we can find the error in the Python source.

A quick search for "can only concatenate" results in a hit in `unicodeobject.c`:

```c
if (!PyUnicode_Check(right)) {
    PyErr_Format(PyExc_TypeError,
                  "can only concatenate str (not \"%.200s\") to str",
                  Py_TYPE(right)->tp_name);
    return NULL;
}
```

Trying our best to ignore Python's terrible style, we can see it checks to see if the RHS of the operation is a Unicode subclass, and if not, it'll throw this `TypeError`. The fact that this checks explicitly the RHS, and not the LHS made me try the following:

```python
>>> (5 + 5) + ' = 5 + 5'
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

So it looks like we'll have to handle two cases. More than likely we just go to the integer object's add method, and handle that case. For now, we'll focus on `str + int`, then `int + str`.

~~For the former case, I think I'll cast the RHS if it's not already a Unicode type, add it to the LHS Unicode, then destroy the cast (as to not leak memory).~~

So we don't have to cast after all, we can just get a new PyObject using `PyObject_Str` which basically calls `str(RHS)`, or `RHS.__str__()`. Then we use that as the new RHS. We'll still need to release it to ensure we don't leak memory. Remember, this code has to work, it doesn't have to be __that__ good :).

After building, we have 5 failed tests, but our feature works! It's more than likely that the tests __explicitly__ test that you can't concatenate `str + int` (or similar), so we'll have to change (or ignore :eyes: those tests).

```python
>>> '5 + 5 = ' + (5 + 5)
'5 + 5 = 10'
```

## TODO

 - [ ] IP Addresses as native types
 - [ ] Integer increment operator
 - [ ] Implicit `str` + `int` concatenation
 - [ ] Allow setting attributes of built-in/extension types

## Testing

```bash
./configure
make -j`nproc` test
```