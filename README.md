# registry-py

## Overview

Metaprogramming means writing programs that manipulate other programs, such as compilers, interpreters, macros, etc.
Python enables metaprogramming in a number of ways, one of which is `metaclasses`, which are special classes that can
hook into child class creation methods, and can enforce invariants on those child classes. The
[abc.ABCMeta](https://docs.python.org/3/library/abc.html#module-abc) class is one example, and allows for the
declaration of [abstractmethod](https://docs.python.org/3/library/abc.html#abc.abstractmethod)s which must be
implemented by child classes. `registry-py` uses metaprogramming to implement the
[registry pattern](https://github.com/faif/python-patterns/blob/master/patterns/behavioral/registry.py), where a `metaclass`
stores a map of all child classes. This pattern is particularly useful for auto-discovery or auto-registration, where a
program doesn't doesn't need to know where code is stored to know what's been implemented.

## Installation

### Install from PyPi (preferred method)

```bash
pip install lc-registry
```

### Install from GitHub with Pip

```bash
pip install git+https://github.com/libcommon/registry-py.git@vx.x.x#egg=lc_registry
```

where `x.x.x` is the version you want to download.

## Install by Manual Download

To download the source distribution and/or wheel files, navigate to
`https://github.com/libcommon/registry-py/tree/releases/vx.x.x/dist`, where `x.x.x` is the version you want to install,
and download either via the UI or with a tool like wget. Then to install run:

```bash
pip install <downloaded file>
```

Do _not_ change the name of the file after downloading, as Pip requires a specific naming convention for installation files.

## Dependencies

`registry-py` does not have external dependencies. Only Python versions >= 3.6 are officially supported.

## Getting Started

Suppose you want to implement a plugin system for a command line tool. One approach could be to require that plugin
files are written to a specific folder, say `src/plugins/contrib`, and use
[importlib.import_module](https://docs.python.org/3.8/library/importlib.html#importlib.import_module) to import modules
from there on program startup. However, plugin files not in the proper directory will not get loaded, and finding
plugins requires walking a directory tree. Using a registry would alleviate both these issues.

```python
# plugin_registry.py
from abc import ABCMeta
from typing import Any, ClassVar, Dict, Type

from lc_registry import RegistryMetaclassMixin


class PluginRegistry(RegistryMetaclassMixin, ABCMeta):
    """Metaclass with registry."""
    __slots__ = ()
    _REGISTRY: ClassVar[Dict[str, Type[Any]]] = dict()

    @classmethod
    def _add_class(cls, name: str, new_cls: Type[Any]) -> None:
        # All plugins must be named "*Plugin"
        # NOTE: This is just an example what you can do with _add_class
        if name == "PluginBase" or not name.endswith("Plugin"):
            return None
        cls._REGISTRY[name] = new_cls
        return None

    @classmethod
    def run_plugins(cls):
        # Use cls._REGISTRY (or cls.registry() method) to
        # iterate over each plugin and run it.


class PluginBase(metaclass=PluginRegistry):
    """Base class for all plugins. All child classes
    will be added to the plugin registry once they come
    into scope (AKA when Python creates the type).
    """
    __slots__ = ()

    @abstractmethod
    def run_plugin(self, context: Dict[str, Any]) -> None:
        """Plugin entrypoint."""
        raise NotImplementedError


# line_count_plugin.py
from typing import Any, Dict

from plugin_registry import PluginBase


class LineCountPlugin(PluginBase):
    """Plugin that counts the number of lines in a file."""
    __slots__ = ()

    def run_plugin(self, context: Dict[str, Any]) -> None:
        if "filepath" in context:
            filepath = context.get("filepath")
            line_count = 0
            with open(filepath) as source_file:
                for _ in source_file:
                    line_count += 1
            print("{}\t{}".format(filepath, line_count))
```

Note the doc string for the `PluginBase` class in the example above: "_... once they come into scope (AKA when
Python creates the type)_." This is **important** - child classes will only be added to the registry if Python
loads them into the program, meaning if they get imported somewhere.

## Contributing/Suggestions

Contributions and suggestions are welcome! To make a feature request, report a bug, or otherwise comment on existing
functionality, please file an issue. For contributions please submit a PR, but make sure to lint, type-check, and test
your code before doing so. Thanks in advance!
