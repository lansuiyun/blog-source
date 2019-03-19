---
title: fastjson
date: 2016-05-14 15:22:27
tags: fastjson
categories: java tool
---
Fastjson是一个Java语言编写的高性能功能完善的JSON库。它采用一种“假定有序快速匹配”的算法，把JSON Parse的性能提升到极致，是目前Java语言中最快的JSON库。Fastjson接口简单易用，已经被广泛使用在缓存序列化、协议交互、Web输出、Android客户端等多种应用场景。
<!--more-->
### 主要API
fastjson入口类是com.alibaba.fastjson.JSON，主要的API是JSON.toJSONString，和parseObject。
- 序列化
```
String jsonString = JSON.toJSONString(obj);
```
- 反序列化
```
Vo vo = JSON.parseObject("...",Vo.class);
```
- 泛型反序列化
```
List<Vo> list = JSON.parserObject("...",new TypeReference<List<Vo>>(){});
```

### 定制序列化
#### 通过@JSONField定制序列化
- JSONField介绍
```
package com.alibaba.fastjson.annotation;
 
public @interface JSONField {
    // 配置序列化和反序列化的顺序，1.1.42版本之后才支持
    int ordinal() default 0;
 
     // 指定字段的名称
    String name() default "";
 
    // 指定字段的格式，对日期格式有用
    String format() default "";
 
    // 是否序列化
    boolean serialize() default true;
 
    // 是否反序列化
    boolean deserialize() default true;
}
```
- 配置方式
可以配置在getter/setter方法或者字段上。例如：
```
 public class A {
      private int id;

      @JSONField(name="ID")
      public int getId() {return id;}
      @JSONField(name="ID")
      public void setId(int value) {this.id = id;}
 }
```
```
 public class A {
      @JSONField(name="ID")
      private int id;

      public int getId() {return id;}
      public void setId(int value) {this.id = id;}
 }
```
- 使用format配置日期格式化
```
 public class A {
      // 配置date序列化和反序列使用yyyyMMdd日期格式
      @JSONField(format="yyyyMMdd")
      public Date date;
 }
```
- 序列化、反序列化时排除指定字段
```
 public class A {
      @JSONField(serialize=false)
      public Date date;
 }

 public class A {
      @JSONField(deserialize=false)
      public Date date;
 }
```
- 使用ordinal指定字段的顺序。缺省根据fieldName的字母序进行序列化
```
public static class VO {
    @JSONField(ordinal = 3)
    private int f0;
 
    @JSONField(ordinal = 2)
    private int f1;
 
    @JSONField(ordinal = 1)
    private int f2;
}
```
#### 使用@JSONType配置
和JSONField类似，但JSONType配置在类上，而不是field或者getter/setter方法上。

#### 通过SerializeFilter定制序列化
通过SerializeFilter可以使用扩展编程的方式实现定制序列化。fastjson提供了多种SerializeFilter：
 
- PropertyPreFilter 根据PropertyName判断是否序列化
- PropertyFilter 根据PropertyName和PropertyValue来判断是否序列化
- NameFilter 修改Key，如果需要修改Key,process返回值则可
- ValueFilter 修改Value
- BeforeFilter 序列化时在最前添加内容
- AfterFilter 序列化时在最后添加内容
以上的SerializeFilter在JSON.toJSONString中可以使用。
```
  SerializeFilter filter = ...; // 可以是上面5个SerializeFilter的任意一种。
  JSON.toJSONString(obj, filter);
```
##### PropertyFilter 根据PropertyName和PropertyValue来判断是否序列化
```
 public interface PropertyFilter extends SerializeFilter {
      boolean apply(Object object, String propertyName, Object propertyValue);
  }
```
可以通过扩展实现根据object或者属性名称或者属性值进行判断是否需要序列化。例如：
```
    PropertyFilter filter = new PropertyFilter() {
 
        public boolean apply(Object source, String name, Object value) {
            if ("id".equals(name)) {
                int id = ((Integer) value).intValue();
                return id >= 100;
            }
            return false;
        }
    };
 
    JSON.toJSONString(obj, filter); // 序列化的时候传入filter
```
##### PropertyPreFilter 根据PropertyName判断是否序列化
和PropertyFilter不同只根据object和name进行判断，在调用getter之前，这样避免了getter调用可能存在的异常。
```
 public interface PropertyPreFilter extends SerializeFilter {
      boolean apply(JSONSerializer serializer, Object object, String name);
  }
```
- SimplePropertyPreFilter接口
```
public class SimplePropertyPreFilter implements PropertyPreFilter {      
    public SimplePropertyPreFilter(String... properties){
          this(null, properties);
    }

    public SimplePropertyPreFilter(Class<?> clazz, String... properties){
          // ... ...
    }

    public Class<?> getClazz() {
          return clazz;
    }

    public Set<String> getIncludes();

    public Set<String> getExcludes();

    /**     * @since 1.2.9     */public int getMaxLevel();

    /**     * @since 1.2.9     */public void setMaxLevel(int maxLevel)

    //...
}
```
你可以配置includes、excludes。当class不为null时，针对特定类型；当class为null时，针对所有类型。
 当includes的size > 0时，属性必须在includes中才会被序列化，excludes优先于includes。
- 使用方式
```
VO vo = new VO();

vo.setId(123);
vo.setName("flym");

SimplePropertyPreFilter filter = new SimplePropertyPreFilter(VO.class, "name");
Assert.assertEquals("{\"name\":\"flym\"}", JSON.toJSONString(vo, filter));

public static class VO {
      private int    id;
      private String name;

      public int getId() {
          return id;
      }
      public void setId(int id) {
          this.id = id;
      }

      public String getName() {
          return name;
      }
      public void setName(String name) {
          this.name = name;
    }
}
```

##### NameFilter 序列化时修改Key
如果需要修改Key,process返回值则可
```
public interface NameFilter extends SerializeFilter {
    String process(Object object, String propertyName, Object propertyValue);
}
```
fastjson内置一个PascalNameFilter，用于输出将首字符大写的Pascal风格。 例如：
```
import com.alibaba.fastjson.serializer.PascalNameFilter;

Object obj = ...;
String jsonStr = JSON.toJSONString(obj, new PascalNameFilter());
```
##### ValueFilter 序列化是修改Value
```
  public interface ValueFilter extends SerializeFilter {
      Object process(Object object, String propertyName, Object propertyValue);
  }
```
##### BeforeFilter 序列化时在最前添加内容
```
  public abstract class BeforeFilter implements SerializeFilter {
      protected final void writeKeyValue(String key, Object value) { ... }
      // 需要实现的抽象方法，在实现中调用writeKeyValue添加内容
      public abstract void writeBefore(Object object);
  }
```
##### AfterFilter 序列化时在最前添加内容
```
  public abstract class AfterFilter implements SerializeFilter {
      protected final void writeKeyValue(String key, Object value) { ... }
      // 需要实现的抽象方法，在实现中调用writeKeyValue添加内容
      public abstract void writeAfter(Object object);
  }
```

##### 类级别配置Filter
对于框架来说，如果在toJSONString的时候，传入SerializeFilter，会导致对所有的类型做过滤，性能会受到一定影响。在1.2.10版本之后，fastjson提供类级别的SerializeFilter支持。
```
package com.alibaba.fastjson.serializer;

public class SerializeConfig {
    public void addFilter(Class<?> clazz, SerializeFilter filter);
}
```
```
public class ClassNameFilterTest extends TestCase {
    public void test_filter() throws Exception {
        NameFilter upcaseNameFilter = new NameFilter() {
            public String process(Object object, String name, Object value) {
                return name.toUpperCase();
            }
        };
        SerializeConfig.getGlobalInstance() //
                       .addFilter(A.class, upcaseNameFilter);

        Assert.assertEquals("{\"ID\":0}", JSON.toJSONString(new A()));
        Assert.assertEquals("{\"id\":0}", JSON.toJSONString(new B()));
    }

    public static class A {
        public int id;
    }

    public static class B {
        public int id;
    }
}
```
##### BeanToArray
在fastjson中，支持一种叫做BeanToArray的映射模式。普通模式下，JavaBean映射成json object，BeanToArray模式映射为json array。
- Sample 1
```
class Mode {
   public int id;
   public int name;
}

Model model = new Model();
model.id = 1001;
model.name = "gaotie";

// {"id":1001,"name":"gaotie"}
String text_normal = JSON.toJSONString(model); 

// [1001,"gaotie"]
String text_beanToArray = JSON.toJSONString(model, SerializerFeature.BeanToArray); 

// support beanToArray & normal mode
JSON.parseObject(text_beanToArray, Mode.class, Feature.SupportArrayToBean); 

```
上面的例子中，BeanToArray模式下，少了Key的输出，节省了空间，json字符串较小，性能也会更好。
- Sample 2
```
class Company {
     public int code;
     public List<Department> departments = new ArrayList<Department>();
}

@JSONType(serialzeFeatures=SerializerFeature.BeanToArray, parseFeatures=Feature.SupportArrayToBean))
class Department {
     public int id;
     public Stirng name;
     public Department() {}
     public Department(int id, String name) {this.id = id; this.name = name;}
}


Company company = new Company();
company.code = 100;
company.departments.add(new Department(1001, "Sales"));
company.departments.add(new Department(1002, "Financial"));

// {"code":10,"departments":[[1001,"Sales"],[1002,"Financial"]]}
String text = JSON.toJSONString(commpany); 
```
在这个例子中，如果Company的属性departments元素很多，局部采用BeanToArray就可以获得很好的性能，而整体又能够获得较好的可读性。
- 性能
使用BeanToArray模式，可以获得媲美protobuf的性能。
#### 通过ParseProcess定制反序列化
ParseProcess是编程扩展定制反序列化的接口。fastjson支持如下ParseProcess：
 
- ExtraProcessor 用于处理多余的字段
- ExtraTypeProvider 用于处理多余字段时提供类型信息
##### 使用ExtraProcessor 处理多余字段
```
public static class VO {
    private int id;
    private Map<String, Object> attributes = new HashMap<String, Object>();
    public int getId() { return id; }
    public void setId(int id) { this.id = id;}
    public Map<String, Object> getAttributes() { return attributes;}
}

ExtraProcessor processor = new ExtraProcessor() {
    public void processExtra(Object object, String key, Object value) {
        VO vo = (VO) object;
        vo.getAttributes().put(key, value);
    }
};

VO vo = JSON.parseObject("{\"id\":123,\"name\":\"abc\"}", VO.class, processor);
Assert.assertEquals(123, vo.getId());
Assert.assertEquals("abc", vo.getAttributes().get("name"));
```
##### 使用ExtraTypeProvider 为多余的字段提供类型
```
public static class VO {
    private int id;
    private Map<String, Object> attributes = new HashMap<String, Object>();
    public int getId() { return id; }
    public void setId(int id) { this.id = id;}
    public Map<String, Object> getAttributes() { return attributes;}
}

class MyExtraProcessor implements ExtraProcessor, ExtraTypeProvider {
    public void processExtra(Object object, String key, Object value) {
        VO vo = (VO) object;
        vo.getAttributes().put(key, value);
    }

    public Type getExtraType(Object object, String key) {
        if ("value".equals(key)) {
            return int.class;
        }
        return null;
    }
};
ExtraProcessor processor = new MyExtraProcessor();

VO vo = JSON.parseObject("{\"id\":123,\"value\":\"123456\"}", VO.class, processor);
Assert.assertEquals(123, vo.getId());
Assert.assertEquals(123456, vo.getAttributes().get("value")); // value本应该是字符串类型的，通过getExtraType的处理变成Integer类型了。
```

### 自定义序列化
fastjson提供了自定义序列化的接口，当你通过配置无法满足自定义序列化需求时，你可以使用ObjectSerializer接口。
#### API
- ObjectSerializer接口
```
package com.alibaba.fastjson.serializer;

public interface ObjectSerializer {
    void write(JSONSerializer serializer, 
               Object object, 
               Object fieldName, 
               Type fieldType, 
               int features) throws IOException;
}
```
- 注册API
```
package com.alibaba.fastjson.serializer;

public class SerializeConfig {
    public boolean put(Type key, ObjectSerializer value);
}
```
#### Sample
- 实现ObjectSerializer
```
public class CharacterSerializer implements ObjectSerializer {
    public void write(JSONSerializer serializer, 
                      Object object, 
                      Object fieldName, 
                      Type fieldType, 
                      int features) throws IOException {
        SerializeWriter out = serializer.out;

        Character value = (Character) object;
        if (value == null) {
            out.writeString("");
            return;
        }

        char c = value.charValue();
        if (c == 0) {
            out.writeString("\u0000");
        } else {
            out.writeString(value.toString());
        }
    }
}
```
- 注册
```
SerializeConfig.getGlobalInstance().put(Character.class, new CharacterSerializer());
```
### 自定义反序列化
fastjson支持注册ObjectDeserializer实现自定义反序列化。要自定义序列化，首先需要实现一个ObjectDeserializer，然后注册到ParserConfig中。
#### API
- ObjectDeserializer 定义
```
package com.alibaba.fastjson.parser.deserializer;

public interface ObjectDeserializer {
    <T> T deserialze(DefaultJSONParser parser, Type type, Object fieldName);

    int getFastMatchToken();
}
```
- 注册自定义ObjectDeserializer API
```
package com.alibaba.fastjson.parser;

public class ParserConfig {
    public void putDeserializer(Type type, ObjectDeserializer deserializer);
}
```
#### Sample
- Model定义
```
public static enum OrderActionEnum {
                                    FAIL(1), SUCC(0);

    private int code;

    OrderActionEnum(int code){
        this.code = code;
    }
}

public static class Msg {

    public OrderActionEnum actionEnum;
    public String          body;
}
```
- 自定义实现ObjectDeserializer
```
public static class OrderActionEnumDeser implements ObjectDeserializer {

    @SuppressWarnings("unchecked")
    @Overridepublic <T> T deserialze(DefaultJSONParser parser, Type type, Object fieldName) {
        Integer intValue = parser.parseObject(int.class);
        if (intValue == 1) {
            return (T) OrderActionEnum.FAIL;
        } else if (intValue == 0) {
            return (T) OrderActionEnum.SUCC;
        }
        throw new IllegalStateException();
    }

    @Overridepublic int getFastMatchToken() {
        return JSONToken.LITERAL_INT;
    }

}
```
- 注册并使用ObjectDeserializer
```
ParserConfig.getGlobalInstance().putDeserializer(OrderActionEnum.class, new OrderActionEnumDeser());

{
    Msg msg = JSON.parseObject("{\"actionEnum\":1,\"body\":\"A\"}", Msg.class);
    Assert.assertEquals(msg.body, "A");
    Assert.assertEquals(msg.actionEnum, OrderActionEnum.FAIL);
}
{
    Msg msg = JSON.parseObject("{\"actionEnum\":0,\"body\":\"B\"}", Msg.class);
    Assert.assertEquals(msg.body, "B");
    Assert.assertEquals(msg.actionEnum, OrderActionEnum.SUCC);
}
```

### 处理超大对象和超大JSON文本
当需要处理超大JSON文本时，需要Stream API，在fastjson-1.1.32版本中开始提供Stream API。
#### 序列化
##### 超大JSON数组序列化
如果你的JSON格式是一个巨大的JSON数组，有很多元素，则先调用startArray，然后挨个写入对象，然后调用endArray。
```
  JSONWriter writer = new JSONWriter(new FileWriter("/tmp/huge.json"));
  writer.startArray();
  for (int i = 0; i < 1000 * 1000; ++i) {
        writer.writeValue(new VO());
  }
  writer.endArray();
  writer.close();
```
##### 超大JSON对象序列化
如果你的JSON格式是一个巨大的JSONObject，有很多Key/Value对，则先调用startObject，然后挨个写入Key和Value，然后调用endObject。
```
  JSONWriter writer = new JSONWriter(new FileWriter("/tmp/huge.json"));
  writer.startObject();
  for (int i = 0; i < 1000 * 1000; ++i) {
        writer.writeKey("x" + i);
        writer.writeValue(new VO());
  }
  writer.endObject();
  writer.close();
```
#### 反序列化
```
  JSONReader reader = new JSONReader(new FileReader("/tmp/huge.json"));
  reader.startArray();
  while(reader.hasNext()) {
        VO vo = reader.readObject(VO.class);
        // handle vo ...
  }
  reader.endArray();
  reader.close();
```
```
  JSONReader reader = new JSONReader(new FileReader("/tmp/huge.json"));
  reader.startObject();
  while(reader.hasNext()) {
        String key = reader.readString();
        VO vo = reader.readObject(VO.class);
        // handle vo ...
  }
  reader.endObject();
  reader.close();
```
### LabelFilter
在Fastjson 1.2.7版本之后，fastjson提供LabelFilter功能，用于不同的场景定制序列化。
 
需要使用LabelFilter，首先需要用@JSONField配置label。如：
```
public static class VO {
    private int    id;
    private String name;
    private String password;
    private String info;
    @JSONField(label = "normal")
    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    @JSONField(label = "normal")
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    @JSONField(label = "secret")
    public String getPassword() {
        return password;
    }
    public void setPassword(String password) {
        this.password = password;
    }
    public String getInfo() {
        return info;
    }   
    public void setInfo(String info) {
        this.info = info;
    }
}
```
根据不同场景过滤，如下：
```
VO vo = new VO();
vo.setId(123);
vo.setName("wenshao");
vo.setPassword("ooxxx");
vo.setPassword("ooxxx");

String text1 = JSON.toJSONString(vo, Labels.includes("normal"));
Assert.assertEquals("{\"id\":123,\"name\":\"wenshao\"}", text1);

String text2 = JSON.toJSONString(vo, Labels.excludes("secret"));
Assert.assertEquals("{\"id\":123,\"info\":\"fofo\",\"name\":\"wenshao\"}", text2);
```
### 从流中反序列化
在1.2.11版本中，fastjson新增加了对InputStream的支持
```
package com.alibaba.fastjson;

public abstract class JSON {
    public static <T> T parseObject(InputStream is, //Type type, //Feature... features) throws IOException;

    public static <T> T parseObject(InputStream is, //Charset charset, //Type type, //Feature... features) throws IOException;
}
```
- Sample
```
import com.alibaba.fastjson;
import java.nio.charset.Charset;
class Model {
    public int value;
}

InputStream is = ...Model model = JSON.parseObject(is, Model.class);
Model model2 = JSON.parseObject(is, Charset.from("UTF-8"), Model.class);
```
### 将对象序列化到流中
在1.2.11版本中，JSON类新增对OutputStream/Writer直接支持。
```
package com.alibaba.fastjson;

public abstract class JSON {
   public static final int writeJSONString(OutputStream os, // Object object, // SerializerFeature... features) throws IOException;

    public static final int writeJSONString(OutputStream os, //Charset charset, //  Object object, // SerializerFeature... features) throws IOException;

   public static final int writeJSONString(Writer os, // Object object, // SerializerFeature... features) throws IOException;
}
```
- Sample
```
import com.alibaba.fastjson;
import java.nio.charset.Charset;
class Model {
    public int value;
}

Model model = new Model();
model.value = 1001;

OutputStream os = ...;
JSON.writeJSONString(os, model);
JSON.writeJSONString(os, Charset.from("GB18030"), model);

Writer writer = ...;
JSON.writeJSONString(os, model);
```

### JSONPath
fastjson 1.2.0之后的版本支持JSONPath。这是一个很强大的功能，可以在java框架中当作对象查询语言（OQL）来使用。
#### API
```
package com.alibaba.fastjson;

public class JSONPath {          
     //  求值，静态方法
     public static Object eval(Object rootObject, String path);

     // 计算Size，Map非空元素个数，对象非空元素个数，Collection的Size，数组的长度。其他无法求值返回-1
     public static int size(Object rootObject, String path);

     // 是否包含，path中是否存在对象
     public static boolean contains(Object rootObject, String path) { }

     // 是否包含，path中是否存在指定值，如果是集合或者数组，在集合中查找value是否存在
     public static boolean containsValue(Object rootObject, String path, Object value) { }

     // 修改制定路径的值，如果修改成功，返回true，否则返回false
     public static boolean set(Object rootObject, String path, Object value) {}

     // 在数组或者集合中添加元素
     public static boolean array_add(Object rootObject, String path, Object... values);
}
```
建议缓存JSONPath对象，这样能够提高求值的性能。
#### 支持语法
| JSONPATH  |描述|
|-:|:-|
| $    |根对象，例如$.name|
| [num]     |数组访问，其中num是数字，可以是负数。例如$[0].leader.departments[-1].name|
| [num0,num1,num2...] |数组多个元素访问，其中num是数字，可以是负数，返回数组中的多个元素。例如$[0,3,-2,5]|
| [start:end]    |数组范围访问，其中start和end是开始小表和结束下标，可以是负数，返回数组中的多个元素。例如$[0:5]|
| [start:end :step]   | 数组范围访问，其中start和end是开始小表和结束下标，可以是负数；step是步长，返回数组中的多个元素。例如$[0:5:2]|
| [?(key)]  |对象属性非空过滤，例如$.departs[?(name)]|
| [key > 123]    |数值类型对象属性比较过滤，例如$.departs[id >= 123]，比较操作符支持=,!=,>,>=,<,<=|
| [key = '123']  |字符串类型对象属性比较过滤，例如$.departs[name = '123']，比较操作符支持=,!=,>,>=,<,<= |
| [key like 'aa%']    |字符串类型like过滤，例如$.departs[name like 'sz*']，通配符只支持% ,支持not like|
| [key rlike 'regexpr']    |字符串类型正则匹配过滤，例如departs[name like 'aa(.)*']，正则语法为jdk的正则语法，支持not rlike|
| [key in ('v0', 'v1')]    |IN过滤, 支持字符串和数值类型 例如 \$.departs[name in ('wenshao','Yako')]  \$.departs[id not in (101,102)] |
| [key between 234 and 456]    | BETWEEN过滤, 支持数值类型，支持not between 例如 \$.departs[id between 101 and 201],\$.departs[id not between 101 and 201] |
| length() 或者 size()  |数组长度。例如$.values.size(),支持类型java.util.Map和java.util.Collection和数组|
| .    |属性访问，例如$.name|
| *    |对象的所有属性，例如$.leader.*|
| ['key']   |属性访问。例如$['name']|
| ['key0','key1']     |多个属性访问。例如$['id','name']
以下两种写法的语义是相同的：
```
$.store.book[0].title
```
```
$['store']['book'][0]['title']
```
#### 语法示例
| JSONPath	|语义 |
|:-:|:-:|
| $	|根对象 |
| $[-1]	|最后元素 |
| $[:-2]	|第1个至倒数第2个 |
| $[1:]	|第2个之后所有元素|
| $[1,2,3]	|集合中1,2,3个元素|
#### API 示例
```
public void test_entity() throws Exception {
   Entity entity = new Entity(123, new Object());

  Assert.assertSame(entity.getValue(), JSONPath.eval(entity, "$.value")); 
  Assert.assertTrue(JSONPath.contains(entity, "$.value"));
  Assert.assertTrue(JSONPath.containsValue(entity, "$.id", 123));
  Assert.assertTrue(JSONPath.containsValue(entity, "$.value", entity.getValue())); 
  Assert.assertEquals(2, JSONPath.size(entity, "$"));
  Assert.assertEquals(0, JSONPath.size(new Object[], "$")); 
}

public static class Entity {
   private Integer id;
   private String name;
   private Object value;

   public Entity() {}
   public Entity(Integer id, Object value) { this.id = id; this.value = value; }
   public Entity(Integer id, String name) { this.id = id; this.name = name; }
   public Entity(String name) { this.name = name; }

   public Integer getId() { return id; }
   public Object getValue() { return value; }        
   public String getName() { return name; }

   public void setId(Integer id) { this.id = id; }
   public void setName(String name) { this.name = name; }
   public void setValue(Object value) { this.value = value; }
}
```
- 读取集合多个元素的某个属性
```
List<Entity> entities = new ArrayList<Entity>();
entities.add(new Entity("wenshao"));
entities.add(new Entity("ljw2083"));

List<String> names = (List<String>)JSONPath.eval(entities, "$.name"); // 返回enties的所有名称
Assert.assertSame(entities.get(0).getName(), names.get(0));
Assert.assertSame(entities.get(1).getName(), names.get(1));
```
- 返回集合中多个元素
```
List<Entity> entities = new ArrayList<Entity>();
entities.add(new Entity("wenshao"));
entities.add(new Entity("ljw2083"));
entities.add(new Entity("Yako"));

List<Entity> result = (List<Entity>)JSONPath.eval(entities, "[1,2]"); // 返回下标为1和2的元素
Assert.assertEquals(2, result.size());
Assert.assertSame(entities.get(1), result.get(0));
Assert.assertSame(entities.get(2), result.get(1));
```
- 按范围返回集合的子集
```
List<Entity> entities = new ArrayList<Entity>();
entities.add(new Entity("wenshao"));
entities.add(new Entity("ljw2083"));
entities.add(new Entity("Yako"));

List<Entity> result = (List<Entity>)JSONPath.eval(entities, "[0:2]"); // 返回下标从0到2的元素
Assert.assertEquals(3, result.size());
Assert.assertSame(entities.get(0), result.get(0));
Assert.assertSame(entities.get(1), result.get(1));
Assert.assertSame(entities.get(2), result.get(1));
```
- 通过条件过滤，返回集合的子集
```
List<Entity> entities = new ArrayList<Entity>();
entities.add(new Entity(1001, "ljw2083"));
entities.add(new Entity(1002, "wenshao"));
entities.add(new Entity(1003, "yakolee"));
entities.add(new Entity(1004, null));

List<Object> result = (List<Object>) JSONPath.eval(entities, "[id in (1001)]");
Assert.assertEquals(1, result.size());
Assert.assertSame(entities.get(0), result.get(0));
```
- 根据属性值过滤条件判断是否返回对象，修改对象，数组属性添加元素
```
Entity entity = new Entity(1001, "ljw2083");
Assert.assertSame(entity , JSONPath.eval(entity, "[id = 1001]"));
Assert.assertNull(JSONPath.eval(entity, "[id = 1002]"));

JSONPath.set(entity, "id", 123456); //将id字段修改为123456
Assert.assertEquals(123456, entity.getId().intValue());

JSONPath.set(entity, "value", new int[0]); //将value字段赋值为长度为0的数组
JSONPath.arrayAdd(entity, "value", 1, 2, 3); //将value字段的数组添加元素1,2,3

```