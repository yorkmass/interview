# Java面试

## 一、基础篇

### 1、能不能自己写一个类也叫Java.lang.String? 

<https://www.iteye.com/blog/rainlife-70072>

<https://www.cnblogs.com/guweiwei/p/6641785.html>



可以，但是即使你写了这个类，也没有用。

这个问题涉及到加载器的委托机制，在类加载器的结构图（在下面）中，BootStrap是顶层父类，ExtClassLoader是BootStrap类的子类，ExtClassLoader又是AppClassLoader的父类这里以java.lang.String为例，当我是使用到这个类时，Java虚拟机会将java.lang.String类的字节码加载到内存中。

为什么只加载系统通过的java.lang.String类而不加载用户自定义的java.lang.String类呢？

因加载某个类时，优先使用父类加载器加载需要使用的类。如果我们自定义了java.lang.String这个类，加载该自定义的String类，该自定义String类使用的加载器是AppClassLoader，根据优先使用父类加载器原理，AppClassLoader加载器的父类为ExtClassLoader，所以这时加载String使用的类加载器是ExtClassLoader，但是类加载器ExtClassLoader在jre/lib/ext目录下没有找到String.class类。然后使用ExtClassLoader父类的加载器BootStrap，父类加载器BootStrap在JRE/lib目录的rt.jar找到了String.class，将其加载到内存中。这就是类加载器的委托机制。

 所以，用户自定义的java.lang.String不被加载，也就是不会被使用。

 来自 <<https://www.cnblogs.com/guweiwei/p/6641785.html>> 

### 2、StringBuffer类的方法 

StringBuffer 方法

以下是 StringBuffer 类支持的主要方法：

| 序号 | 方法描述                                                     |
| ---- | ------------------------------------------------------------ |
| 1    | public StringBuffer append(String s)   将指定的字符串追加到此字符序列。 |
| 2    | public StringBuffer reverse()    将此字符序列用其反转形式取代。 |
| 3    | public delete(int start, int end)   移除此序列的子字符串中的字符。 |
| 4    | public insert(int offset, int i)   将 int 参数的字符串表示形式插入此序列中。 |
| 5    | replace(int start, int end, String str)   使用给定 String 中的字符替换此序列的子字符串中的字符。 |

下面的列表里的方法和 String 类的方法类似：

| 序号 | 方法描述                                                     |
| ---- | ------------------------------------------------------------ |
| 1    | int capacity()   返回当前容量。                              |
| 2    | char charAt(int index)   返回此序列中指定索引处的 char 值。  |
| 3    | void ensureCapacity(int minimumCapacity)   确保容量至少等于指定的最小值。 |
| 4    | void getChars(int srcBegin, int srcEnd, char[] dst, int   dstBegin)   将字符从此序列复制到目标字符数组 dst。 |
| 5    | int indexOf(String str)   返回第一次出现的指定子字符串在该字符串中的索引。 |
| 6    | int indexOf(String str, int fromIndex)   从指定的索引处开始，返回第一次出现的指定子字符串在该字符串中的索引。 |
| 7    | int lastIndexOf(String str)   返回最右边出现的指定子字符串在此字符串中的索引。 |
| 8    | int lastIndexOf(String str, int fromIndex)   返回 String 对象中子字符串最后出现的位置。 |
| 9    | int length()    返回长度（字符数）。                         |
| 10   | void setCharAt(int index, char ch)   将给定索引处的字符设置为 ch。 |
| 11   | void setLength(int newLength)   设置字符序列的长度。         |
| 12   | CharSequence subSequence(int start, int end)   返回一个新的字符序列，该字符序列是此序列的子序列。 |
| 13   | String substring(int start)   返回一个新的 String，它包含此字符序列当前所包含的字符子序列。 |
| 14   | String substring(int start, int end)   返回一个新的 String，它包含此序列当前所包含的字符子序列。 |
| 15   | String toString()   返回此序列中数据的字符串表示形式。       |

### 3、String Api 

下面是 String 类支持的方法，更多详细，参看 [Java String API](https://www.runoob.com/manual/jdk1.6/java/lang/String.html) 文档:

| SN(序号) | 方法描述                                                     |
| -------- | ------------------------------------------------------------ |
| 1        | [char   charAt(int index)](https://www.runoob.com/java/java-string-charat.html)   返回指定索引处的 char 值。 |
| 2        | [int   compareTo(Object o)](https://www.runoob.com/java/java-string-compareto.html)   把这个字符串和另一个对象比较。 |
| 3        | [int   compareTo(String anotherString)](https://www.runoob.com/java/java-string-compareto.html)   按字典顺序比较两个字符串。 |
| 4        | [int   compareToIgnoreCase(String str)](https://www.runoob.com/java/java-string-comparetoignorecase.html)   按字典顺序比较两个字符串，不考虑大小写。 |
| 5        | [String   concat(String str)](https://www.runoob.com/java/java-string-concat.html)   将指定字符串连接到此字符串的结尾。 |
| 6        | [boolean   contentEquals(StringBuffer sb)](https://www.runoob.com/java/java-string-contentequals.html)   当且仅当字符串与指定的StringBuffer有相同顺序的字符时候返回真。 |
| 7        | [static   String copyValueOf(char[\] data)](https://www.runoob.com/java/java-string-copyvalueof.html)   返回指定数组中表示该字符序列的 String。 |
| 8        | [static   String copyValueOf(char[\] data, int offset, int count)](https://www.runoob.com/java/java-string-copyvalueof.html)   返回指定数组中表示该字符序列的 String。 |
| 9        | [boolean   endsWith(String suffix)](https://www.runoob.com/java/java-string-endswith.html)   测试此字符串是否以指定的后缀结束。 |
| 10       | [boolean   equals(Object anObject)](https://www.runoob.com/java/java-string-equals.html)   将此字符串与指定的对象比较。 |
| 11       | [boolean   equalsIgnoreCase(String anotherString)](https://www.runoob.com/java/java-string-equalsignorecase.html)   将此 String 与另一个 String 比较，不考虑大小写。 |
| 12       | [byte[\]   getBytes()](https://www.runoob.com/java/java-string-getbytes.html)    使用平台的默认字符集将此 String 编码为 byte 序列，并将结果存储到一个新的 byte 数组中。 |
| 13       | [byte[\]   getBytes(String charsetName)](https://www.runoob.com/java/java-string-getbytes.html)   使用指定的字符集将此 String 编码为 byte 序列，并将结果存储到一个新的 byte 数组中。 |
| 14       | [void   getChars(int srcBegin, int srcEnd, char[\] dst, int dstBegin)](https://www.runoob.com/java/java-string-getchars.html)   将字符从此字符串复制到目标字符数组。 |
| 15       | [int   hashCode()](https://www.runoob.com/java/java-string-hashcode.html)   返回此字符串的哈希码。 |
| 16       | [int   indexOf(int ch)](https://www.runoob.com/java/java-string-indexof.html)   返回指定字符在此字符串中第一次出现处的索引。 |
| 17       | [int   indexOf(int ch, int fromIndex)](https://www.runoob.com/java/java-string-indexof.html)   返回在此字符串中第一次出现指定字符处的索引，从指定的索引开始搜索。 |
| 18       | [int   indexOf(String str)](https://www.runoob.com/java/java-string-indexof.html)    返回指定子字符串在此字符串中第一次出现处的索引。 |
| 19       | [int   indexOf(String str, int fromIndex)](https://www.runoob.com/java/java-string-indexof.html)   返回指定子字符串在此字符串中第一次出现处的索引，从指定的索引开始。 |
| 20       | [String   intern()](https://www.runoob.com/java/java-string-intern.html)    返回字符串对象的规范化表示形式。 |
| 21       | [int   lastIndexOf(int ch)](https://www.runoob.com/java/java-string-lastindexof.html)    返回指定字符在此字符串中最后一次出现处的索引。 |
| 22       | [int   lastIndexOf(int ch, int fromIndex)](https://www.runoob.com/java/java-string-lastindexof.html)   返回指定字符在此字符串中最后一次出现处的索引，从指定的索引处开始进行反向搜索。 |
| 23       | [int   lastIndexOf(String str)](https://www.runoob.com/java/java-string-lastindexof.html)   返回指定子字符串在此字符串中最右边出现处的索引。 |
| 24       | [int   lastIndexOf(String str, int fromIndex)](https://www.runoob.com/java/java-string-lastindexof.html)    返回指定子字符串在此字符串中最后一次出现处的索引，从指定的索引开始反向搜索。 |
| 25       | [int length()](https://www.runoob.com/java/java-string-length.html)   返回此字符串的长度。 |
| 26       | [boolean   matches(String regex)](https://www.runoob.com/java/java-string-matches.html)   告知此字符串是否匹配给定的正则表达式。 |
| 27       | [boolean   regionMatches(boolean ignoreCase, int toffset, String other, int ooffset, int   len)](https://www.runoob.com/java/java-string-regionmatches.html)   测试两个字符串区域是否相等。 |
| 28       | [boolean   regionMatches(int toffset, String other, int ooffset, int len)](https://www.runoob.com/java/java-string-regionmatches.html)   测试两个字符串区域是否相等。 |
| 29       | [String   replace(char oldChar, char newChar)](https://www.runoob.com/java/java-string-replace.html)   返回一个新的字符串，它是通过用 newChar 替换此字符串中出现的所有 oldChar 得到的。 |
| 30       | [String   replaceAll(String regex, String replacement)](https://www.runoob.com/java/java-string-replaceall.html)   使用给定的 replacement 替换此字符串所有匹配给定的正则表达式的子字符串。 |
| 31       | [String   replaceFirst(String regex, String replacement)](https://www.runoob.com/java/java-string-replacefirst.html)    使用给定的 replacement 替换此字符串匹配给定的正则表达式的第一个子字符串。 |
| 32       | [String[\]   split(String regex)](https://www.runoob.com/java/java-string-split.html)   根据给定正则表达式的匹配拆分此字符串。 |
| 33       | [String[\]   split(String regex, int limit)](https://www.runoob.com/java/java-string-split.html)   根据匹配给定的正则表达式来拆分此字符串。 |
| 34       | [boolean   startsWith(String prefix)](https://www.runoob.com/java/java-string-startswith.html)   测试此字符串是否以指定的前缀开始。 |
| 35       | [boolean   startsWith(String prefix, int toffset)](https://www.runoob.com/java/java-string-startswith.html)   测试此字符串从指定索引开始的子字符串是否以指定前缀开始。 |
| 36       | [CharSequence   subSequence(int beginIndex, int endIndex)](https://www.runoob.com/java/java-string-subsequence.html)    返回一个新的字符序列，它是此序列的一个子序列。 |
| 37       | [String   substring(int beginIndex)](https://www.runoob.com/java/java-string-substring.html)   返回一个新的字符串，它是此字符串的一个子字符串。 |
| 38       | [String   substring(int beginIndex, int endIndex)](https://www.runoob.com/java/java-string-substring.html)   返回一个新字符串，它是此字符串的一个子字符串。 |
| 39       | [char[\]   toCharArray()](https://www.runoob.com/java/java-string-tochararray.html)   将此字符串转换为一个新的字符数组。 |
| 40       | [String   toLowerCase()](https://www.runoob.com/java/java-string-tolowercase.html)   使用默认语言环境的规则将此 String 中的所有字符都转换为小写。 |
| 41       | [String   toLowerCase(Locale locale)](https://www.runoob.com/java/java-string-tolowercase.html)    使用给定 Locale 的规则将此 String 中的所有字符都转换为小写。 |
| 42       | [String   toString()](https://www.runoob.com/java/java-string-tostring.html)    返回此对象本身（它已经是一个字符串！）。 |
| 43       | [String   toUpperCase()](https://www.runoob.com/java/java-string-touppercase.html)   使用默认语言环境的规则将此 String 中的所有字符都转换为大写。 |
| 44       | [String   toUpperCase(Locale locale)](https://www.runoob.com/java/java-string-touppercase.html)   使用给定 Locale 的规则将此 String 中的所有字符都转换为大写。 |
| 45       | [String   trim()](https://www.runoob.com/java/java-string-trim.html)   返回字符串的副本，忽略前导空白和尾部空白。 |
| 46       | [static   String valueOf(primitive data type x)](https://www.runoob.com/java/java-string-valueof.html)   返回给定data type类型x参数的字符串表示形式。 |
| 47       | [contains(CharSequence   chars)](https://www.runoob.com/java/java-string-contains.html)   判断是否包含指定的字符系列。 |
| 48       | [isEmpty()](https://www.runoob.com/java/java-string-isempty.html)   判断字符串是否为空。 |

###  4、创建了几个String对象 

```java
String a="a";

String b=a+"b";

String c="ab";

String d="ab";

System.out.println(b==c);//不在缓存池false

System.out.println(c==d);//在缓存池，字符串直接量在缓存池中true

String f="a"+"b"+"c"+"d";//只创建了一个对象
```

 