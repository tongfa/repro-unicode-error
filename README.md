# Reproduce unicode issue in VSCode.

I believe the underlying issue is unicode characters in test function names, i.e. the circles in:

```
def test_ᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤᐤ952():
```

When running a large set of unit tests each test name is provided to the pytest invocation on the client
machine as command line args. If there are enough tests to cross a 1024 byte boundary (or maybe its 3000),
then there is a chance that PipeManager will do a read that does not terminate on a unicode code point boundary.

For example, say the character ᐤ is encoded as two bytes, A and B, then it's possible by reading 1024 bytes that
the final character of the bunch is A, and that B will be available in a subsequent read. Decoding a string
under this condition will result in the following error:

```
Traceback (most recent call last):
  File "/home/ubuntu/.vscode-server/extensions/ms-python.python-2024.6.0/python_files/vscode_pytest/run_pytest_script.py", line 39, in <module>
    data = sock.read(3000)
  File "/home/ubuntu/.vscode-server/extensions/ms-python.python-2024.6.0/python_files/testing_tools/socket_manager.py", line 75, in read
    data = part.decode("utf-8")
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe1 in position 2999: unexpected end of data
Finished running tests!
```

There's an easy fix, upon the above exception read an extra byte and try again decoding utf again.

