---
title: Slice in Golang
date: 2019-05-12 15:35:54
tags: Golang Array Slice
photos: ["../images/GoSlice.JPG"]
---
This article is a summary from [Andrew Gerrand's blog](https://blog.golang.org/go-slices-usage-and-internals)

Golang has an unique type **slice** which is an abstraction built on top of Go's **array** type. They are really similar but providing different means of working with sequences of typed data. So to understand slices we must first understand arrays.<!-- more -->
</br>

## Arrays in Go
An array in Go has to specify its **length** and **element type**. **The size of the array is fixed and its length is part of its type**. For example `[4]int` and `[5]int` are distinct and have different types even though they all store integers. And contrary to **C/C++**, the initial value of an array will be filled with **0** if it is not initialized.
```Go
var a [4]int
a[0] = 1
i := a[0]
j := a[1]
//i == 1
//j == 0
```
Go's arrays are values. **An array variable denotes the entire array**; it is not a pointer to the first array element (as would be the case in C/C++). This means that when you assign or pass around an array value you will make a copy of its contents. (To avoid the copy you could pass a pointer to the array, but then that's a pointer to an array, not an array)

An array literal can be specified like so:
```Go
b := [2]string{"aa", "bb"}
```
Or, you can have the compiler counting the array elements for you:
```Go
b := [...]string{"aa", "bb"}
```
In both cases, the type of b is **[2]string**.
</br>

## Slices in Go
Arrays are a bit inflexible, so you don't see them too often in the code. Slices, though, are everywhere. Unlike an array type, a slice type has no specific length:
```Go
b := []string{"aa", "bb"}
```
We can use build-in function `make()` to define a slice:
```Go
func make([]T, len, cap) []T
```
**T** represent the type of the elements. Function **make** accepts type, length and capacity(optional) as parameters. When it is called, **make** will allocate an array and returns a slice that refers to that array
```Go
var s []byte
s = make([]byte, 5, 5)
//s == []byte{0, 0, 0, 0, 0}
```
If **cap** is not specified, it will be init as the value of **len**. We can use the build-in functions `len()` and `cap()` to check the length and capacity of a slice:
```Go
len(s) == 5
cap(s) == 5
```
The zero value of a slice is **nil**. The len and cap functions will both return **0** for a nil slice.

A slice can also be formed by "slicing" an existing slice or array, for example, the expression b[1:4] creates a slice including elements 1 through 3 of b:
```Go
b := []byte{'a', 'b', 'c', 'd', 'e', 'f'}
// b[1:4] == []byte{'b', 'c', 'd'}, sharing the same storage as b
```
The start and end indices of a slice expression are optional; they default to zero and the slice's length respectively:
```Go
// b[:2] == []byte{'a', 'b'}
// b[2:] == []byte{'c', 'd', 'e', 'f'}
// b[:] == b
```
This is also the syntax to create a slice given an array:
```Go
x := [3]string{"Лайка", "Белка", "Стрелка"}
s := x[:] // a slice referencing the storage of x
```

Slicing does not copy the slice's data. It creates a new slice value that points to the original array. This makes slice operations as efficient as manipulating array indices. Therefore, modifying the elements of a re-slice modifies the elements of the original slice:
```Go
d := []byte{'a', 'b', 'c', 'd'}
e := d[2:]
// e == []byte{'c', 'd'}

// now change the re-slice will also change the original slice  
e[1] = 'm'
// e == []byte{'c', 'm'}
// d == []byte{'a', 'b', 'c', 'm'}
```
A slice cannot be grown beyond its capacity. Attempting to do so will cause a ***runtime panic***, just as when indexing outside the bounds of a slice or array. Similarly, slices cannot be re-sliced below zero to access earlier elements in the array.
</br>

## Double the capacity of a slice
To increase the capacity of a slice, we must create a new, larger slice and **copy** the contents of the original slice into it. The belowing example shows how to create a new slice **t** whihc doubles the capacity of **s**:
```Go
t := make([]byte, len(s), (cap(s) * 2))
for i:= range s {
    t[i] = s[i]
}
s = t   //reassign s to t
```
The loop can be replaced by the build-in function `copy()`, which copies the data from source and returns the number of elements copied:
```Go
func copy(dst, src []T) int
```
The function **copy** supports copying between slices of different lengths (it will copy only up to the smaller number of elements) and the case that two slices refer to the same array. Using **copy**, the above double size code snippet can be rewritten as:
```Go
t := make([]byte, len(s), (cap(s) * 2))
copy(t, s)
s = t
```
A common operation is to append new data to the tail of a slice:
```Go
func AppendByte(slice []byte, data ...type) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { //if the original capacity is not big enough     
        newSlice := make([]byte, (n + 1) * 2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n] //shrink the capacity to the length of data  
    copy(slice[m:n], data)
    return slice
}
```
This customized AppendByte function is really useful because we can fully control the size of a slice. However most programs do need such complete control. Go provides a build-in function `append()` which appends slice x to the end of slice s, expanding s if needed:
```Go
func append(s []T, x ...T) []T
```
Using **...** to append one slice to the end of another:
```Go
a := []string{"aa", "bb"}
b := []string{"cc", "dd"}
a = append(a, b...) //same as append(a, b[0], b[1], b[2])   
```
Another example of append:
```Go
func Filter(s []int, fn func(int) bool) []int {
    var p []int // p == nil
    for _, v := range s {
        if fn(v) {
            p = append(p, v)
        }
    }
    return p
}
```




