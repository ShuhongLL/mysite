---
title: Memory Leaks in Serveral Commonly Used Programming Languages
date: 2019-05-05 15:37:22
tags: [C++, Java, Python, NodeJS]
photos: ["../images/Memory-Leaks.JPG"]
---
Usually when we talk about memory leak we are actually talking about the memory leak in the heap memory. That is when an object being initialized, it will be allocated to a piece of memory in the heap and can be manipulated, after we perform some operations and terminate the whole procedure, that piece of memory is not erased but held in the heap, marked as being occupied but no reference point to it.

Wiki's Def:
>**Memory** leak is a type of resource leak that occurs when a computer program incorrectly manages memory allocations in such a way that 
- memory which is no longer needed is not released
- an object is stored in memory but cannot be accessed by the running code

We usually encounter this issue in programming languages that don't have **GC** (Garbage Collector), for example C++ and C. For such languages, we have to manage the memory by ourselves which, if not done properly, will expose the risk of memory leak.
</br>

## This is really common in C++
Let's take a look in C++. There are literally hundreds of ways that can cause the memory leak and most of them won't be detected during compilation and even the runtime. Only a few leaks will not have any impact on the system; however if we are running a huge application and those leaks accumulate, that will significantly reduce the real runtime performance of the system.

We all know that when we allocate an object, we have to release the memory if this object is not used anymore. The way we release the memory is simply call the buid-in function ***free( )*** or ***delete[ ]***. However in C++ the procedure can exit anywhere. An exception can be thrown in the half way so that the code doesn't ever reach the line to release memory:
```c++
int sample( int n ) {
    void  *ptr = malloc( 16 );
    if ( n )
        return -1; //memory leak here
    free( ptr );
    return 0;
}
```
or:
```c++
class Sample {
    public:
        init( ) { int *n = new int;  throw any_exception( ); }
        ~init( ) { delete n; }
    private:
        int *n;
};
Sample *n = new Sample; //memory leak here
```
The solution to the above examples is also really simple: check control flows and **do remember to call the destructor before anywhere the procedure may exit**. Well if you want to do it in a fancy way, you can use ***smart pointer*** alternatively:
```c++
class Sample {
    public:
        init( ) { n = std::make_shared<\int>( new int ) }
        ~init( ) {}
    private:
        std::shared_ptr<\int> n;
};
```
Smart pointer helps you manage this object and if it is not referred anymore, release its memory.
</br>

## Can free( )/delete release everything in memory?
You may think your program has a really concrete and strict control flow and you are so confident that **free( )** or **delete** is called before the procedure exits but does it release all the unrefered memory? **No! it is able to release the memory where the pointer is currently pointing to but not the pointer itself!** The pointer will still point to the original memory address but the content has been already removed. In this circumstance, the value of the pointer does not equal to **NULL**, instead some random values that cannot be predicted.
```c++
int main( ) {
    char *p = ( char* ) malloc( sizeof( char ) * 100 );
    strcpy( p, "hello" );
    free( p );
    if ( p != NULL ) //doesn't prevent issue
        strcpy( p, "world" ); //throw error
}
```
This pointer p is called ***wild pointer*** and will only be erased after the whole procedure is finished or terminated. The wild pointer is really risky because of its random behavior. To prevent it, **always set the pointer to be NULL when it is not used/the memory it points to is released**.

***Note***: when you define a pointer without setting up its initial value, that pointer will also be a ***wild pointer*** and has a value of some random number (which doesn't equal to **NULL**). Hence it is necessary to set the value of a pointer to be **NULL** if it cannot be asigned a value at the beginning.

For some simple pointers, they can be reasigned to **NULL** to prevent ***wild pointer***, however for a pointer referring to a hierarchical object, simply setting to **NULL** cannot resolve the potential issues. For example, you are using a ***vector*** in C++ :
```c++
vector <\string> v
int main( ) {
    for ( int i=0; i<1000000; i++ )
        v.push_back( "test" );
    
    cout << v.capacity( ) << endl;  //memory usage: 54M
    v.clear( );
    cout << v.capacity( ) << endl;  //memory usage: still 54M
}
```
Even though we have cleared the vector and all its elements were indeed released, the capacity of the vector is still unchanged. **clear( )** removed all its element but cannot shrink the size of the container which has already been allocated. The same thing happens to other containers such as **deque**. To handle that, before **C++ 11**, we can swap the pointer:
```c++
int main( ) {
    ...
    v.clear( );
    vector</string>(v).swap(v); //new a vector with the same content and swap
    cout << v.capacity( ) << endl;  //memory usage: 0
}
```
after C++ 11, it provides function **shrink_to_fit( )** to remove the extra allocated memory.
</br>

## GC doesn't avoid memory leaks
It's not suprising that GC can prevent most cases of memory leaks because it is runnig in an individual thread, checking the memory regularly and removing the unreferred objects. It is so powerful that porgrammers rarely pay attention to memory management and be aware of the memory leaks. **Java** is such language which has powerful and unruly GC that can be hardly controlled (call **System.gc( )** doesn't certainly invoke GC). It helps to manage the memory in jvm, but it cannot totally avoid memory leaks in Java.

There are mainly two cases that can lead to memory leaks in Java. One is the object which has a longer lifecycle keeps a reference to another object which has a shorter lifecycle:
```java
public class Sample {
    Object object;
    public void anymethod( ){
        object = new Object( );
        ...
    }
    ...
}
```
If ***object*** is only used inside ***anymethod( )***, then after stack pops ***anymethod( )***, the lifecycle of ***object*** should also be ended. But for here, because class ***Sample*** is still proceeding and it keeps the reference to ***object***, ***object*** cannot be collected by GC and hence leaks the memory. The solution will be either init ***object*** inside ***anymethod( )*** (as a local varible) or set ***object*** to be ***null*** after ***anymethod*** is finished.

Another case is the use of ***HashSet***. ***HashSet*** is the implement of hash-table and it stores elements according to their different hash values. In order to push and withdraw the sample object in the ***HashSet***, we need to override the method ***HashCode( )*** so that the same object has the same hash vaule and being stored in the same place in ***HashSet***. However, if we push something into the ***HashSet*** and then change some properties of this object (those properties are most likely to be used to calculate the hashcode), the hashcode of this object may vary and when we refer this object back to our ***HashSet*** to do some operations, for example delete this object from the ***HashSet***, this object might not be found in the set and hence cannot be deleted:
```java
    HashSet<Obejct> set = new HashSet<Object>( );
    Object something = new Object( );
    set.add( something );
    something.doSomethingChanges( );
    set.contains( something );  //this may return false
    set.remove( something );  //something cannot be removed if the previous line returns false
```
</br>

##Python




