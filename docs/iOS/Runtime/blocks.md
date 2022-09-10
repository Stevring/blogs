# OC Blocks All in One
## Intruduction
A Block is an **object** / **c-structure** / **function with data** that ecapsulate a piece of code and variables, and can be invoked / executed later.

Blocks are the building blocks for iOS multi-thread programming, and are extensively used in callbacks / delegation / \<maybe some others\>

Block syntax tip: http://fuckingblocksyntax.com/

## Tips for Using

1. avoid strong reference cycle
2. use `__block` modifiers in case if you want to modify a captured variable in a block.

## Anatomy of a Block

Take the simplest block as an example:

```Objective-C
void(^myBlock)(void) = ^(void) {
    NSLog(@"%d %d %d %d", local_nonstatic, local_static, global_nonstatic, global_static);
};
```

It is implemented with three structures: impl, func, desc.

### Impl
```c
struct __main_block_impl_1 {
  struct __block_impl impl;
  struct __main_block_desc_1* Desc;
  int local_nonstatic;
  int *local_static;
  __main_block_impl_1(void *fp, struct __main_block_desc_1 *desc, int _local_nonstatic, int *_local_static, int flags=0) : local_nonstatic(_local_nonstatic), local_static(_local_static) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
The `__main_block_impl_0` struct holds three things:  
1. `impl` (meta info, most important is the FuncPtr that points to the block's code), 
2.  `Desc` description of the block (see following), 
3.  external variables captured by the block (in this example, `local_nonstatic` and `local_static`)
    1.  auto variables are captured by value
    2.  local statics are captured by pointer
    3.  global statics and nonstatics are not captured, but directly referenced.

### Func

A static c function will be generated for this block and it encapsulated the actual code to execute.

```c
static void __main_block_func_1(struct __main_block_impl_1 *__cself) {
  int local_nonstatic = __cself->local_nonstatic; // bound by copy
  int *local_static = __cself->local_static; // bound by copy
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_qc_t2xr6bnn27nfywg43j28jtqc0000gp_T_main_5a80bd_mi_0, local_nonstatic, (*local_static), global_nonstatic, global_static);
}
```

### Desc 

The Desc struct describes the size and helper information (discussed later) of the Impl struct.

```c
static struct __main_block_desc_1 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_1_DATA = { 0, sizeof(struct __main_block_impl_1)};
```

## __block Modifier 

In the example above, the local variable is captured by value and that is why we cannot modify its value inside the block. In order to modify its value, we can add a `__block` modifier at its declaration, let's see what will be changed.

Code:

```c
1. dispatch_block_t getBlock() {
2.     static int local_static;
3.     __block int local_nonstatic;
4.     
5.     void(^myBlock)(void) = ^(void) {
6.         local_nonstatic = 10;
7.         NSLog(@"%d %d %d %d", local_nonstatic, local_static, global_nonstatic, global_static);
8.     };
9.     NSLog(@"%@", myBlock);
10.     return myBlock;
11. }
```

1. `local_nonstatic` in the 3rd line is initialized as a structure, not a simple integer.
    ```c
    struct __Block_byref_local_nonstatic_0 {
    void *__isa;
    __Block_byref_local_nonstatic_0 *__forwarding;
    int __flags;
    int __size;
    int local_nonstatic;
    };

    __Block_byref_local_nonstatic_0 local_nonstatic = {(void*)0,(__Block_byref_local_nonstatic_0 *)&local_nonstatic, 0, sizeof(__Block_byref_local_nonstatic_0)};
    ```

    the code for changing the value of `local_nonstatic` is
    ```c
    static void __getBlock_block_func_0(struct __getBlock_block_impl_0 *__cself) {
        __Block_byref_local_nonstatic_0 *local_nonstatic = __cself->local_nonstatic; // bound by ref
        (local_nonstatic->__forwarding->local_nonstatic) = 10;
    }
    ```
    We're using the `__forwarding` pointer to access the `local_nonstatic` field in the byref structure. `__forwarding` is initialized as address of the byref structure on the stack.
2. a `copy` and a `dispose` function pointer are added to the `Desc` structure

    ```c
    static void __getBlock_block_copy_0(struct __getBlock_block_impl_0*dst, struct __getBlock_block_impl_0*src) {_Block_object_assign((void*)&dst->local_nonstatic, (void*)src->local_nonstatic, 8/*BLOCK_FIELD_IS_BYREF*/);}

    static void __getBlock_block_dispose_0(struct __getBlock_block_impl_0*src) {_Block_object_dispose((void*)src->local_nonstatic, 8/*BLOCK_FIELD_IS_BYREF*/);}

    static struct __getBlock_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __getBlock_block_impl_0*, struct __getBlock_block_impl_0*);
    void (*dispose)(struct __getBlock_block_impl_0*);
    } __getBlock_block_desc_0_DATA = { 0, sizeof(struct __getBlock_block_impl_0), __getBlock_block_copy_0, __getBlock_block_dispose_0};
    ```

    `copy` and `dispose` are helper functions for the compiler to copy or release a block.

    When a block is copied to the heap, the byref structure is also copied to the heap. 
    - The `__forwarding` pointer of the original stack byref will from now on points to the byref structure on the heap.
    - The `__forwarding` pointer of the new heap byref will points to itself.
3. the `flags` field in the `Impl` are no longer 0:
   ```c
   void(*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &local_static, (__Block_byref_local_nonstatic_0 *)&local_nonstatic, 570425344));
   ```
   `570425344 = (1 << 25) | (1 << 29)`, from [apple's ABI](https://clang.llvm.org/docs/Block-ABI-Apple.html#imported-variables) specification:
   ```c
    enum {
        BLOCK_IS_NOESCAPE      =  (1 << 23),

        BLOCK_HAS_COPY_DISPOSE =  (1 << 25),
        BLOCK_HAS_CTOR =          (1 << 26), // helpers have C++ code
        BLOCK_IS_GLOBAL =         (1 << 28),
        BLOCK_HAS_STRET =         (1 << 29), // IFF BLOCK_HAS_SIGNATURE
        BLOCK_HAS_SIGNATURE =     (1 << 30),
    };
   ```
   It states the Impl struct as custom copy / dispose helper functions. Currently I do not understand the `BLOCK_HAS_STRET` flag.


## Types of Blocks

Summarized in https://www.jianshu.com/p/387730aec62a