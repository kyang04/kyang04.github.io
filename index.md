This is a tutorial on creating syntax highlighting for cell magics in Jupyter Notebook. 

---

### 1. Creating a custom cell magic

Cell magics are additional functions that extend the functionality of notebooks by creating custom behavior for cells

#### Importing Modules



```
from IPython.core.magic import Magics, magics_class, cell_magic
from IPython import get_ipython
```

#### Define and Register Magic

```
@magics_class
class MyMagics(Magics):
    @cell_magic
    def cmagic(self, line, cell):
        "my cell magic"
        return line, cell

# In order to actually use these magics, you must register them with a
# running IPython.

def load_ipython_extension(ipython):
    """
    Any module file that define a function named `load_ipython_extension`
    can be loaded via `%load_ext module.path` or be configured to be
    autoloaded by IPython at startup time.
    """
    # You can register the class itself without instantiating it.  IPython will
    # call the default constructor on it.
    ipython.register_magics(MyMagics)
```
