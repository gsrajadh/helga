# helga

[![Build Status](https://travis-ci.org/shaunduncan/helga.png)](https://travis-ci.org/shaunduncan/helga)


## About

An python-based IRC bot using Twisted. Original inspiration came from [thepeopleseason/olga](https://github.com/thepeopleseason/olga).
Why re-implement another bot? Because olga is written in perl, and I wanted something a bit more sane to look at.


## Requirements

All requirements for helga are listed in ``requirements.txt``. However, there is a single
external requirement, and that is MongoDB. You don't need it, per se, but many of the included
plugins use MongoDB for storing consistent state between restarts.


## Getting Started

Start by creating a virtualenv where helga will reside:

    $ virtualenv helga
    $ cd helga
    $ source bin/activate

Then grab the latest copy and install requirements:

    $ git clone https://github.com/shaunduncan/helga src/helga
    $ cd src/helga
    $ python setup.py develop
    $ pip install -r requirements.txt

Once you have performed the above steps, there will be a ``helga`` executable
placed in the ``bin`` dir of your virtualenv. Run helga by calling this:

    $ /path/to/venv/bin/helga

Note that this uses the default settings file, ``helga.settings`` to start. You can, and should, use
your own custom setttings. This file must a be an importable python file on $PYTHONPATH. To run helga
with your custom settings file, set an environment variable ``HELGA_SETTINGS`` to be a python import path:

    $ export HELGA_SETTINGS=path.to.mysettings

This will preserve any defaults in ``helga.settings``, but you can override at will.


## Plugins

### Overview

Helga supports plugins outside of the core source code. Plugins have a minimal API, but there
are some basic rules that should be followed. All core plugin implementations can be found
in ``helga.plugins.core``. The basic requirement for plugins is that they have a ``process``
attribute that is a callable and determines if the plugin should handle a message, and
a ``run`` method that actually performs the legwork of what the plugin should do. By convention,
the ``process`` method should accept four arguments:

- **client**: an instance of ``helga.comm.Client``
- **channel**: the channel on which the message was received
- **nick**: the current nick of the message sender
- **message**: the message string itself

The ``run`` is a bit different as it is up to the plugin implementation itself to decide what
arguments are necessary to generate a response. This method should be called by ``process`` and
should return one of:

- None or empty string, if no response is to be sent over IRC
- Non-empty string for a single line response
- List of strings for multiline responses

Below is a simple example:

```python
import time
from helga.plugins.core import Plugin

class MyPlugin(Plugin):
    def run(self, channel, nick, message):
        return 'Current timestamp: {0}'.format(time.time())

    def process(self, channel, nick, message):
        if message.startswith('!time'):
            return self.run(channel, nick, message)
```

### Plugin Types

For the most part, there are two main types of plugins: commands and matches. Commands are plugins
that require a user to specifically ask for helga to perform some action. For example,
``helga haiku`` or ``helga google something to search``. Matches are on the other hand are
intended to be autoresponders that give some extra meaning or context to what a user has said.
For example, if helga matches for a string "foo":

    <sduncan> i'm talking about foo in this message
    <helga> sduncan is talking about foo

For the sake of simplicity, there are two convenient decorators for authoring these types
of plugins (which is usually the case). For example:

```python
from helga.plugins import command, match

@command('foo', aliases=['foobar'], help="The foo command")
def foo(client, channel, nick, message, cmd, args):
    # This is run on "helga foo" or "helga foobar"
    return "Running the foo command"

@match(r'bar')
def bar(client, channel, nick, message, matches):
    # This will run whenever a user mentions the word 'bar'
    return "{0} said bar!".format(nick)
```

You may notice in the above example that each decorated function accepts different arguments.
For commands, there are two additional arguments ``cmd`` and ``args``. The former is the parsed
command that was used to run the method (which could be "foo" in the above case, or the alias
"foobar"). The latter is a list of whitespace delimited strings that follow the parsed commend.
For example ``helga foo a b c`` would mean the args param would be ``['a', 'b', 'c']``.

For the match plugin, the single additional argument is ``matches`` which is for the most part,
the result of ``re.findall``. However, the ``@match`` decorator accepts a callable in place of
a regex string. This callable should accept one argument: the message being processed. It should
return a value that can be evaluated for truthiness and will be passed to the decorated function
as the ``matches`` parameter.

### Preprocessors

Plugins can also be message preprocessors. These are callables that may perform some modification
on an incoming message prior to that message being delivered to any plugins. Preprocessors should
accept arguments (in order) for ``client``, ``channel``, ``nick``, and ``message`` and should
return a three-tuple consisting of (in order) ``channel``, ``nick``, and ``message``. To declare
a function as a preprocessor, a convenient decorator can be used:

```python
from helga.plugins import preprocessor

@preprocessor
def blank_message(client, channel, nick, message):
    return channel, nick, ''
```

### Complex plugins

Some plugins do both matching and act as a command. For this reason, plugin decorators are chainable.
However, remember that different plugin types expect decorated functions to accept different arguments.
It is best to accept ``*args`` for these:

```python
from helga.plugins import command, match, preprocessor

@preprocessor
@match(r'bar')
@command('foo')
def complex(client, channel, nick, message, *args):
    # len(args) == 0 for preprocessors
    # len(args) == 1 for matches
    # len(args) == 2 for commands
```

### Publishing plugins

Helga uses setuptools entry points for plugin loading. Once you've written a plugin you wish to use,
you will need to make sure your python package's setup.py contains an entry_point under the group
name ``helga_plugins``. For example:

    ```
    entry_points = {
        'helga_plugins': [
            'plugin_name = mylib.mymodule:MyPluginClass',
        ],
    },
    ```

Note that if you are using decorated function for a plugin, you will want to specify the method name
for your entry point, i.e. ``mylib.mymodule:myfn``.


## Tests

All tests are written to be run via ``tox``. To run the test suite, inside your virtualenv:

    $ cd src/helga
    $ tox

## Contributing

Contributions are welcomed, as well as any bug reports! Please note that any pull request will be denied
if tests run via tox do not pass

## License

Copyright (c) 2013 Shaun Duncan

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
