# Overview

This is an [LZ4](http://lz4.org)-frame compression library for Python v3.2+ (and 2.7+), bound to Yann Collet's [LZ4 C implementation](https://github.com/lz4/lz4).


# Installing / packaging
```shell
# To get from PyPI
pip3 install py-lz4framed_ph4

# To only build extension modules inline (e.g. in repository)
python3 setup.py build_ext -i

# To build & install globally
python3 setup.py install

# To install locally with pip
pip install --upgrade --find-links=. .
```
**Notes**

- The above as well as all other python3-using commands should also run with v2.7+
- This fork is based on https://github.com/Iotic-Labs/py-lz4framed

# Improvements

This fork has several improvements I needed for my other project.

* Streamed decompression continuation (on reconnect)
* Streamed decompression state clone - checkpointing
* Streamed decompression marshalling - failure recovery, checkpointing

## More on improvements 

The scenario improvements address is downloading & decompressing large LZ4
data stream on the fly (hundreds of GBs). If the download stream is interrupted
the original decompressor had no way to resume the decompression where it stopped.

## Continuation

The main motivation is to recover from aforementioned interruptions.
Decompressor object now supports changing of the file-like object that is read from.
If input socket stream went down we can re-connect and continue from the 
position it stopped. More in test `test_decompressor_fp_continuation`.

## State cloning

If the processing logic is more complex you can use `clone_decompression_context`
to clone decompressor context (the whole decompression state) and revert 
to this checkpoint if something breaks. More in test `test_decompressor_fp_clone`.

## State marshalling

In order to recover also from program crashes you can marshal / serialize
the decompressor context to the (byte) string which can be later
unmarshalled / deserialized and continue from that point. Marshalled state
can be stored e.g., to a file. More in test `test_decompressor_fp_marshalling`.

## Random access archive

Situation: 800 GB LZ4 encrypted file. You want random access the file
 so it can be map/reduced or processed in parallel from different offsets.

Marshalled decompressor state takes only the required
amount of memory. If the state dump is performed on the block boundaries
(i.e., when the size hint from the previous call was provided by the input stream)
 the marhsalled size would be only 184 B, in the best case scenario, 66 kB in the worse case - 
 when LZ4 file is using linked mode.
  
Anyway, when state marshalling returns this small state the application
can build a meta file, the mapping: position in the input stream -> decompressor context.
With this meta file a new decompressor can jump to the particular checkpoint. 

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

# Further reading

* https://github.com/lz4/lz4/wiki/lz4_Frame_format.md
* https://github.com/lz4/lz4/wiki/lz4_Block_format.md
* https://ticki.github.io/blog/how-lz4-works/
* https://fastcompression.blogspot.cz/2011/05/lz4-explained.html
