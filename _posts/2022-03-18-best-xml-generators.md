---
layout: post
title: "best xml generators IMO"
---

## tcl xmlgen
- [xmlgen](http://tclxml.sourceforge.net/xmlgen/xmlgen.html)
- [wiki tcl - xmlgen](https://wiki.tcl-lang.org/page/xmlgen+%2F+htmlgen)
```
package require xmlgen
namespace import ::xmlgen::*
declaretags voo doo 

voo color=red align=left ! {
  doo - some text for doo
  doo - another doo element
  put text on the voo-level
} 
```
output
```
<voo color="red" align="left">
  <doo>doo some text for doo</doo>
  <doo>another doo element</doo>
  text on the voo-level
</voo> 
```

## groovy MarkupBuilder
- [groovy MarkupBuilder](http://docs.groovy-lang.org/docs/latest/html/documentation/core-domain-specific-languages.html#_markupbuilder)
```
def xml = new groovy.xml.MarkupBuilder()
xml.langs(type:"current"){
  language("Java")
  language("Groovy")
  language("JavaScript")
}
```
output
```
<langs type="current">
  <language>Java</language>
  <language>Groovy</language>
  <language>JavaScript</language>
</langs>
```

## python lxml.builder
- [lxml.builder module](https://lxml.de/apidoc/lxml.builder.html)
```
from lxml import etree as ET
from lxml.builder import E

# some common inline elements
A = E.a
I = E.i
B = E.b

def CLASS(v):
    # helper function, 'class' is a reserved word
    return {'class': v}

page = (
    E.html(
        E.head(
            E.title("This is a sample document")
        ),
        E.body(
            E.h1("Hello!", CLASS("title")),
            E.p("This is a paragraph with ", B("bold"), " text in it!"),
            E.p("This is another paragraph, with a ",
                A("link", href="http://www.python.org"), "."),
            E.p("Here are some reserved characters: <spam&egg>."),
            ET.XML("<p>And finally, here is an embedded XHTML fragment.</p>"),
        )
    )
)

print ET.tostring(page)
```
output
```
<html>
  <head>
    <title>This is a sample document</title>
  </head>
  <body>
    <h1 class="title">Hello!</h1>
    <p>This is a paragraph with <b>bold</b> text in it!</p>
    <p>This is another paragraph, with <a href="http://www.python.org">link</a>.</p>
    <p>Here are some reserved characters: &lt;spam&amp;egg&gt;.</p>
    <p>And finally, here is an embedded XHTML fragment.</p>
  </body>
</html>
```

## python yattag
- [yattag](https://www.yattag.org/)  
Once thought the design of yattag was very clever and amazing, but now it seems that there is still too much syntax noise in the code..  
```
#ignore
```
