# <center>String的equals()</center>

<br></br>



## Solution
----
```java
public class Test {
    private int num;
    private String data;

    public boolean equals(Object obj) {
        if(this == obj)
            return true;
        else if((obj == null) || (obj.getClass() != this.getClass())) 
            return false;
        
        // object must be Test at this point
        Test test = (Test)obj;

        return num == test.num &&
         (data == test.data || (data != null && data.equals(test.data)));
     }

     public int hashCode() {
         int hash = 7;
         hash = 31 * hash + num;
         hash = 31 * hash + (null == data ? 0 : data.hashCode());

         return hash;
     }
 }
```

<br></br>



## getClass() vs instanceof
----
```java
if((obj == null) || (obj.getClass() != this.getClass())) {
    return false;
}
```

This conditional check is preferred instead of the conditional check given by 

```java
if(!(obj instanceof Test)) {
    return false; 
}
```

Because the first condition ensures that it will return false if the argument is a subclass of the class Test. However, in case of the second condition it fails. Thus, it might violate the symmetry requirement of the contract. **The `instanceof` check is correct only if the class is final, so that no subclass would exist. The first condition will work for both, final and non-final classes.** 

* `instanceof` tests whether the object reference on the left-hand side (LHS) is an instance of the type on the right-hand side (RHS) or some subtype.
* `getClass() == ...` tests whether the types are identical.

```java
class A { }  
class B extends A { }  

Object o1 = new A();  
Object o2 = new B();  

o1 instanceof A => true  
o1 instanceof B => false  
o2 instanceof A => true // <================ HERE  
o2 instanceof B => true  

o1.getClass().equals(A.class) => true  
o1.getClass().equals(B.class) => false  
o2.getClass().equals(A.class) => false // <===============HERE  
o2.getClass().equals(B.class) => true
```

<br></br>



## Guidelines
----
1. Use `==` to check if the argument is the reference to this object. 

2. Cast the method argument to the correct type. Again, the correct type may not be the same class. Also, since this step is done after the above type-check condition, it will not result in a _ClassCastException_. 

3. Compare significant variables of both, the argument object and this object and check if they are equal. If all of them are equal then return true, otherwise return false. Again, as mentioned earlier, while comparing these class members/variables; *primitive variables* can be compared directly with `==` after performing any necessary conversions (such as float to Float.floatToIntBits or double to Double.doubleToLongBits). Whereas, *object references* can be compared by invoking their equals method recursively. Also need to ensure that invoking equals method on these object references does not result in a `NullPointerException`.

4. Do not change the type of the argument of the equals method. It takes a `java.lang.Object` as an argument, do not use your own class instead. If do that, you will not be overriding the equals method, but you will be overloading it instead; which would cause problems. 

5. Do not forget to override the hashCode method whenever you override the equals method.

6. The class `java.lang.StringBuffer` does not override the equals method, and hence it inherits the implementation from `java.lang.Object` class.

7. If a class field is not used in `equals()`, you should not use it in `hashCode()` method.

<br></br>



## Write Immutable Class
----
* All of its fields are `final` and the class is declared `final`
* Any fields that contain references to mutable objects, such as arrays, collections, or mutable classes like Date: 
    * Are private
    * Are never returned or otherwise exposed to callers
    * Are the only reference to the objects that they reference
    * Do not change the state of the referenced objects after construction

```java
class ImmutableArrayHolder {
  private final int[] theArray;

  // Right way to write a constructor -- copy the array
  public ImmutableArrayHolder(int[] anArray) {
    this.theArray = (int[]) anArray.clone();
  }

  // Wrong way to write a constructor -- copy the reference
  // The caller could change the array after the call to the constructor
  public ImmutableArrayHolder(int[] anArray) {
    this.theArray = anArray;
  }

  // Right way to write an accessor -- don't expose the array reference
  public int getArrayLength() { return theArray.length }
  public int getArray(int n)  { return theArray[n]; }

  // Right way to write an accessor -- use clone()
  public int[] getArray()     { return (int[]) theArray.clone(); }

  // Wrong way to write an accessor -- expose the array reference
  // A caller could get the array reference and then change the contents
  public int[] getArray()       { return theArray }
}
```