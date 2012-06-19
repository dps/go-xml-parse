go-xml-parse
============

Streaming XML parser example in Go

Intro
-----

I've recently been messing around with the XML dumps of Wikipedia. These are pretty huge XML files - for instance the most recent revision is 36G when uncompressed. That's a lot of XML!

I've been experimenting with a few different languages and parsers for my task (which also happens to involve some non trivial processing for each article) and found Go to be a great fit.

Go has a common library package for parsing xml (encoding/xml) which is very convenient to code against. However, the simple version of the API requires parsing the whole document at once, which for 36G is not a viable strategy. 

The parser can also be used in a streaming mode but I found the documentation and examples online to be terse and non-existant respectively, so here is my example code for parsing wikipedia with encoding/xml and a little explanation! (full example code at https://github.com/dps/go-xml-parse/blob/master/go-xml-parse.go)

Here's a little snippet of an example wikipedia page in the doc:

```xml
<page> 
  <title>Apollo 11</title> 
    <redirect title="Foo bar" /> 
    ... 
     <revision> 
     ... 
       <text xml:space="preserve"> 
       {{Infobox Space mission 
       |mission_name=<!--See above->; 
       |insignia=Apollo_11_insignia.png 
     ... 
       </text> 
     </revision> 
</page>
```

In our Go code, we define a struct to match the <page> element, its nested <redirect> element and grab a couple of fields we're interested in (<text> and <title>).
```go
type Redirect struct { 
    Title string `xml:"title,attr"` 
} 

type Page struct { 
    Title string `xml:"title"` 
    Redir Redirect `xml:"redirect"` 
    Text string `xml:"revision>text"` 
}
```
Now we would usually tell the parser that a wikipedia dump contains a bunch of <page>s and try to read the whole thing, but let's see how we stream it instead.

It's quite simple when you know how - iterate over tokens in the file until you encounter a StartElement with the name "page" and then use the magic decoder.DecodeElement API to unmarshal the whole following page into an object of the Page type defined above. Cool!

```go
decoder := xml.NewDecoder(xmlFile) 

for { 
    // Read tokens from the XML document in a stream. 
    t, _ := decoder.Token() 
    if t == nil { 
        break 
    } 
    // Inspect the type of the token just read. 
    switch se := t.(type) { 
    case xml.StartElement: 
        // If we just read a StartElement token 
        // ...and its name is "page" 
        if se.Name.Local == "page" { 
            var p Page 
            // decode a whole chunk of following XML into the
            // variable p which is a Page (se above) 
            decoder.DecodeElement(&p, &se) 
            // Do some stuff with the page. 
            p.Title = CanonicalizeTitle(p.Title)
            ...
        } 
...
```


I hope this saves you some time if you need to parse a huge XML file yourself.