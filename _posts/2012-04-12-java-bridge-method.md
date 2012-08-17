---
layout: post
title: "Java bridge method"
description: ""
category: 
tags: []
---

Much has been said about Java Generics. Almost every Java book has separate chapter describing this mechanism and giving good advices how to work with them to not get into some troubles. But still we can be surprised quite often. The Generics are really powerful feature in Java. Allow to write much more type safe code then before but are quite complex especially at the way how were implemented in Java. 

Let's take the following code:

    interface Stack<E> {
      void push(E element);
      E pop();
      boolean empty();
    }

    public class StringArrayStack implements Stack<String> {
      private String[] elements;
      private int position = 0;
  
      public StringArrayStack(int capacity) { this.elements = new String[capacity]; }

      public void push(String element) { this.elements[position++] = element; }   
      public String pop() { return this.elements[--position]; }
      public boolean empty() { return this.position == 0; } 
  
      public static void main(String[] args) {
        Stack stack = new StringArrayStack(10);
        stack.push("a"); stack.push("b"); stack.push("c");
      }
    }


Really basic Stack with its array based implementation. It may look fine at first glance but can be the source of tricky problem. To show it we need to "improve" the code by modifying the `void push(E element)` method to accept many elements:

    interface Stack<E> {
      void push(E... elements);
      E pop();
      boolean empty();
    }

    public class StringArrayStack implements Stack<String> {
      private String[] elements;
      private int position = 0;
  
      public StringArrayStack(int capacity) { this.elements = new String[capacity]; }

      public void push(String... elements) { 
        for (String element : elements) { this.elements[position++] = element; } 
      }
      public String pop() { return this.elements[--position]; }
      public boolean empty() { return this.position == 0; } 
  
      public static void main(String[] args) {
        Stack stack = new StringArrayStack(10);
        stack.push("a", "b", "c");
      }
    }

The change in the code is quite small and also looks valid but running it gives:

    Exception in thread "main" java.lang.ClassCastException: [Ljava.lang.Object; cannot be cast to [Ljava.lang.String;
      at StringArrayStack.push(StringArrayStack.java:1)
      at StringArrayStack.main(StringArrayStack.java:23)


Why? Some of you may know the good advise regarding generics and varargs saying that should not be used together. Also I deliberately haven't mentioned about the compiler warning which might be helpful. But let's pretend that we don't know the advise and we ignore the warring as often happens. How can we solve it? Where to start?

**Stacktrace**

Typical place to start in case of an exception. Unfortunately our stacktrace isn't really useful. It says that the problem exists at line 1 in `StringArrayStack` which can not be because it's only class declaration line. To solve it we need to go deeper.

**Going deeper**

Generics were added to Java in the way that it provides total interoperability between generic code and old libraries that use raw types. This approach is called <i>type erasure</i> which means that type information is available only at compile time and after that it's erased by the compiler. So to explain the problem we should check generated class files to see how our code was translated to the bytecode:

    javap -v Stack.class

    interface Stack<E extends java.lang.Object>
      SourceFile: "StringArrayStack.java"
      Signature: #16                          // <E:Ljava/lang/Object;>Ljava/lang/Object;
      minor version: 0
      major version: 51
      flags: ACC_INTERFACE, ACC_ABSTRACT
    Constant pool:
       #1 = Class              #2             //  Stack
       #2 = Utf8               Stack
       #3 = Class              #4             //  java/lang/Object
       #4 = Utf8               java/lang/Object
       #5 = Utf8               push
       #6 = Utf8               ([Ljava/lang/Object;)V
       #7 = Utf8               Signature
       #8 = Utf8               ([TE;)V
       #9 = Utf8               pop
      #10 = Utf8               ()Ljava/lang/Object;
      #11 = Utf8               ()TE;
      #12 = Utf8               empty
      #13 = Utf8               ()Z
      #14 = Utf8               SourceFile
      #15 = Utf8               StringArrayStack.java
      #16 = Utf8               <E:Ljava/lang/Object;>Ljava/lang/Object;
    {
      public abstract void push(E...);
        flags: ACC_PUBLIC, ACC_VARARGS, ACC_ABSTRACT
        Signature: #8                           // ([TE;)V

      public abstract E pop();
        flags: ACC_PUBLIC, ACC_ABSTRACT
        Signature: #11                          // ()TE;

      public abstract boolean empty();
        flags: ACC_PUBLIC, ACC_ABSTRACT
    }


    javap -v StringArrayStack.class

    public class StringArrayStack extends java.lang.Object implements Stack<java.lang.String>
      SourceFile: "StringArrayStack.java"
      Signature: #61                          // Ljava/lang/Object;LStack<Ljava/lang/String;>;
      minor version: 0
      major version: 51
      flags: ACC_PUBLIC, ACC_SUPER
    Constant pool:
       #1 = Class              #2             //  StringArrayStack
       #2 = Utf8               StringArrayStack
       #3 = Class              #4             //  java/lang/Object
       #4 = Utf8               java/lang/Object
       #5 = Class              #6             //  Stack
       #6 = Utf8               Stack
       #7 = Utf8               elements
       #8 = Utf8               [Ljava/lang/String;
       #9 = Utf8               position
      #10 = Utf8               I
      #11 = Utf8               <init>
      #12 = Utf8               (I)V
      #13 = Utf8               Code
      #14 = Methodref          #3.#15         //  java/lang/Object."<init>":()V
      #15 = NameAndType        #11:#16        //  "<init>":()V
      #16 = Utf8               ()V
      #17 = Fieldref           #1.#18         //  StringArrayStack.position:I
      #18 = NameAndType        #9:#10         //  position:I
      #19 = Class              #20            //  java/lang/String
      #20 = Utf8               java/lang/String
      #21 = Fieldref           #1.#22         //  StringArrayStack.elements:[Ljava/lang/String;
      #22 = NameAndType        #7:#8          //  elements:[Ljava/lang/String;
      #23 = Utf8               LineNumberTable
      #24 = Utf8               LocalVariableTable
      #25 = Utf8               this
      #26 = Utf8               LStringArrayStack;
      #27 = Utf8               capacity
      #28 = Utf8               push
      #29 = Utf8               ([Ljava/lang/String;)V
      #30 = Utf8               element
      #31 = Utf8               Ljava/lang/String;
      #32 = Utf8               StackMapTable
      #33 = Class              #8             //  "[Ljava/lang/String;"
      #34 = Utf8               pop
      #35 = Utf8               ()Ljava/lang/String;
      #36 = Utf8               empty
      #37 = Utf8               ()Z
      #38 = Utf8               main
      #39 = Methodref          #1.#40         //  StringArrayStack."<init>":(I)V
      #40 = NameAndType        #11:#12        //  "<init>":(I)V
      #41 = String             #42            //  a
      #42 = Utf8               a
      #43 = String             #44            //  b
      #44 = Utf8               b
      #45 = String             #46            //  c
      #46 = Utf8               c
      #47 = InterfaceMethodref #5.#48         //  Stack.push:([Ljava/lang/Object;)V
      #48 = NameAndType        #28:#49        //  push:([Ljava/lang/Object;)V
      #49 = Utf8               ([Ljava/lang/Object;)V
      #50 = Utf8               args
      #51 = Utf8               stack
      #52 = Utf8               LStack;
      #53 = Utf8               ()Ljava/lang/Object;
      #54 = Methodref          #1.#55         //  StringArrayStack.pop:()Ljava/lang/String;
      #55 = NameAndType        #34:#35        //  pop:()Ljava/lang/String;
      #56 = Methodref          #1.#57         //  StringArrayStack.push:([Ljava/lang/String;)V
      #57 = NameAndType        #28:#29        //  push:([Ljava/lang/String;)V
      #58 = Utf8               SourceFile
      #59 = Utf8               StringArrayStack.java
      #60 = Utf8               Signature
      #61 = Utf8               Ljava/lang/Object;LStack<Ljava/lang/String;>;
    {
      public StringArrayStack(int);
        flags: ACC_PUBLIC
        Code:
          stack=2, locals=2, args_size=2
             0: aload_0       
             1: invokespecial #14                 // Method java/lang/Object."<init>":()V
             4: aload_0       
             5: iconst_0      
             6: putfield      #17                 // Field position:I
             9: aload_0       
            10: iload_1       
            11: anewarray     #19                 // class java/lang/String
            14: putfield      #21                 // Field elements:[Ljava/lang/String;
            17: return        
          LineNumberTable:
            line 13: 0
            line 11: 4
            line 13: 9
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                   0      18     0  this   LStringArrayStack;
                   0      18     1 capacity   I

      public void push(java.lang.String...);
        flags: ACC_PUBLIC, ACC_VARARGS
        Code:
          stack=5, locals=6, args_size=2
             0: aload_1       
             1: dup           
             2: astore        5
             4: arraylength   
             5: istore        4
             7: iconst_0      
             8: istore_3      
             9: goto          37
            12: aload         5
            14: iload_3       
            15: aaload        
            16: astore_2      
            17: aload_0       
            18: getfield      #21                 // Field elements:[Ljava/lang/String;
            21: aload_0       
            22: dup           
            23: getfield      #17                 // Field position:I
            26: dup_x1        
            27: iconst_1      
            28: iadd          
            29: putfield      #17                 // Field position:I
            32: aload_2       
            33: aastore       
            34: iinc          3, 1
            37: iload_3       
            38: iload         4
            40: if_icmplt     12
            43: return        
          LineNumberTable:
            line 16: 0
            line 17: 43
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                   0      44     0  this   LStringArrayStack;
                   0      44     1 elements   [Ljava/lang/String;
                  17      17     2 element   Ljava/lang/String;
          StackMapTable: number_of_entries = 2
               frame_type = 255 /* full_frame */
              offset_delta = 12
              locals = [ class StringArrayStack, class "[Ljava/lang/String;", top, int, int, class "[Ljava/lang/String;" ]
              stack = []
               frame_type = 24 /* same */


      public java.lang.String pop();
        flags: ACC_PUBLIC
        Code:
          stack=4, locals=1, args_size=1
             0: aload_0       
             1: getfield      #21                 // Field elements:[Ljava/lang/String;
             4: aload_0       
             5: dup           
             6: getfield      #17                 // Field position:I
             9: iconst_1      
            10: isub          
            11: dup_x1        
            12: putfield      #17                 // Field position:I
            15: aaload        
            16: areturn       
          LineNumberTable:
            line 18: 0
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                   0      17     0  this   LStringArrayStack;

      public boolean empty();
        flags: ACC_PUBLIC
        Code:
          stack=1, locals=1, args_size=1
             0: aload_0       
             1: getfield      #17                 // Field position:I
             4: ifne          9
             7: iconst_1      
             8: ireturn       
             9: iconst_0      
            10: ireturn       
          LineNumberTable:
            line 19: 0
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                   0      11     0  this   LStringArrayStack;
          StackMapTable: number_of_entries = 1
               frame_type = 9 /* same */


      public static void main(java.lang.String[]);
        flags: ACC_PUBLIC, ACC_STATIC
        Code:
          stack=5, locals=2, args_size=1
             0: new           #1                  // class StringArrayStack
             3: dup           
             4: bipush        10
             6: invokespecial #39                 // Method "<init>":(I)V
             9: astore_1      
            10: aload_1       
            11: iconst_3      
            12: anewarray     #3                  // class java/lang/Object
            15: dup           
            16: iconst_0      
            17: ldc           #41                 // String a
            19: aastore       
            20: dup           
            21: iconst_1      
            22: ldc           #43                 // String b
            24: aastore       
            25: dup           
            26: iconst_2      
            27: ldc           #45                 // String c
            29: aastore       
            30: invokeinterface #47,  2           // InterfaceMethod Stack.push:([Ljava/lang/Object;)V
            35: return        
          LineNumberTable:
            line 22: 0
            line 23: 10
            line 24: 35
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                   0      36     0  args   [Ljava/lang/String;
                  10      26     1 stack   LStack;

      public java.lang.Object pop();
        flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
        Code:
          stack=1, locals=1, args_size=1
             0: aload_0       
             1: invokevirtual #54                 // Method pop:()Ljava/lang/String;
             4: areturn       
          LineNumberTable:
            line 1: 0
          LocalVariableTable:
            Start  Length  Slot  Name   Signature

      public void push(java.lang.Object...);
        flags: ACC_PUBLIC, ACC_BRIDGE, ACC_VARARGS, ACC_SYNTHETIC
        Code:
          stack=2, locals=2, args_size=2
             0: aload_0       
             1: aload_1       
             2: checkcast     #33                 // class "[Ljava/lang/String;"
             5: invokevirtual #56                 // Method push:([Ljava/lang/String;)V
             8: return        
          LineNumberTable:
            line 1: 0
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
    }

A lot of information here. Description of the class file structure and Java bytecode is huge topic in itself but some details can be extracted quite easily.

The first listing just says that we have three methods in the interface. Not really informative for our case.

The second one is more interesting. There are following methods:

- `void push(java.lang.String...)`
- `java.lang.String pop()`
- `boolean empty()`
- `void main(java.lang.String[])`
- `java.lang.Object pop()`
- `void push(java.lang.Object...)`

We can see here that `push` and `pop` methods are duplicated. `push` is the one from our stacktrace so let's take it and look what does it do:

- `0: aload_0` - loads local method variable at index 0 (this reference) to the operand stack
- `1: aload_1` - loads local method variable at index 1 (`Object[]` array reference) to the operand stack
- `2: checkcast #33` - checks whether the object is of given type (`String[]`)
- `invokevirtual #56` - invokes `void push(java.lang.String...)` passing the array reference

In Java it can expressed like:

    public void push(Object[] elements) {
      push((String[]) elements);
    }

The method is called bridge method and is generated in our implementation because the class implements generic Stack interface with `void push(E...)` method. After type erasure while compiling the source code our E type is backed by plain `java.lang.Object` and the method declaration in fact is: `void push(Object...)`. And because our implementation class contains `void push(String...)` we need this bridge method which implements the interface method and forwards to our implementation. The `pop` method works the same way except casting step.

But we still don't know where does the exception come from? We need to check the main method. The most important is the following part:

    10: aload_1       
    11: iconst_3      
    12: anewarray     #3                  // class java/lang/Object
    15: dup           
    16: iconst_0      
    17: ldc           #41                 // String a
    19: aastore       
    20: dup           
    21: iconst_1      
    22: ldc           #43                 // String b
    24: aastore       
    25: dup           
    26: iconst_2      
    27: ldc           #45                 // String c
    29: aastore       


It creates a new array of size 3 and element type `java.lang.Object` and puts there our 3 strings.

The next line of this method:

    30: invokeinterface #47,  2           // InterfaceMethod Stack.push:([Ljava/lang/Object;)V

invokes the interface method which is implemented by our bridge method passing the reference to the array created in previous steps. As we know the array is of type `Object[]` and the bridge method casts it to `String[]`. Ups.

**The solution**

The problem can be easily solved just by not ignoring complier warning and creating our `StringArrayStack` in the safe way:

    Stack<String> stack = new StringArrayStack();

Which will change the generated bytecode for the main method at line 12 to:

    12: anewarray     #3                 // class java/lang/String

As we can see now the type of the array elements is String so the bridge method won't be even used.

But still the `void push(E...)` method is the source of potential problems. Let's "improve" the code a little more:

    interface Stack<E> {
      void push(E... elements);
      E pop();
      boolean empty();
    }

    abstract class AbstractStack<E> implements Stack<E> {
      public void push(List<E> list) {
        for (E element : list) { this.push(element); }
      }
    }

    public class StringArrayStack extends AbstractStack<String> {
      private String[] elements;
      private int position = 0;
  
      public StringArrayStack(int capacity) { this.elements = new String[capacity]; }

      public void push(String... elements) { 
        for (String element : elements) { this.elements[position++] = element; }
      }
      public String pop() { return this.elements[--position]; }
      public boolean empty() { return this.position == 0; } 
  
      public static void main(String[] args) {
        AbstractStack<String> stack = new StringArrayStack(10);
        stack.push("a", "b");
        stack.push(Arrays.asList("c", "d"));
      }
    }

The code is type safe but the `ClassCastException` is thrown. Why? Check the bytecode.

**Final notes**

- The compiler warnings especially in case of generics shouldn't be ignored. Otherwise, it's quite probable that we will be punished later (it's just postponed error).
- Reading Java bytecode doesn't hurt and can be really helpful.
- The way generics are implemented isn't an exception in Java. Java nested classes are also introduced on the compiler level - look how the access to private members of outer classes in done - and some other things like co-variant return types as well.