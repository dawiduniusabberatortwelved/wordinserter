Wordinserter
===
This module allows you to insert HTML into a Word Document, as well as allowing you to programmatically build 
word documents in pure Python (Python 3.x only at the moment). After running `pip install wordinserter` you can use the
`wordinserter` CLI to quickly generate test documents:

```bash
# Download https://raw.githubusercontent.com/orf/wordinserter/master/Tests/docs/table_widths.html
wordinserter table_widths.html --style="table { background-color: red }"
```

This should open Word and insert three tables, each of them styled with a red background.

This library currently supports many features, all through HTML:

   - All standard styles (italics, bold, underlined) through HTML tags
   - Named bookmarks in documents via element `id` attributes
   - A subset of CSS for elements, with more that can be easily added as needed
   - Including document-wide stylesheets while adding elements
   - In-built syntax highlighting for `<pre>` and `<code>` blocks
   - Supports complex merged tables, with rowspans and colspans
   - Arbitrarily nested lists of differing types (bullet, numbered, roman numerals)
   - Hyperlinks to bookmarks within the document using classic links or using Word 'fields'
   - Images, with support for footnotes, 404 and embedded base64 data-uri images
    
There is a [comparison document](https://rawgit.com/orf/wordinserter/master/tests/comparison/report.html) showing 
the output of WordInserter against Chrome, check it out to see what the library can do.

API
===
The API is really simple to use:

``` python
from wordinserter import parse, insert

operations = parse(html, parser="html")
insert(operations, document=document, constants=constants)
```
    
Inserting HTML into a Word document is a two step process: first the input has to be parsed into a sequence 
of operations, which is then *inserted* into a Word document. This library currently only supports inserting using the 
Word COM interface which means it is Windows specific at the moment.

Below is a more complex example including starting word that will insert a representation of the HTML code
into the new word document, including the image, caption and list.

``` python
from wordinserter import insert, parse
from comtypes.client import CreateObject

# This opens Microsoft Word and creates a new document.
word = CreateObject("Word.Application")
word.Visible = True # Don't set this to True in production!
document = word.Documents.Add()
from comtypes.gen import Word as constants

html = """
<h3>This is a title</h3>
<p><img src="http://placehold.it/150x150" alt="I go below the image as a caption"></p>
<p><i>This is <b>some</b> text</i> in a <a href="http://google.com">paragraph</a></p>
<ul>
    <li>Boo! I am a <b>list</b></li>
</ul>
"""

# Parse the HTML into a list of operations then feed them into insert.
operations = parse(html, parser="html")
insert(operations, document=document, constants=constants)
```

What's with the constants part? Wordinserter is agnostic to the COM library you use. Each library exposes constant 
values that are needed by Wordinserter in a different way: the pywin32 library exposes it as win32com.client.constants 
whereas the comtypes library exposes them as a module that resides in comtypes.gen. Rather than guess which one you 
are using Wordinserter requires you to pass the right one in explicitly. If you need to mix different constant groups you 
can use the `CombinedConstants` class:

```python
from wordinserter import CombinedConstants
from comtypes.gen import Word as word_constants
from comtypes.gen import Office as office_constants

constants = CombinedConstants(word_constants, office_constants)
```


### Install
Get it [from PyPi here](https://pypi.python.org/pypi/wordinserter), using `pip install wordinserter`. This has been built with word 2010 and 2013, older 
versions may produce different results.


## Supported Operations
WordInserter currently supports a range of different operations, including code blocks, font size/colors, images, 
hyperlinks, numbered and bullet lists.

#### Stylesheets?
Wordinserter has support for stylesheets! Every element can be styled with inline styles (`style='whatever'`) but this
gets tedious at scale. You can pass CSS stylesheets to the `parse` function:

```python
html = "<p class="mystyle">Hello Word</p>"
stylesheet = """
.mystyle {
    color: red;
}
"""

operations = parse(html, parser="html", stylesheets=[stylesheet])
insert(operations, document=document, constants=constants)

```

This will render "Hello Word" in red. Inheritance is respected, so child styles override parent ones.


#### Why aren't my lists showing up properly?
There are two ways people write lists in HTML, one with each sub-list as a child of the parent list, or as a child of a
list element. Below is a sample of the two different ways, both of which display correctly in all browsers:

```html
<ol>
    <li>
        I'm a list element
    </li>
    <ul>
        <li>I'm a sub list!</li>
    </ul>
</ol>
```
```html
<ol>
    <li>
        I'm a list element
        <ul>
            <li>I'm a sub list!</li>
        </ul>
    </li>
</ol>
```

The second way is correct according to the HTML specification. `lxml` parses the first structure incorrectly in some cases,
which leads to weird list behavior. There isn't much this library can do about that, so make sure your lists are
in the second format.

One other thing to note: Word does not support lists with mixed list-types on a single level. i.e this HTML will render
incorrectly:

```html
<ol>
    <li>
        <ul><li>Unordered List On Level #1</li></ul>
        <ol><li>Ordered List On Level #1</li></ul>
    </li>
</ol>
```
