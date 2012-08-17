---
layout: post
title: "Java array memory allocation"
description: ""
category: 
tags: []
---

There are two ways to create an array in Java:

- array creation expression: `int[] array = new int[5]`
- array initializer: `int[] array = {0,0,0,0,0}`

Both cases require the element type and the array length to be specified. These two values are needed to allocate appropriate memory structure for the array. Arrays in Java are just objects and theirs memory representation is quite similar:

![](http://4.bp.blogspot.com/-WeX8Ap6bQVM/T3xLiQrsUmI/AAAAAAAACPk/mvKb3tjOypk/s1600/JavaObjectStructure.png)

As we can see we have the header which contains 2 elements (more details later) and additional "size" element in case of arrays. After the header there are object fields or array elements placed. Anyway, this is still quite generic description. What if we want to get more detailed and real memory layout for our array? Is there a way to get it from the Java level?

**32 vs 64 bits**

Before trying to access the memory we need to know which platform we're going to use. The diagram below presents possible cases:

![](http://4.bp.blogspot.com/-yUbT_ye66Fk/T32hcRQL7xI/AAAAAAAACQU/el5taNPiQuY/s1600/JavaArrayMemory32vs64bit.png)

It's important to remember that all Java objects are aligned to an 8 byte granularity. Because of that we have additional paddings in some cases.

Because of it's simplicity 32-bit platform was chosen for the following examples.

**The Unsafe**

Java allocates and releases memory for our objects on it's own and hides memory addresses from us behind object references (in fact they are hidden pointers). But at the same time we have backdoor solution called `sun.misc.Unsafe` which allows to play directly with the memory.

Note: As the class name says it's really unsafe way of accessing memory and shouldn't be used in any production code.

Because of its private constructor we need a small trick to create an instance:

    try {
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      Unsafe unsafe = (Unsafe) field.get(null);
    } catch (Exception e) {}


Having the instance we can use the API which is quite rich e.g. we can read and write values at given addresses and even allocate and free real Java objects. For our purpose we need to just read the memory structure representing our array object, from the begin to the end. We don't know the address of given object but's it's not necessary for this case:

    try {
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      Unsafe unsafe = (Unsafe) field.get(null);

      int[] array = {1,2,3,4,5,6,7,8,9};
      long numberOfWords = 3 + 9;

      for (long i = 0; i < numberOfWords; i++) {
        int cur = unsafe.getInt(array, i * 4);
        System.out.println(Integer.toBinaryString(cur));
      }
    } catch (Exception e) {}
    
The output:

    1
    1100000000010000101011010000
    1001
    1
    10
    11
    100
    101
    110
    111
    1000
    1001

Each line represents 4-byte value:

- Line 1: Mark
- Line 2: `int[]` class object pointer
- Line 3: the array size
- Line 4-12: the array elements

**Multi-dimensional arrays**

Java implements multidimensional arrays as "arrays of array references":

![](http://2.bp.blogspot.com/-PDsnOr3i394/T3xL1DKvShI/AAAAAAAACPw/KZqqQp7vG8U/s1600/Java2DimArrayInMemory.png)

Printing the array:

    try {
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      Unsafe unsafe = (Unsafe) field.get(null);

      int[][] array = { {1,2,3},{4,5,6},{7,8,9} }
      long numberOfWords = 6 + 6 * 3;

      for (long i = 0; i < numberOfWords; i++) {
        int cur = unsafe.getInt(array, i * 4);
        System.out.println(Integer.toBinaryString(cur));
      }
    } catch (Exception e) {}

gives:

    1
    1011001011100011100111011000
    11
    101000100001110111100100000
    101000100001110111100111000
    101000100001110111101010000
    1
    1011000000010000101011010000
    11
    1
    10
    11
    1
    1011000000010000101011010000
    11
    100
    101
    110
    1
    1011000000010000101011010000
    11
    111
    1000
    1001


We have 4 arrays of the same size 3. The only difference is that the first array contains references/pointers to arrays with concrete int values.