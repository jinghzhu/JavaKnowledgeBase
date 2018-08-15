# <center>String.equals()</center>



<br></br>

```java
public class Test {
    private int num;
    private String data;

    public boolean equals(Object obj) {
        if(this == obj)
            return true;
        
        if(obj == null || obj.getClass() != this.getClass()) 
            return false;
        
        Test test = (Test)obj; // object must be Test.

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



## Guidelines
----
1. Use `==` to check if the argument is the reference to this object. 

2. Cast the method argument to correct type. The correct type may not be the same class. Also, since this step is done after above type-check condition, it will not result in a _ClassCastException_. 

3. Compare significant variables of both, the argument object and this object and check if they are equal. If all of them are equal then return true. Again, as mentioned, while comparing these class members/variables; *primitive variables* can be compared directly with `==` after performing any necessary conversions. Whereas, *object references* can be compared by invoking their equals method recursively. Also need to ensure that invoking equals method on these object references does not result in a `NullPointerException`.

4. Do not change the type of argument of `equals()` method. It takes a `java.lang.Object` as an argument, do not use your own class instead. If do that, you will not be overriding `equals()` method. 

5. Do override `hashCode()` method whenever you override `equals()` method.

6. Class `java.lang.StringBuffer` does not override `equals()` method, and hence it inherits the implementation from `java.lang.Object` class.

<br></br>



## Immutable Class
----
* All of its fields are `final` and the class is declared `final`.

* Any fields that contain references to mutable objects, such as arrays, collections, or mutable classes like Date: 
    * private
    * never returned or otherwise exposed to callers
    * the only reference to the objects that they reference
    * not change the state of the referenced objects after construction

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