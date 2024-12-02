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

### 2. Creating a parser from Lezer grammar

This tutorial will cover a grammar for a calculator syntax highlighting. 
More information on creating your own grammar: https://lezer.codemirror.net/docs/guide/#writing-a-grammar

example.grammar
```
@top Program { expression }

expression { Name | Number | BinaryExpression }

BinaryExpression { "(" expression ("+" | "-") expression ")" }

@tokens {
  Name { @asciiLetter+ }
  Number { @digit+ }
}
```
To convert this file into a parser, run this command:
```lezer-generator example.grammar```

Import this parser into the index.ts file
```import { parser } from "./example.grammar"```
### 3. Create style tags for grammar

To define syntax highlighting for the tokens defined in your grammar 

```
import { styleTags, tags as t } from "@lezer/highlight";
import { HighlightStyle } from "@codemirror/language";

export const hyplHighlight = styleTags({
  Identifier: t.variableName,
  Name: t.name,
  "( )": t.paren,
  NewExpression: t.modifier
});

export const hyplHighlightStyle = HighlightStyle.define([
  { tag: t.variableName, color: "#2689C7" },
  { tag: t.name, color: "#d90cfe" },
  { tag: t.modifier, color: "#91041e" },
]);

export const hyplHighlightExtension = [hyplHighlight, hyplHighlightStyle];
```
