This is a tutorial on creating syntax highlighting for cell magics in Jupyter Notebook. 

---
### Initial setup

Begin by creating a new repository on GitHub.
At the top level, create a Python virtual environment 
```python3 -m venv venv```

Activate environment ```source venv/bin/activate``` and create a pyproject.toml with the following dependencies

```
[tool.poetry]
name = "your_repo"
version = "0.1.0"
description = ""
authors = [" <your.email@example.com>"]
packages = [
    { include = "your_package"}
]

[tool.poetry.dependencies]
python = "3.11.x"
ipython = "^7.0"
notebook = "^7.0"
jupyter = "*"

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

### 1. Creating a custom cell magic

We will create a basic cell magic that computes simple arithmetic functions
Create a folder with a file of the same name ```calculator/calculator.py```
Create an ```__init__.py``` file that contains 
```
from .echo_append import load_ipython_extension
```

Your calculator.py file is where you will define the behavior of your cell magic

Cell magics are additional functions that extend the functionality of notebooks by creating custom behavior for cells

#### Importing Modules


calculator.py
```
from IPython.core.magic import Magics, magics_class, cell_magic
from IPython import get_ipython
```

#### Define and Register Magic
calculator.py
```
@magics_class
class MyMagics(Magics):
    @cell_magic
    def calculator(self, line, cell):
        expression = cell.strip()
        result = eval(expression)  
        print(f"{expression} = {result}")  

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
You should now have a functional cell magic that be loaded and executed 
using ```%load_ext calculator``` and ```%%calculator```

![image](https://github.com/user-attachments/assets/f5e40287-777a-4028-8d79-e7d4d4604632)


Your file structure should look something like this
![image](https://github.com/user-attachments/assets/6dd3adaa-83b8-4d63-beb3-7a7d325fbeee)

### 2. Creating a parser from Lezer grammar

To create the parser for the Lezer grammar, I recommend making a separate GitHub repo which you will later install in the Jupyter extension
Begin by cloning this repo: https://github.com/kyang04/example_code_mirror_highlighting 

After installing the necessary dependencies, navigate to the 
src/syntax.grammar
```
@top Program { expression* }

expression {
  Identifier |
  Name |
  Application { "(" expression* ")" }
}

NewExpression { @specialize<Identifier, "int"> expression }

@tokens {
  Identifier { $[a-z] $[a-zA-Z0-9_]* } 
  Name { $[A-Z] $[a-zA-Z0-9_]* } 
}

@skip {
  NewExpression
}

@detectDelim
}
```
This is an example ```.grammar```, modify this file to create a grammar of your specifications.
Documentation for creating your own lezer grammar can be found here: https://lezer.codemirror.net/docs/guide/#writing-a-grammar

After making all your changes to the grammar file run this command to convert it into a parser:
```lezer-generator src/example.grammar -o parser.js```

You should now see a file in your src folder called parser.js containing the information from the .grammar file

### 3. Create style tags for grammar

To define syntax highlighting for the tokens defined in your grammar, you have to assign each token
defined in your grammar with a variable


Navigate to ```src/highlight.ts```
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
```Hypl``` is the name of the language in this example repo, change this to your desired name.
In ```hyplHighlight```, each token/expression declared in the ```.grammar``` is assigned a styleTag, e.g t.variableName.
Be sure to assign every token/expression to a styleTag
In ```hyplHighlightStyle```, each tag is associated with a unique color in hex code. Choose a color for each tag

Import your custom highlighting into your index.ts file and export your syntax highlighting

```
import { parser } from "./syntax.grammar"
import { LRLanguage, LanguageSupport, indentNodeProp, foldNodeProp, foldInside, delimitedIndent, syntaxHighlighting } from "@codemirror/language"
import { hyplHighlight, hyplHighlightStyle } from './highlight';

export const HyplLanguage = LRLanguage.define({
    parser: parser.configure({
        props: [
            indentNodeProp.add({
                Application: delimitedIndent({ closing: ")", align: false }),
            }),
            foldNodeProp.add({
                Application: foldInside,
            }),
            hyplHighlight, 
        ],
    }),
    languageData: {
        commentTokens: { line: ";" },
    },
});

export function Hypl() {
    return new LanguageSupport(HyplLanguage, syntaxHighlighting(hyplHighlightStyle));
}
```

(Hypl should be replaced with the name of your language)

### 4. Testing and packaging your syntax highlighting 

We will be using rollup to distribute syntax highlighting. 

Run ```npx rollup -c``` in your src folder to package your parser and highlighting together 

You should now have a dist folder.

To test your syntax highlighting, ```npm run dev``` will allow you to open a browser to write text.


### 5. Creating a Jupyter Extension

We will create a jupyter extension to deploy the syntax highlighting 

You will need to install copier as a dependency first 

```
mkdir my_first_extension
cd my_first_extension
copier copy --trust https://github.com/jupyterlab/extension-template .
```

You now have a template for a jupyter extension

### 6. Installing CodeMirror syntax highlighting as a dependency

Create a makefile with the following structure 

```
all: clean init run

your_jupyter_extension/node_modules/your_highlight:
	cd syntax_highlighting && jlpm add your_highlight@git@github.com:github-user/your_highlight.git

.venv:
	python3 -m venv .venv
	.venv/bin/activate

.PHONY: clean
clean:
	cd your_jupyter_extension && jlpm your_highlight

.PHONY: build .venv
build: .init your_jupyter_extension/node_modules/your_highlight
	chmod +x .venv/bin/activate
	.venv/bin/activate
	pip install -ve your_jupyter_extension
	cd your_jupyter_extension && jlpm run build
	chmod -x .venv/bin/activate

.PHONY: run
run:
	jupyter notebook --port 8888 --no-browser

.PHONY: init
init: .venv .init
	chmod +x .venv/bin/activate
	./venv/bin/activate
	poetry install
	chmod -x .venv/bin/activate

.init:
	touch .init
```

To install your codemirror extension into this jupyter extension, you will run make build.
Whenever you make changes to your codemirror extension, always be sure to run make clean and make build again
to see your updates in Jupyter Notebook

### 7. Limiting syntax highlighting to cell magics

To prevent your custom syntax highlighting from applying to your entire notebook, you will need to 
ensure that the codemirror syntax only takes effect when your cell magic is present

```
import { Extension, Compartment } from "@codemirror/state";
import { JupyterFrontEnd, JupyterFrontEndPlugin } from "@jupyterlab/application";
import { EditorExtensionRegistry, IEditorExtensionRegistry } from "@jupyterlab/codemirror";
import { EditorState } from "@codemirror/state";
import { python } from "@codemirror/lang-python";
import { your_language } from "your_language_repo";

const languageConf = new Compartment();

const autoLanguage = EditorState.transactionExtender.of((tr) => {
  if (!tr.docChanged) return null;
  const docIsLangae = /^\s*%%calculator/.test(tr.newDoc.sliceString(0, 100)); // checking for magic
  return {
    effects: languageConf.reconfigure(docIsHypl ? your_language() : python()), // choose hypl or python based on cell magic
  };
});


function yourSyntaxExtension(): Extension {
  return [languageConf.of(python()), autoLanguage];
}
```

The code snippet above chooses syntax highlighting based on whether the ```%%calculator``` cell magic is present
If so, the syntax highlighting you've defined will appear, else it will continue to use Python highlighting

The following code registers your code mirror highlighting into jupyter notebook
```
const plugin: JupyterFrontEndPlugin<void> = {
  id: "@jupyterlab-examples/codemirror-extension:plugin",
  description: "A JupyterLab extension adding Hypl syntax highlighting.",
  autoStart: true,
  requires: [IEditorExtensionRegistry],
  activate: (app: JupyterFrontEnd, extensions: IEditorExtensionRegistry) => {
    // Register the editor extension
    extensions.addExtension(
      Object.freeze({
        name: "your_highlight",
        factory: () =>
          // The factory is called for every new CodeMirror editor
          EditorExtensionRegistry.createConfigurableExtension(() => yourSyntaxExtension())
      })
    );
  },
};

export default plugin;
```

