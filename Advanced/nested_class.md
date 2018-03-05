# <center>Nested Class</center>



<br></br>

Nested classes are further divided into two types: static nested classes and Java inner class.

Nested classes enable you to logically group classes that are only used in one place, increase the use of encapsulation, and create more readable and maintainable code.

```java
class Outter {
    private int age = 12;
      
    class Inner {
        private int age = 13;
        public void print() {
            System.out.println("内部类变量：" + this.age);
            System.out.println("外部类变量：" + Outter.this.age);
        }
    }
}
```

<br></br>



## Static Nested Class
----
If the nested class is static, then it’s called static nested class. 

**Static nested classes can access only static members of the outer class. Static nested class is same as any other top-level class and is nested for only packaging convenience.**

```java
public class OuterClass {
    private String sex;
    public static String name = "chenssy";
    
    /**
     *静态内部类
     */
    static class InnerClass1{
        /* 在静态内部类中可以存在静态成员 */
        public static String _name1 = "chenssy_static";
        
        public void display(){
            /* 
             * 静态内部类只能访问外围类的静态成员变量和方法
             * 不能访问外围类的非静态成员变量和方法
             */
            System.out.println("OutClass name :" + name);
        }
    }
    
    /**
     * 非静态内部类
     */
    class InnerClass2{
        /* 非静态内部类中不能存在静态成员 */
        public String _name2 = "chenssy_inner";
        /* 非静态内部类中可以调用外围类的任何成员,不管是静态的还是非静态的 */
        public void display(){
            System.out.println("OuterClass name：" + name);
        }
    }
    
    public void display(){
        /* 外围类访问静态内部类：内部类. */
        System.out.println(InnerClass1._name1);
        /* 静态内部类 可以直接创建实例不需要依赖于外围类 */
        new InnerClass1().display();
        
        /* 非静态内部的创建需要依赖于外围类 */
        OuterClass.InnerClass2 inner2 = new OuterClass().new InnerClass2();
        /* 方位非静态内部类的成员需要使用非静态内部类的实例 */
        System.out.println(inner2._name2);
        inner2.display();
    }
    
    public static void main(String[] args) {
        OuterClass outer = new OuterClass();
        outer.display();
    }
}
```

Output:
```
chenssy_static
OutClass name :chenssy
chenssy_inner
OuterClass name：chenssy
```

<br></br>



## Java Inner Class
----
Any non-static nested class is known as inner class. 

**Inner classes can access all the variables and methods of the outer class. Since inner classes are associated with instance, we can’t have any static variables in them.** Object of inner class are part of the outer class object and to create an instance of inner class, we first need to create instance of outer class.

There are two special kinds of java inner classes.
1. 局部内部类local inner class 
2. 匿名内部类anonymous inner class

<br>


### Local Inner Class
If a class is defined in a method body, it’s known as local inner class. 

**Since local inner class is not associated with Object, we can’t use private, public or protected access modifiers with it. The only allowed modifiers are abstract or final. A local inner class can access all the members of the enclosing class and local final variables in the scope it’s defined.**

定义在方法里：
```java
public class Parcel5 {
    public Destionation destionation(String str){
        class PDestionation implements Destionation{
            private String label;
            private PDestionation(String whereTo){
                label = whereTo;
            }
            public String readLabel(){
                return label;
            }
        }
        return new PDestionation(str);
    }
    
    public static void main(String[] args) {
        Parcel5 parcel5 = new Parcel5();
        Destionation d = parcel5.destionation("chenssy");
    }
}
```

定义在作用域内:
```java
public class Parcel6 {
    private void internalTracking(boolean b){
        if(b){
            class TrackingSlip{
                private String id;
                TrackingSlip(String s) {
                    id = s;
                }
                String getSlip(){
                    return id;
                }
            }
            TrackingSlip ts = new TrackingSlip("chenssy");
            String string = ts.getSlip();
        }
    }
    
    public void track(){
        internalTracking(true);
    }
    
    public static void main(String[] args) {
        Parcel6 parcel6 = new Parcel6();
        parcel6.track();
    }
}
```


### Anonymous Inner Class
A local inner class without name is known as anonymous inner class. 

**Anonymous inner class always extend a class or implement an interface. Since an anonymous class has no name, it is not possible to define a constructor for an anonymous class. Anonymous inner classes are accessible only at the point where it is defined.**

```java
public class OuterClass {
    public InnerClass getInnerClass(final int num,String str2){
        return new InnerClass(){
            int number = num + 3;
            public int getNumber(){
                return number;
            }
        };  /* 注意：分号不能省 */
    }
    
    public static void main(String[] args) {
        OuterClass out = new OuterClass();
        InnerClass inner = out.getInnerClass(2, "chenssy");
        System.out.println(inner.getNumber());
    }
}

interface InnerClass {
    int getNumber();
}
```

注意：
1. 匿名内部类是没有访问修饰符的。
2. new匿名内部类，这个类首先是要存在的。如果将那个InnerClass接口注释掉，就会出现编译出错。
3. `getInnerClass()`方法的形参，第一个形参用final修饰，第二个没有。同时发现第二个形参在匿名内部类中没有使用过，所以当所在方法的形4. 匿名内部类没有构造方法。

<br></br>



## Benefits of Nested Class
----
1. If a class is useful to only one class, it makes sense to keep it nested and together. It helps in packaging of the classes.
2. Nested classes increases encapsulation. Note that inner classes can access outer class private members and at the same time we can hide inner class from outer world.
3. Nesting small classes within top-level classes places the code closer to where it is used and makes code more readable and maintainable.
4. 每个内部类都能独立地继承一个（接口的）实现，所以无论外围类是否已经继承了某个（接口的）实现，对于内部类都没有影响。

<br></br>



## Full Example
----

```java
public class OuterClass {
     
    private static String name = "OuterClass";
    private int i;
    protected int j;
    int k;
    public int l;
 
    //OuterClass constructor
    public OuterClass(int i, int j, int k, int l) {
        this.i = i;this.j = j;this.k = k;this.l = l;
    }
 
    public int getI() {
        return this.i;
    }
 
    //static nested class, can access OuterClass static variables/methods
    static class StaticNestedClass {
        private int a;
        protected int b;
        int c;
        public int d;
 
        public int getA() {
            return this.a;
        }
 
        public String getName() {
            return name;
        }
    }
 
 
    //inner class, non static and can access all the variables/methods of outer class
    class InnerClass {
        private int w;
        protected int x;
        int y;
        public int z;
 
        public int getW() {
            return this.w;
        }
 
        public void setValues() {
            this.w = i;this.x = j;this.y = k;this.z = l;
        }

        @Override
        public String toString() {
            return "w=" + w + ":x=" + x + ":y=" + y + ":z=" + z;
        }
 
        public String getName() {
            return name;
        }
    }
 
    //local inner class
    public void print(String initial) {
        //local inner class inside the method
        class Logger {
            String name;
 
            public Logger(String name) {
                this.name = name;
            }
 
            public void log(String str) {
                System.out.println(this.name + ": " + str);
            }
        }
 
        Logger logger = new Logger(initial);
        logger.log(name);
        logger.log("" + this.i);
        logger.log("" + this.j);
        logger.log("" + this.k);
        logger.log("" + this.l);
    }
 
    //anonymous inner class
    public String[] getFilesInDir(String dir, final String ext) {
        File file = new File(dir);
        //anonymous inner class implementing FilenameFilter interface
        String[] filesList = file.list(new FilenameFilter() {
            @Override
            public boolean accept(File dir, String name) {
                return name.endsWith(ext);
            }
        });
        return filesList;
    }
}
```

<br></br>