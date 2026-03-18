# 1.CUDA执行



## 1.存内存

### 1.存GPU

#### 1.cudaMalloc()

**在设备的全局内存中，分配指定大小的线性内存空间**



```c++
cudaError_t cudaMalloc(void** devPtr, size_t size);


```

* **`devPtr` (类型: `void**`)**
  * **含义：** 指向目标指针的指针。`cudaMalloc` 执行成功后，会把分配好的 GPU 内存的首地址写入到这个变量中。
  * **为什么是双重指针？** C/C++ 是值传递的。如果你只传一个普通的指针 `void* p`，`cudaMalloc` 内部只能修改 `p` 的副本，无法改变你外面的那个指针变量。为了让 `cudaMalloc` 能够真正修改你的指针，使其指向显存地址，你必须把**指针的地址**传进去。
  * **强制类型转换：** 因为它是 `void**`，所以在传入你自己定义的类型指针（比如 `float*` 或 `int*`）的地址时，通常需要加上 `(void**)` 进行强制类型转换。

* **`size` (类型: `size_t`)**
  * **含义：** 你想要分配的内存大小，单位是字节 (Bytes)。
  * **最佳实践：** 永远不要硬编码数字。应该使用 `元素个数 * sizeof(数据类型)` 的形式来计算，例如 `N * sizeof(float)`。

* **返回值 (Return Value)**
  * **类型：** `cudaError_t`（一个枚举类型）。
  * **作用：** 用于指示内存分配是否成功。
  * **常见结果：**
    * `cudaSuccess`：分配成功。
    * `cudaErrorMemoryAllocation`：显存不足，分配失败。
  * **注意：** 在编写严谨的 CUDA 代码时，**必须**检查这个返回值，否则一旦显存爆满导致分配失败，后续的 Kernel 启动和内存拷贝都会引发崩溃，且极难排查。





```c++
#define N 512

int main(){
    int *d_a;
    int size = N * sizeof(int)

    cudaMalloc((void**)&d_a,size);
    
    #用后一定要记得free
    cudaFree(d_a);
}
    
```







### 2.存CPU



#### 1.malloc()

向操作系统的**堆（Heap）**空间申请一块指定大小的连续内存区域。

```C++
void* malloc(size_t size);
```

**`malloc` 的三大核心特征**

1. **未初始化（包含“垃圾值”）：** `malloc` 仅仅是向操作系统“圈地”，它**不会**清空这块内存上的历史数据。如果你分配了内存后直接读取，会得到不可预测的随机值。如果需要分配并清零，应该使用 `calloc()`。
2. **堆区分配（Heap Allocation）：** 与函数内部声明的局部变量（分配在栈区 Stack，函数结束自动销毁）不同，`malloc` 分配的内存在堆区。它的生命周期由程序员完全掌控，直到你手动释放它。
3. **必须配对 `free()`：** 这是 C/C++ 系统编程中最容易踩坑的地方。使用 `malloc` 借来的内存，用完后必须调用 `free(ptr)` 归还给操作系统。否则会造成**内存泄漏（Memory Leak）**，长时间运行会导致程序耗尽系统内存而崩溃。





```C++

int main(){
    int *h_a;
    int size = N * sizeof(int);

    h_a= (int*)malloc(size);
 
    #用了一定要记得free
    free(h_a);
}
```





## 2.在GPU中运算



### **1.定义内核函数**

```c++
#该函数的内容将在gpu中被执行，cpu上被调用
__gobal__ void mykernel(void){

}
```





**CUDA 内置变量说明**

* **`threadIdx`**
  表示线程在其所在的线程块（thread block）中的索引。同一个线程块中的每个线程都会具有一个不同的索引。

* **`blockDim`**
  表示线程块的维度（尺寸），该维度是在启动内核（kernel）时的执行配置中指定的。

* **`blockIdx`**
  表示线程块在网格（grid）中的索引。网格中的每个线程块都会具有一个不同的索引。

* **`gridDim`**
  表示网格的维度（尺寸），该维度是在内核启动时的执行配置中指定的。









### 2.执行内核函数

```c++
int main(){
	
	int threadsPerBlock = 256; 
    // 设定 Grid 中的 Block 数（向上取整的标准公式）
    int blocksPerGrid = (numElements + threadsPerBlock - 1) / threadsPerBlock;
    // 3. 启动 Kernel
    mykernel<<<blocksPerGrid, threadsPerBlock>>>();
}
```







## 3.内存交换



### 1.cudaMemcpy()

```c++
cudaMemcpy(void* dst, const void* src, size_t count, cudaMemcpyKind kind);

#dst 目标指针
#src 被复制的指针
#count 拷贝内存大小，单位是字节（Bytes）
#cudaMemcpyDefault：根据传入指针的虚拟内存地址自动推断传输方向(前提是你的系统支持UVA)

		
```







```c++
#define N 512

int main(){
    int *d_a;
    int *h_a;
    int size = N * sizeof(int);

    h_a = (int*)malloc(size);    
    for(int i = 0;i<N;i++){
        h_a[i] = 0;
    }
    
    cudaMalloc((void**)&d_a,size);
    cudaMemcpy(d_a, h_a, size, cudaMemcpyHostToDevice);
    #cudaMemcpy(d_a, h_a, size, cudaMemcpyDefault);
    
    cudaFree(d_a);
    free(h_a);
}
    
```







# 2.GPU内存管理

















# 注意事项



## 1.gpu中的输出函数

在内核函数中，不可以使用c++自带的`cout`输出，因为gpu中未定义

要使用`printf`这个函数



并且要注意，很有可能这么写后可以运行，但是终端无输出，这是因为cpu不会等待gpu运行结束，而是接着继续运行

所以你需要在调用函数后使用`cudaDeviceSynchronize();`等待gpu上所有线程运行结束













