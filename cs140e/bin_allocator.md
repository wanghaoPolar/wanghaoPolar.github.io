# bin allocator

本文对应的是 CS140e assignment 2 中的 [bin allocator](https://web.stanford.edu/class/cs140e/assignments/2-fs/#subphase-e-bin-allocator) 一节

## allocator

allocator 是在运行时动态分配 heap 上的内存的工具，rust 中的 allocator 相关方法的类型如下：

```rs
unsafe fn alloc(&mut self, layout: Layout) -> Result<*mut u8, AllocErr>;
unsafe fn dealloc(&mut self, ptr: *mut u8, layout: Layout);
```

在申请内存时调用 alloc 方法，调用方需要传入 layout。
在 free 对应的内存时，调用方需要传入之前返回的地址，以及申请时传入的 layout。

layout 类型的签名：

```rs
trait Layout {
  // 申请内存的大小
  fn size(&self) -> usize;
  // 申请内存的 alignment
  fn align(&self) -> usize;
}
```

### aligenment

比较有意思的是 align 这个属性，具体可以参考[这篇文章](https://www.ibm.com/developerworks/library/pa-dalign/)

总结下来，alignment 的意义是：

1. 内存在读取时并不是「真的」以 byte 为单位的，为了效率，不同的处理器会以 2-，4-，8-，16- 甚至 32-byte 为一个单位进行读取。
2. 如果内存读取的起始地址不是处理器的读取单位的倍数，会浪费一次读取。具体原因见![图](./doubleByteAccess.jpg)
3. 一些处理器甚至不支持读取 unaligned 的内存地址。比如早期的 [68000](https://everymac.com/systems/by_processor/68000.html) 会报错，最早的 Mac OS 不会处理这类报错，导致用户必须重启电脑。PowerPC 采取类似的策略，当读取 unaligend 的 float-point 时，会报错给操作系统，在软件层面进行读取。
4. 现代处理器的 [atomic instructions](http://faculty.ycp.edu/~dhovemey/spring2011/cs365/lecture/lecture20.html) 要求对应的地址至少要 four-byte aligned。否则在两次读取之间可能破坏 atomic instructions 的前置条件。
5. struct alignment： 在定义一个 struct 时，编译器可能会通过在不同字段间增加 padding 来保证各字段的地址都是 aligned 的，比如：

```c
void Munge64( void *data, uint32_t size ) {
typedef struct {
    char    a;
    long    b;
    char    c;
}   Struct;
```

实际的内存分配可能类似于：

| Field Type | Field Name | Field Offset | Field Size | Field End |
| ---------- | ---------- | ------------ | ---------- | --------- |
| char | a | 0 | 1 | 1 |
| | padding | 1 | 1 | 2 |
| long | b | 2 | 4 | 6 |
| char | c | 6 | 1 | 7 |
| | padding | 7 | 1 | 8 |

Total Size in Bytes: 8

### api 比较

对比一下 c 和 rust 在处理内存分配/回收时的方法：

rust

```rs
unsafe fn alloc(&mut self, layout: Layout) -> Result<*mut u8, AllocErr>;
unsafe fn dealloc(&mut self, ptr: *mut u8, layout: Layout);
```

c

```c
void *malloc(size_t size);
void free(void *pointer);
```

在 c 中，allocator 只保证在 32 位机器上使用 8-byte align，在 64 位机器上使用 16-byte align。（POSIX 方法 [posix_memalign](https://www.systutorials.com/docs/linux/man/3-posix_memalign/) 后来修正了这个行为）

在 rust 中，调用方可以主动声明 aligement（通过传入 layout）。

另外，在 rust 中 dealloc 时需要使用者传入申请内存时使用的 layout，Allocator 不需要记住相关状态。而在 c 中，「记住某个地址对应的值的大小和 aligement」是 Allocator 需要实现的。

rust 相较 c 更加灵活，能在更多的处理器上保证 align。另外 Allocator 的实现也更加简单。

## bin allocator

bin allocator 是课程中要求实现的两种 allocator 之一。（另一种，bump allocator，比较简单）

在 bin allocator 中，allocator 维护了一个 array，每个元素都是一个链表（通常称为一个 bin），bin 中的每个元素都是一个地址，这个地址指向一段可以分配的内存。一个 bin 中的所有内存的长度都是一致的。

通过合理规划每个 bin 对应的内存长度，allocator 可以满足不同大小的内存分配需求，同时尽可能地减少浪费。

通常的做法是，根据 bin 在 array 中的 index 来确定其内存长度，第 k 个 bin 对应的内存长度 = `2^(k + 3)`，例如

- bin 0(`2^3` bytes) 处理 `(0, 2^3]` 的分配请求
- bin 1(`2^4` bytes) 处理 `(2^3, 2^4]` 的分配请求
- bin k(`2^(k+3)` bytes) 处理 `(2^(k+2), 2^(k+3)]` 的分配请求

bin allocator 的工作方式如下

alloc：

1. 收到 alloc 请求时，在已有的 bin 中查找是否有满足条件的 block
2. 如果有，从 bin 中抽出 block，返回对应的地址（`addr: *mut u8`)
3. 如果没有，找到一个满足需求的最小的 bin，申请一片新的 block，返回对应的地址

dealloc：

1. 收到 dealloc 请求时，根据 addr 和 layout 查找到之前分配出去的是哪个 bin 里的 block。
2. 把该 block 重新插入到对应的 bin 里。

其中比较麻烦的一个问题是，由于需要 align，所以一个 block 对应的起始地址不一定等于最终返回给调用方的 addr

我使用的解决方案是在返回的 addr 之前存储一个 usize，等 dealloc 时直接从收到的 addr 之前读取 block 的起始地址。

当然，采取这个方案在计算分配 align 和 size 时需要对应的处理。

## linkedlist

还有一个比较有趣的东西是课程代码中提供的 linkedlist 这个数据结构。

rust 在处理 linkedlist 这类 recursive 的数据结构时比较麻烦，有[很多方案](http://cglab.ca/~abeinges/blah/too-many-lists/book/)。考虑到课程中直接面向底层，连 allocator 都要自己写，所以一些常用的方法就不能使用了。课程代码中提供了一个适用于当前情况的 linkedlist 实现，在 assignment 中叫做 `intrusive linked list`。

大概的工作方法就是

1. 用一个空指针 init 为 head。
2. 在 push 进新的地址时，会把之前的 head 的地址存在新的地址里，然后将新的地址设为 head。所以在多次 push 之后，linked list 里的每一个元素都存着之前一个元素的地址。
3. pop 的时候，直接读一下就能拿到之前的 head 了。
