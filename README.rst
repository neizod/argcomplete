argcomplete - Bash tab completion for argparse
==============================================
*Tab complete all the things!*

Argcomplete provides easy, extensible command line tab completion of arguments for your Python script.

It makes two assumptions:

* You're using bash or zsh as your shell
* You're using `argparse <http://docs.python.org/2.7/library/argparse.html>`_ to manage your command line arguments/options

Argcomplete is particularly useful if your program has lots of options or subparsers, and if your program can
dynamically suggest completions for your argument/option values (for example, if the user is browsing resources over
the network).

Installation
------------
::

    pip install argcomplete
    activate-global-python-argcomplete

See `Activating global completion`_ below for details about the second step (or if it reports an error).

Refresh your bash environment (start a new shell or ``source /etc/profile``).

Synopsis
--------
Python code (e.g. ``my-awesome-script.py``):

.. code-block:: python

    #!/usr/bin/env python
    # PYTHON_ARGCOMPLETE_OK
    import argcomplete, argparse
    parser = argparse.ArgumentParser()
    ...
    argcomplete.autocomplete(parser)
    args = parser.parse_args()
    ...

Shellcode (only necessary if global completion is not activated - see `Global completion`_ below), to be put in e.g. ``.bashrc``::

    eval "$(register-python-argcomplete my-awesome-script.py)"

argcomplete.autocomplete(*parser*)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This method is the entry point to the module. It must be called **after** ArgumentParser construction is complete, but
**before** the ``ArgumentParser.parse_args()`` method is called. The method looks for an environment variable that the
completion hook shellcode sets, and if it's there, collects completions, prints them to the output stream (fd 8 by
default), and exits. Otherwise, it returns to the caller immediately.

.. admonition:: Side effects

 Argcomplete gets completions by running your program. It intercepts the execution flow at the moment
 ``argcomplete.autocomplete()`` is called. After sending completions, it exits using ``exit_method`` (``os._exit``
 by default). This means if your program has any side effects that happen before ``argcomplete`` is called, those
 side effects will happen every time the user presses ``<TAB>`` (although anything your program prints to stdout or
 stderr will be suppressed). For this reason it's best to construct the argument parser and call
 ``argcomplete.autocomplete()`` as early as possible in your execution flow.

Specifying completers
---------------------
You can specify custom completion functions for your options and arguments. Two styles are supported: callable and
readline-style. Callable completers are simpler. They are called with the following keyword arguments:

* ``prefix``: The prefix text of the last word before the cursor on the command line. All returned completions should begin with this prefix.
* ``action``: The ``argparse.Action`` instance that this completer was called for.
* ``parser``: The ``argparse.ArgumentParser`` instance that the action was taken by.
* ``parsed_args``: The result of argument parsing so far (the ``argparse.Namespace`` args object normally returned by
  ``ArgumentParser.parse_args()``).

Completers should return their completions as a list of strings. An example completer for names of environment
variables might look like this:

.. code-block:: python

    def EnvironCompleter(prefix, **kwargs):
        return (v for v in os.environ if v.startswith(prefix))

To specify a completer for an argument or option, set the ``completer`` attribute of its associated action. An easy
way to do this at definition time is:

.. code-block:: python

    from argcomplete.completers import EnvironCompleter

    parser = argparse.ArgumentParser()
    parser.add_argument("--env-var1").completer = EnvironCompleter
    parser.add_argument("--env-var2").completer = EnvironCompleter
    argcomplete.autocomplete(parser)

If you specify the ``choices`` keyword for an argparse option or argument (and don't specify a completer), it will be
used for completions.

A completer that is initialized with a set of all possible choices of values for its action might look like this:

.. code-block:: python

    class ChoicesCompleter(object):
        def __init__(self, choices=[]):
            self.choices = [str(choice) for choice in choices]

        def __call__(self, prefix, **kwargs):
            return (c for c in self.choices if c.startswith(prefix))

The following two ways to specify a static set of choices are equivalent for completion purposes:

.. code-block:: python

    from argcomplete.completers import ChoicesCompleter

    parser.add_argument("--protocol", choices=('http', 'https', 'ssh', 'rsync', 'wss'))
    parser.add_argument("--proto").completer=ChoicesCompleter(('http', 'https', 'ssh', 'rsync', 'wss'))

Note that if you use the ``choices=<completions>`` option, argparse will show
all these choices in the ``--help`` output by default. To prevent this, set
``metavar`` (like ``parser.add_argument("--protocol", metavar="PROTOCOL",
choices=('http', 'https', 'ssh', 'rsync', 'wss'))``).

The following `script <https://raw.github.com/kislyuk/argcomplete/master/docs/examples/describe_github_user.py>`_ uses
``parsed_args`` and `Requests <http://python-requests.org/>`_ to query GitHub for publicly known members of an
organization and complete their names, then prints the member description:

.. code-block:: python

    #!/usr/bin/env python
    # PYTHON_ARGCOMPLETE_OK
    import argcomplete, argparse, requests, pprint

    def github_org_members(prefix, parsed_args, **kwargs):
        resource = "https://api.github.com/orgs/{org}/members".format(org=parsed_args.organization)
        return (member['login'] for member in requests.get(resource).json() if member['login'].startswith(prefix))

    parser = argparse.ArgumentParser()
    parser.add_argument("--organization", help="GitHub organization")
    parser.add_argument("--member", help="GitHub member").completer = github_org_members

    argcomplete.autocomplete(parser)
    args = parser.parse_args()

    pprint.pprint(requests.get("https://api.github.com/users/{m}".format(m=args.member)).json())

`Try it <https://raw.github.com/kislyuk/argcomplete/master/docs/examples/describe_github_user.py>`_ like this::

    ./describe_github_user.py --organization heroku --member <TAB>

If you have a useful completer to add to the `completer library
<https://github.com/kislyuk/argcomplete/blob/master/argcomplete/completers.py>`_, send a pull request!

Readline-style completers
~~~~~~~~~~~~~~~~~~~~~~~~~
The readline_ module defines a completer protocol in rlcompleter_. Readline-style completers are also supported by
argcomplete, so you can use the same completer object both in an interactive readline-powered shell and on the bash
command line. For example, you can use the readline-style completer provided by IPython_ to get introspective
completions like you would get in the IPython shell:

.. _readline: http://docs.python.org/2/library/readline.html
.. _rlcompleter: http://docs.python.org/2/library/rlcompleter.html#completer-objects
.. _IPython: http://ipython.org/

.. code-block:: python

    import IPython
    parser.add_argument("--python-name").completer = IPython.core.completer.Completer()

You can also use `argcomplete.CompletionFinder.rl_complete <https://argcomplete.readthedocs.org/en/latest/#argcomplete.CompletionFinder.rl_complete>`_
to plug your entire argparse parser as a readline completer.

Printing warnings in completers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Normal stdout/stderr output is suspended when argcomplete runs. Sometimes, though, when the user presses ``<TAB>``, it's
appropriate to print information about why completions generation failed. To do this, use ``warn``:

.. code-block:: python

    from argcomplete import warn

    def AwesomeWebServiceCompleter(prefix, **kwargs):
        if login_failed:
            warn("Please log in to Awesome Web Service to use autocompletion")
        return completions

Using a custom completion validator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
By default, argcomplete validates your completions by checking if they start with the prefix given to the completer. You
can override this validation check by supplying the ``validator`` keyword to ``argcomplete.autocomplete()``:

.. code-block:: python

    def my_validator(current_input, keyword_to_check_against):
        # Pass through ALL options even if they don't all start with 'current_input'
        return True

    argcomplete.autocomplete(parser, validator=my_validator)

Global completion
-----------------
In global completion mode, you don't have to register each argcomplete-capable executable separately. Instead, bash
will look for the string **PYTHON_ARGCOMPLETE_OK** in the first 1024 bytes of any executable that it's running
completion for, and if it's found, follow the rest of the argcomplete protocol as described above.

.. admonition:: Bash version compatibility

 Global completion requires bash support for ``complete -D``, which was introduced in bash 4.2. On OS X or older Linux
 systems, you will need to update bash to use this feature. Check the version of the running copy of bash with
 ``echo $BASH_VERSION``. On OS X, install bash via `Homebrew <http://brew.sh/>`_ (``brew install bash``), add
 ``/usr/local/bin/bash`` to ``/etc/shells``, and run ``chsh`` to change your shell.

 Global completion is not currently compatible with zsh.

.. note:: If you use setuptools/distribute ``scripts`` or ``entry_points`` directives to package your module,
 argcomplete will follow the wrapper scripts to their destination and look for ``PYTHON_ARGCOMPLETE_OK`` in the
 destination code.

Activating global completion
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The script ``activate-global-python-argcomplete`` will try to install the file
``bash_completion.d/python-argcomplete.sh`` (`see on GitHub`_) into an appropriate location on your system
(``/etc/bash_completion.d/`` or ``~/.bash_completion.d/``). If it
fails, but you know the correct location of your bash completion scripts directory, you can specify it with ``--dest``::

    activate-global-python-argcomplete --dest=/path/to/bash_completion.d

Otherwise, you can redirect its shellcode output into a file::

    activate-global-python-argcomplete --dest=- > file

The file's contents should then be sourced in e.g. ``~/.bashrc``.

.. _`see on GitHub`: https://github.com/kislyuk/argcomplete/blob/master/argcomplete/bash_completion.d/python-argcomplete.sh

Debugging
---------
Set the ``_ARC_DEBUG`` variable in your shell to enable verbose debug output every time argcomplete runs. Alternatively,
you can bypass the bash completion shellcode altogether, and interact with the Python code directly with something like
this::

    PROGNAME=./{YOUR_PY_SCRIPT} TEST_ARGS='some_arguments with autocompl' _ARC_DEBUG=1 COMP_LINE="$PROGNAME $TEST_ARGS" COMP_POINT=31 _ARGCOMPLETE=1 $PROGNAME 8>&1 9>>~/autocomplete_debug.log

Then tail::

    tail -f ~/autocomplete_debug.log

Acknowledgments
---------------
Inspired and informed by the optcomplete_ module by Martin Blais.

.. _optcomplete: http://pypi.python.org/pypi/optcomplete

Links
-----
* `Project home page (GitHub) <https://github.com/kislyuk/argcomplete>`_
* `Documentation (Read the Docs) <https://argcomplete.readthedocs.org/en/latest/>`_
* `Package distribution (PyPI) <https://pypi.python.org/pypi/argcomplete>`_
* `Change log <https://github.com/kislyuk/argcomplete/blob/master/Changes.rst>`_

Bugs
~~~~
Please report bugs, issues, feature requests, etc. on `GitHub <https://github.com/kislyuk/argcomplete/issues>`_.

License
-------
Licensed under the terms of the `Apache License, Version 2.0 <http://www.apache.org/licenses/LICENSE-2.0>`_.

.. image:: https://travis-ci.org/kislyuk/argcomplete.png
        :target: https://travis-ci.org/kislyuk/argcomplete
.. image:: https://img.shields.io/pypi/v/argcomplete.svg
        :target: https://pypi.python.org/pypi/argcomplete
.. image:: https://img.shields.io/pypi/dm/argcomplete.svg
        :target: https://pypi.python.org/pypi/argcomplete
.. image:: https://img.shields.io/pypi/l/argcomplete.svg
        :target: https://pypi.python.org/pypi/argcomplete
.. image:: https://readthedocs.org/projects/argcomplete/badge/?version=latest
        :target: https://argcomplete.readthedocs.org/
