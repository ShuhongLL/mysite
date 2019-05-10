---
title: Memory Leaks in Serveral Commonly Used Programming Languages
date: 2019-05-05 15:37:22
tags: [C++, Java, Python, NodeJS]
photos: ["../images/Memory-Leaks.JPG"]
---
Usually when we talk about memory leaks we are actually talking about the memory leaks in heap memory. When an object is initialized, it will be dynamically allocated to a piece of memory in the heap and ready to be manipulated. After we perform some operations and the whole procedure is finished, the object stored in heap should also be erased; however in the case of memory leak, that piece of memory is not released but still held in the heap, marked as occupied but no reference refers to it.<!-- more -->

Wiki's Def:
>[**Memory leak**](https://en.wikipedia.org/wiki/Memory_leak) is a type of resource leak that occurs when a computer program incorrectly manages memory allocations in such a way that: 
- memory which is no longer needed is not released
- an object is stored in memory but cannot be accessed by the running code

We usually encounter this issue in programming languages that don't have [**GC**](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science), for example C++ and C. For such languages, we have to manage the memory by ourselves which, if not done properly, will expose the risks of memory leaks.
</br>

## This is really common in C++
Let's take a look in C++. There are literally hundreds of ways that can cause memory leaks and most of them won't be detected during compilation and even in runtime. Only a few leaks will not have any impact on the system; however if we are running a huge application and those leaks accumulate, that will significantly reduce the real runtime performance of the whole system.

We all know that when we allocate an object, we have to release the memory if this object is not used anymore. The way we release the memory is simply call the buid-in function **`free()`** or **`delete[]`**. However in C++ the procedure can exit anywhere. An exception can be thrown in the half way so that the code doesn't ever reach the line to release memory:
```c++
int sample(int n) {
    void  *ptr = malloc(16);
    if (n)
        return -1; //memory leak here
    free(ptr);
    return 0;
}
```
or:
```c++
class Sample {
    public:
        init() { int *n = new int;  throw any_exception(); }
        ~init() { delete n; }
    private:
        int *n;
};
Sample *n = new Sample; //memory leak here
```
The solution to the above examples is also really simple: check control flows and **do remember to call the destructor before anywhere the procedure may exit**. Well if you want to do it in a fancy way, you can use ***smart pointer*** alternatively:
```c++
class Sample {
    public:
        init() { n = std::make_shared<int>(new int) }
        ~init() {}
    private:
        std::shared_ptr<int> n;
};
```
Smart pointer helps you manage this object and if it is not referred anymore, release its memory.
</br>

## free( )/delete is not enough
Now your program has such a concrete control flow that **free( )** or **delete** is called before all the possible drop out. That is great but still not enough. **free( )** and **delete** can **only release the memory where the pointer is currently pointing to but not the pointer itself!** The pointer will still point to the original memory address but the content has been already removed. In this circumstance, the value of the pointer does not equal to **NULL**, instead some random values that cannot be predicted.
```c++
int main() {
    char *p = (char*) malloc(sizeof(char) * 100);
    strcpy(p, "hello");
    free(p);
    if (p != NULL) //doesn't prevent issue
        strcpy(p, "world"); // error
}
```
This pointer p is called [***dangling pointer***](https://en.wikipedia.org/wiki/Dangling_pointer) or [***wild pointer***](https://en.wikipedia.org/wiki/Dangling_pointer) and will only be erased after the whole procedure is finished or terminated. The wild pointer is really risky because of its random behavior. Imagine there is something in your room that sometimes can be observed sometimes cannot, randomly breaks your stuff but never leaves footprint. In programming it is called wild pointer, and in real life it is called [**cat**](https://en.wikipedia.org/wiki/Cat). To prevent it, we should **always set the pointer to be NULL when it is not used/the memory is released**.

***Note***: when you define a pointer without setting up its initial value, that pointer will also be a **wild pointer** and has a value of some random number (which doesn't equal to **NULL**). Hence it is necessary to set the value of a pointer to be **NULL** if it cannot be asigned a value at the beginning.

For some simple pointers, they can be reasigned to **NULL** to prevent **wild pointer**, however for a pointer referring to a hierarchical object, simply setting to **NULL** cannot resolve the potential issues. For example, you are using a **`vector`** in C++ :
```c++
vector <string> v
int main() {
    for (int i=0; i<1000000; i++)
        v.push_back("test");
    
    cout << v.capacity() << endl;  //memory usage: 54M
    v.clear();
    cout << v.capacity() << endl;  //memory usage: still 54M
}
```
Even though we have cleared the vector and all its elements were indeed released, the capacity of the vector is still unchanged. **`clear()`** removed all its element but cannot shrink the size of the container. The same thing happens to other containers such as **`deque`**. To handle this, before **C++ 11**, we can swap the pointers:
```c++
int main() {
    ...
    v.clear();
    vector<string>(v).swap(v); //new a vector with the same content and swap    
    cout << v.capacity() << endl;  //memory usage: 0
}
```
after C++ 11, it provides function **`shrink_to_fit()`** to remove the extra allocated memory.
</br>

## GC doesn't avoid memory leaks
It's not suprising that GC can prevent most cases of memory leaks because it is runnig in an individual thread, checking the memory regularly and removing the unreferred objects. It is so powerful that porgrammers rarely pay attention to memory management and be aware of the memory leaks. **Java** is such language which has powerful and unruly GC that can be hardly controlled (call **`System.gc()`** doesn't certainly invoke GC). It helps to manage the memory in jvm, but it cannot avoid memory leaks.

There are mainly two cases that can lead to memory leaks in Java. One is the object which has a longer lifecycle keeps a reference to another object which has a shorter lifecycle:
```java
public class Sample {
    Object object;
    public void anymethod(){
        object = new Object();
        ...
    }
    ...
}
```
If ***object*** is only used inside ***anymethod( )***, then after stack pops ***anymethod( )***, the lifecycle of ***object*** should also be ended. But for here, because class ***Sample*** is still proceeding and keeps the reference of ***object***, ***object*** cannot be collected by GC and hence leaks the memory. The solution will be either init ***object*** inside ***anymethod( )*** (as a local varible) or set ***object*** to be ***null*** after ***anymethod*** is finished.

Another case is the use of **`HashSet`**. ***HashSet*** is the implementation of hash-table and it stores elements according to their different hash values. In order to push and withdraw the same object in the ***HashSet***, we need to override the method **`HashCode()`** so that the same object has the same hash vaule and being stored in the same place in ***HashSet***. However, if we push something into the ***HashSet*** and then change some properties of this object (those properties are most likely to be used to calculate the hashcode), the hashcode of this object may vary and when we refer this object back to our ***HashSet*** to do some operations, for example delete this object from the ***HashSet***, this object might not be found in the set and hence cannot be deleted:
```java
    HashSet<Obejct> set = new HashSet<Object>();
    Object something = new Object();
    set.add(something);
    something.doSomethingChanges();
    set.contains(something);  //this may return false
    set.remove(something);  //'something' cannot be removed if the previous line returns false      
```
</br>

## Python

