# Overview

This is an [LZ4](http://lz4.org)-frame compression library for Python v3.2+ (and 2.7+), bound to Yann Collet's [LZ4 C implementation](https://github.com/lz4/lz4).


# Installing / packaging
```shell
# To get from PyPI
pip3 install py-lz4framed

# To only build extension modules inline (e.g. in repository)
python3 setup.py build_ext -i

# To build & install globally
python3 setup.py install

# To install locally with pip
pip install --upgrade --find-links=. .
```
**Notes**

- The above as well as all other python3-using commands should also run with v2.7+
- This module is also available via [Anaconda (conda-forge)](https://anaconda.org/conda-forge/py-lz4framed)
- PyPI releases are signed with the [Iotic Labs Software release signing key](https://iotic-labs.com/iotic-labs.com.asc)

# Improvements

This fork has several improvements I needed for my other project.

The scenario improvements address is downloading & decompressing large LZ4
data stream on the fly (hundreds of GBs). If the download stream is interrupted
the original decompressor had no way to resume the decompression where it stopped.

The main motivation is to recover from these interruptions.
Decompressor object now supports changing of the file-like object that is read from.
If input socket stream went down we can re-connect and continue from the 
position it stopped. More in test `test_decompressor_fp_continuation`.

If the processing logic is more complex you can use `clone_decompression_context`
to clone decompressor context (the whole decompression state) and revert 
to this checkpoint if something breaks. More in test `test_decompressor_fp_clone`.

In order to recover also from program crashes you can marshal / serialize
the decompressor context to the (byte) string which can be later
unmarshalled / deserialized and continue from that point. Marshalled state
can be stored e.g., to a file. More in test `test_decompressor_fp_marshalling`.

Please note marshalled state is not fully portable between systems with
different endianness & bit width (32 vs. 64). Most of the marshalling
is already universal, but checksum state is just memcpy-ied. So if you 
transfer the state to a different machine and checksum is wrong, this might be the case.

# Usage
Single-function operation:
```python
import lz4framed

compressed = lz4framed.compress(b'binary data')

uncompressed = lz4framed.decompress(compressed)
```
To iteratively compress (to a file or e.g. BytesIO instance):
```python
with open('myFile', 'wb') as f:
    # Context automatically finalises frame on completion, unless an exception occurs
    with Compressor(f) as c:
        try:
            while (...):
               c.update(moreData)
        except Lz4FramedNoDataError:
            pass
```
To decompress from a file-like object:
```python
with open('myFile', 'rb') as f:
    try:
        for chunk in Decompressor(f):
           decoded.append(chunk)
    except Lz4FramedNoDataError:
        # Compress frame data incomplete - error case
        ...
```
See also [lz4framed/\_\_main\_\_.py](lz4framed/__main__.py) for example usage.

# Documentation
```python
import lz4framed
print(lz4framed.__version__, lz4framed.LZ4_VERSION, lz4framed.LZ4F_VERSION)
help(lz4framed)
```

# Command-line utility
```shell
python3 -mlz4framed
USAGE: lz4framed (compress|decompress) (INFILE|-) [OUTFILE]

(De)compresses an lz4 frame. Input is read from INFILE unless set to '-', in
which case stdin is used. If OUTFILE is not specified, output goes to stdout.
```


# Tests

## Static
This library has been checked using [flake8](https://pypi.python.org/pypi/flake8) and [pylint](http://www.pylint.org), using a modified configuration - see _pylint.rc_ and _flake8.cfg_.

## Unit
```shell
python3 -m unittest discover -v .
```

# Why?
The only existing lz4-frame interoperable implementation I was aware of at the time of writing ([lz4tools](https://github.com/darkdragn/lz4tools)) had the following limitations:

- Incomplete implementation in terms of e.g. reference & memory leaks on failure
- Lack of unit tests
- Not thread safe
- Does not release GIL during low level (de)compression operations
- Did not address the requirements for an external project
