# Python Built-ins

## Python Built-in Functions

### Input/Output
- print()
- input()

### Type Conversion and Date Structure Creation
- int()
- float()
- str()
- boolean()
- list()
- tuple()
- set()
- dict()

### Mathematical Operations
- abs()
- max()
- min()
- sum()
- round()
- pow()

### Object Introspection
- len()
- type()
- dir()
- getattr(), hasattr(), setattr(), delattr()


## Built-in Modules
### sys
Offers functions and variables that interact with the Python runtime environment and interpreter itself, such as accessing command line arguments or the Python version.

- Platform Information: 
- Input and Output using sys: sys.stdin, sys.stdout, sys.stderr
- Command-Line Arguments: sys.argv, sys.exit()
- Modules: sys.path, sys.modules()
- Reference Count: sys.getrefcount(obj)
- sys.getrecursionlimit()/sys.setrecursionlimit(num)
- sys.gettrace()/sys.settrace(), sys.getProfile()/sys.setProfile(): used for implementing debuggers, profilers and coverage tools
- sys.getswitchinterval()/sys.setswitchinterval(sec)
- sys.maxsize()


### os
Provides a way of interacting with the operating system, useful for file system operations, managing directories, and handling environment variables.

- Current Work Directory (CWD): os.getcwd(), os.chdir(path)
- Directory: os.mkdir(), os.makedirs(), os.remove()
- 

### math
Implements a variety of mathematical functions for floating-point numbers, including trigonometric functions, logarithms, and constants like pi and e.

### random
Used to generate pseudo-random numbers and perform random operations, such as selecting a random item from a list or shuffling elements.

### datetime
Provides classes for manipulating dates and times.

### re
Offers robust support for regular expressions (regex), enabling powerful pattern matching and string processing.

### json
A built-in module for encoding Python objects into JSON formatted strings and decoding JSON strings back into Python objects, essential for web development and data exchange.

### sqlite3
Provides a DB-API 2.0 implementation for working with SQLite databases.

### urllib
A package that collects modules for handling URLs, used for fetching URLs and parsing URL components.

### collections
Provides specialized container datatypes like OrderedDict, namedtuple, and default dict which offer alternatives to Python's general-purpose built-in containers. 