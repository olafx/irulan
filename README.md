# Irulan

**C++17 multidimensional array (tensor) data structure library.**

**This project is WIP!**

I wrote this library to make my numerical codes easier to write in CUDA/C++.

It provides tensor data structures, completely recursively defined, and of various layouts as to work in parallel with CBLAS, cuBLAS, LAPACK, etc.

What sets this library apart from other tensor libraries is twofold:
- Total control over data layout and what is and isn't compile time.
- No math.



## Types

### Static

Compile time size, direct storage (stack). (`std::array` equivalent.)

```C++
Static::Array<int[3][4][5][6]> A;
```

For regular dense tensors, stores a C-style array of lower order tensors.

```C++
Static::Array<int[3][4]> C = A(0, 0);
```

### Dynamic

Run time size. Compile time order. (`new type[size]` equivalent.)

```C++
Dynamic::Array<int[3]> A {3840, 3840, 3};
```

Dimensions are stored on stack in the object. Data is stored on heap.



## Property System

Properties are templated. They can be given in any order. All have implicit defaults.

```C++
Static:Array<int[3]> {};
Static:Array<int[3], Layout<packed>> {};
Static:Array<Layout<packed>, int[3], Allocate<false>> {};
```



## Access

### Indexing

Elements are accessed by Fortran-style bracket indexing (except 0-based).

```C++
Static::Tensor<float[3][4]> A;
A(0, 1) = 3;
```

If less indicides are given that the array's order, it will correctly interpret this and return for example the beginning of a column.

```C++
Static::Array<int[2][3]> A;
Dynamic::Array<int[2]> B {2, 3};

Static::Array<int[2]> A1 = A(1);
int *B1 = &B(1);

// B1 points to the 2nd column, not 2nd element, so A1 and B1 are consistent
```

### Dimensions

The square bracket operator is used to get dimensions.

```C++
Static:Array<int[3][4][5]> A;
A[1]; // 4
```

For dynamic tensors the dimension can also be set (i.e. the tensor can be reinterpreted).
It's designed to be used like this, but there is no safety, it will never reallocate.

```C++
Dynamic::Array<int[3]> A {16, 16, 3};
A[0] = 4; A[1] = 6; A[2] = 24;
// works fine since 4 * 6 * 24 <= 16 * 16 * 3
```

### Pointer

In the spirit of working in parallel with other libraries, a raw pointer can easily be returned.

```C++
#include <fftw3.h>
#include <vtkNew>
#include <vtkImageImport>
Dynamic::Array<fftw_complex[3]> in {512, 512, 256}, out {512, 512, 256};
auto plan = fftw_plan_dft_3d(in[0], in[1], in[2], in(), out(), FFTW_FORWARD, FFTW_ESTIMATE);
...
vtkNew<vtkImageImport> image_import;
image_import->SetWholeExtent(0, out[0] - 1, 0, out[1] - 1, 0, out[2] - 1);
image_import->SetNumberOfScalarComponents(in.order);
image_import->SetImportVoidPointer(static_cast<void *>(out()));
...
```

Dynamic tensors can be used to wrap existing data.

```C++
Dynamic::Array<float[3], Allocate<false>> A {256, 256, 256};
// A() == NULL
float *a = new float[A[0] * A[1] * A[2]];
A() = a;
Dynamic::Array<float[256][256][256], Allocate<false>> B {A()};
```



## Axis

Currently only column major storage is supported, and is implicit.

```C++
Dynamic::Array<float[2], Axis<column>> A {512, 512};
```



## Layouts

The goal is to ultimately support all tensor layouts by major linear algebra libraries.

### Conventional

The implicit, standard, dense layout.

```C++
Dynamic::Array<float[3], Layout<conventional>> A {3, 4, 5}; // size 60
```

### Packed

Packed storage, used in e.g. Hermitian matrices and triangular matrices. Only defined for symmetric storage.

*Partially implemented; size is correct and generalized to any number of dimensions, indexing is incomplete.*

```C++
Dynamic::Array<float[3], Layout<packed>> A {3}; // size 10
```



## Initializer Lists

Initializer lists are used not only for initialization, but also for assignment.

```C++
Static::Array<int[2][2][2]> A {{{1, 2}, {3, 4}}, {{5, 6}, {7, 8}}};
A(1, 1) = {-1, -2};
```

It's possible to make `constexpr` tensors in this way.

```C++
constexpr Static::Array<int[2][2]> B {{1, 2}, {3, 4}};
```
