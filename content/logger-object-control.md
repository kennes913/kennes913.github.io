+++
title = "dynamically control logger objects in python"
date = 2020-07-16
+++

The python logging library is more powerful than you think. As I've used the logging library more, I've become accustomed to the ways it can be finely controlled in source code. This post will explore 1 way.

Here's a directory structure:

```bash
    .
    ├── pkg
    │   ├── bar.py
    │   ├── baz.py
    │   ├── config.py
    │   └── foo.py
    └── set_loggers.py
```

The logging code is the same in the 3 modules: **foo.py**, **bar.py** and **baz.py**:

```python
import logging

import logging.config

from pkg import config

logging.config.dictConfig(config.configuration.get("logging"))
logger = logging.getLogger(__name__)

def run_statements():
    logger.info("This is an INFO statement.")
    logger.debug("This is a DEBUG statement.")

```

The central logging config is in **config.py**:

```python
# config.py
configuration = {
    "logging": {
        "version": 1,
        "formatters": {
            "lib": {
                "class": "logging.Formatter",
                "datefmt": "%Y-%m-%d %H:%M:%S",
                "format": "%(asctime)s.%(msecs)03d | %(levelname)s | %(name)s: %(message)s",
            },
        },
        "handlers": {
            "stdout": {
                "class": "logging.StreamHandler",
                "level": 0,
                "formatter": "lib",
                "stream": "ext://sys.stdout",
            },
            "null": {
                "class": "logging.NullHandler",
                "level": 0,
                "formatter": "lib"
            },
        },
        "loggers": {
            "pkg.foo": {"handlers": ["null"], "level": "INFO"},
            "pkg.bar": {"handlers": ["null"], "level": "INFO"},
            "pkg.baz": {"handlers": ["null"], "level": "INFO"}
        },
    }
}

```
This sets up how the logger formats logging messages, defines the type of handlers -- where the logging messages go (in this case to stdout) -- and the configured logger objects themselves.

One powerful thing you can do is activate and deactivate logger objects during runtime. Here's some example code utilizing the above logging configuration and loggers:
```python
# set_loggers.py
import logging
import sys
import typing

from pkg import foo, bar, baz, config

def stream_to_stdout(loggers: typing.List) -> None:
    """Activate logging for a module or a set of modules."""
    for logger in loggers:
        l = logging.root.manager.loggerDict.get(logger)
        h = logging.StreamHandler(sys.stdout)
        fmt = config.configuration.get("logging").get("formatters").get("lib")
        h.setFormatter(
            logging.Formatter(fmt=fmt.get("format"), datefmt=fmt.get("datefmt"))
        )
        l.handlers.append(h)


def deactivate_stream_log(loggers: typing.List) -> None:
    """Deactivate logging for a module or a set of modules."""
    for logger in loggers:
        l = logging.root.manager.loggerDict.get(logger)
        for i, handler in enumerate(t.handlers):
            if not isinstance(handler, logging.NullHandler):
                l.handlers.pop(i)

# Nothing should output here

for m in  (foo, bar, baz):
    m.run_statements()

# activate some logging
stream_to_stdout(["pkg.bar", "pkg.baz"])

# bar and baz modules will output
for m in  (foo, bar, baz):
    m.run_statements()

stream_to_stdout(["pkg.foo"])

# foo, bar and baz modules will output
for m in  (foo, bar, baz):
    m.run_statements()

deactivate_stream_log(["pkg.foo", "pkg.bar", "pkg.baz"])

# None will output
for m in  (foo, bar, baz):
    m.run_statements()
```

And the output:

```python
Python 3.7.5 (default, Nov 13 2019, 22:50:53)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.9.0 -- An enhanced Interactive Python. Type '?' for help.
2020-07-15 21:50:24.463 | INFO | pkg.bar: This is an INFO statement.
2020-07-15 21:50:24.464 | INFO | pkg.baz: This is an INFO statement.
2020-07-15 21:50:24.464 | INFO | pkg.foo: This is an INFO statement.
2020-07-15 21:50:24.464 | INFO | pkg.bar: This is an INFO statement.
2020-07-15 21:50:24.464 | INFO | pkg.baz: This is an INFO statement.
```

It's hard to tell from the output, but this source code does 2 important things:
- access the **logging.root.manager** object
- selectively activate and deactivate a logger object by adding/removing a handler

Now you have fine-grained logging control in your source code. All possible because of the existence of the **loggerDict** global object -- a dictionary containing the logging hierarchy of your source code:

```python
In [1]: logging.root.manager.loggerDict
Out[1]:
{..., # other loggers ommitted
 'pkg.foo': <Logger pkg.foo (INFO)>,
 'pkg': <logging.PlaceHolder at 0x10f20e110>,
 'pkg.bar': <Logger pkg.bar (INFO)>,
 'pkg.baz': <Logger pkg.baz (INFO)>}
```

Below you can see what happens when you add a handler to a logger object:

```python
In [4]: for k,v in logging.root.manager.loggerDict.items():
   ...:     if isinstance(v, logging.Logger):
   ...:         print(k, v.handlers)
...
pkg.foo [<NullHandler (NOTSET)>]
pkg.bar [<NullHandler (NOTSET)>]
pkg.baz [<NullHandler (NOTSET)>]
In [5]: stream_to_stdout(["pkg.bar", "pkg.baz"]) # add handlers
In [6]: for k,v in logging.root.manager.loggerDict.items():
   ...:     if isinstance(v, logging.Logger):
   ...:         print(k, v.handlers)
...
pkg.foo [<NullHandler (NOTSET)>]
pkg.bar [<NullHandler (NOTSET)>, <StreamHandler <stdout> (NOTSET)>]
pkg.baz [<NullHandler (NOTSET)>, <StreamHandler <stdout> (NOTSET)>]
```
When a message is logged via logger.info, the logger will chain the message through registered handlers. In this case, the **pkg.bar** and **pkg.baz** loggers send the message to a **NullHandler** object and then to the **StreamHandler** object eventually printing to stdout.
