# <center>Enum</center>

<br></br>



创建枚举类并编译后，编译器会生成一个相关的类，这个类继承`java.lang.Enum`类。

利用javac编译_EnumDemo.java_后生成_Day.class_和_EnumDemo.class_。_Day.class_是枚举类型，即所说的使用关键字`enum`定义枚举类型并编译后，编译器自动生成与枚举相关的类。

```java
// 反编译Day.class
final class Day extends Enum{
   
    public static Day[] values() {  // 编译器添加的静态的values()方法
        return (Day[])$VALUES.clone();
    }

    // 编译器添加的静态的valueOf()方法，接调用Enum类的valueOf方法
    public static Day valueOf(String s) {
        return (Day)Enum.valueOf(com/zejian/enumdemo/Day, s);
    }

    private Day(String s, int i) { // 私有构造函数
        super(s, i);
    }

    // 定义的7种枚举实例
    public static final Day MONDAY;
    public static final Day TUESDAY;
    public static final Day WEDNESDAY;
    public static final Day THURSDAY;
    public static final Day FRIDAY;
    public static final Day SATURDAY;
    public static final Day SUNDAY;
    private static final Day $VALUES[];

    static { // 实例化枚举实例 
        MONDAY = new Day("MONDAY", 0);
        TUESDAY = new Day("TUESDAY", 1);
        WEDNESDAY = new Day("WEDNESDAY", 2);
        THURSDAY = new Day("THURSDAY", 3);
        FRIDAY = new Day("FRIDAY", 4);
        SATURDAY = new Day("SATURDAY", 5);
        SUNDAY = new Day("SUNDAY", 6);
        $VALUES = (new Day[] {
            MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
        });
    }
}
```

从反编译代码看出：
1. 枚举类型编译后转换成final类，且该类继承自`java.lang.Enum`抽象类。

2. 编译器帮助生成7个Day类型实例对象，分别对应枚举中定义的7个日期。即在该类中，存在每个在枚举类型中定义好变量的对应实例对象

3. 编译器生成两个静态方法，`values()`和`valueOf()`。

<br></br>



## 枚举常见方法
----
| 返回类型	| 方法名称                 |	方法说明 |
| ------- | ----------------------  | ---------  |
| int	   | compareTo(E o)      	| 比较枚举与指定对象顺序。 |
| boolean  |	equals(Object other) |	当指定对象等于枚举常量时，返回true。 |
| Class<?> |	getDeclaringClass()  |	返回与枚举常量的枚举类型对应Class对象。 |
| String   |	name()	            | 返回枚举常量名称。 |
| int	   | ordinal()	            | 返回枚举常量序数（在枚举声明中的位置，初始常量序数为0） |
| String   |	toString()          |	返回枚举常量的名称。 |
| static<T extends Enum<T>> | T	static valueOf(Class<T> enumType, String name) |	返回带指定名称的枚举类型的枚举常量。 |

* `compareTo(E o)`比较枚举的大小，其内部实现是根据每个枚举ordinal值比较。
* `name()`方法与`toString()`几乎等同。
* `valueOf(Class<T> enumType, String name)`方法则根据枚举类Class对象和枚举名称获取枚举常量，是静态的。

<br></br>



## Enum类内部代码
----
```java
// 实现Comparable
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable {
    private final int ordinal; // 枚举顺序值
    private final String name; // 枚举字符串名称

    public final String name() {
        return name;
    }
    
    public final int ordinal() {
        return ordinal;
    }
    
    protected Enum(String name, int ordinal) { // 构造方法，只能编译器调用
        this.name = name;
        this.ordinal = ordinal;
    }

    public String toString() {
        return name;
    }

    public final boolean equals(Object other) {
        return this==other;
    }

    
    public final int compareTo(E o) { // 比较ordinal值
        Enum<?> other = (Enum<?>)o;
        Enum<E> self = this;
        if (self.getClass() != other.getClass() && // optimization
            self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        return self.ordinal - other.ordinal;//根据ordinal值比较大小
    }

    @SuppressWarnings("unchecked")
    public final Class<E> getDeclaringClass() {
        Class<?> clazz = getClass(); // 获取class对象引用，getClass()是Object方法
        Class<?> zuper = clazz.getSuperclass(); // 获取父类Class对象引用
        return (zuper == Enum.class) ? (Class<E>)clazz : (Class<E>)zuper;
    }

    public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                                String name) {
        // enumType.enumConstantDirectory()获取map集合，key是name值，value是枚举变量值   
        // enumConstantDirectory是class内部方法，根据class对象获取map集合的值       
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }
}
```

`values()`方法和`valueOf(String name)`方法是编译器生成的static方法。在Enum类没出现`values()`方法，但`valueOf()`方法有出现，只不过编译器生成的`valueOf()`方法需传递name参数，而Enum自带静态方法`valueOf()`需传递两个方法。编译器生成的`valueOf()`方法最终调用Enum类`valueOf()`方法。

通过代码演示这两个方法作用：
```java
Day[] days2 = Day.values();
System.out.println("day2:"+Arrays.toString(days2));
Day day = Day.valueOf("MONDAY");
System.out.println("day:"+day);

/**
 输出结果:
 day2:[MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY]
 day:MONDAY
 */
```

* `values()`方法获取枚举类中所有变量，作为数组返回。
* `valueOf(String name)`方法与Enum类中`valueOf()`方法作用类似。只不过编译器生成的`valueOf()`方法更简洁些，只需传递一个参数。
* 由于`values()`方法由编译器插入到枚举类中的static方法，所以如果将枚举实例向上转型为Enum，那么`values()`方法无法被调用。因为Enum类中没有`values()`方法，`valueOf()`方法也是同样的道理，注意是一个参数的。

```java
Day[] ds=Day.values();  // 正常使用
Enum e = Day.MONDAY; // 向上转型Enum
e.values(); // 无法调用,没有此方法
```
