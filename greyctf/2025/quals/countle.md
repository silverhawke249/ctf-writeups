# Countle Training Centre

## The problem

Do 1,000,000 "operation" challenges consecutively and get the flag... which might be doable, but
upon checking `server.py`, we see that there is an upper limit of 0.1 second per challenge, which
means it would take at least 100,000 seconds to complete. That's more 27 hours! Which means we
should go about it a different way...

## The observations

One thing that stood out the most is the usage of `eval`, which is incredibly unsafe, especially
with user input. The server may blacklist certain words, but there are many ways to get
around that, such as reversing strings, using string concatenation, using escape sequences, and so
on. The server also applies a regex, but the regex `[0-9+\-*/()]+` only checks that the input begins
with any of the character in the character class. The biggest hurdles would be the fact that the
`FLAG` variable from the global scope is shadowed by a local definition within the context of
`eval`, and the removal of all built-in functions in `eval`'s context.

There are a couple approaches for these -- either try to get the interpreter to start a shell, or
start a debugging session on the interpreter. A shell can examine the contents of the Python script,
while a debugging session can step out of the `eval` context to view the value of `FLAG` in the
outer scope.

## The execution

My approach was to initiate a debugging session by calling `pdb.set_trace()` after importing `pdb`.

All built-ins may be removed, but we can still access a lot of things via literals -- in my case,
I used an empty tuple literal (`()`). The `tuple` class derives from the generic `object` class,
and magic methods/properties are still available. Therefore, we could access a list of all currently
available classes that derive from `object` by doing `().__class__.__base__.__subclasses__`.

From this point, we need to experiment a little bit to find the index that corresponds to the `_sitebuiltins._Helper` class. Then, we instantiate it and note that the function bound to calling
an instance of this class is exactly the `help` built-in function. This function has a dictionary
to other builtins, separate from the ones removed from the global scope, so we can invoke
`__import__` in order to import `pdb` programatically. This returns the `pdb` module itself, so we
can simply call `set_trace` from it.

Stringing everything together, we get the payload
`().__class__.__base__.__subclasses__()[158]().__call__.__buil𝘵ins__[_:=\"__impor\\x74__\"](\"pdb\").set_trace()`.
Note that the `t` in `__buil𝘵ins__` is a lookalike character -- Python silently converts it to a
regular ASCII `t` and this avoids tripping the blacklist from the program. In hindsight, the server
actually checks for `builtinscat`, and so just sending `__builtins__` should be allowed.

Sending this payload to the server gives us an interactive `(Pdb)` session! All we have to do is
send `u` to step out of the `eval` context, and execute `print(FLAG)` to obtain the flag.
