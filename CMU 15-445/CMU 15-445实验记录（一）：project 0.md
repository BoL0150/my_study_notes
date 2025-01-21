# CMU 15-445实验记录（一）：project 0

要求实现一个矩阵和矩阵操作类

我们的代码必须遵守Google 的C++规范，使用Clang来自动检查代码样式，如果没有通过任何一项检查，项目成绩将为0 。

执行以下命令来检查代码规范和语法，`format` 目标将自动更正代码。 `check-lint` 和 `check-clang-tidy` 目标将打印错误并指导如何修复它以符合我们的风格指南。

```bash
$ make format
$ make check-lint
$ make check-clang-tidy
```

为了更好地debug，使用debug模式进行编译

```bash
$ cmake -DCMAKE_BUILD_TYPE=DEBUG ..
```

在编译目录下使用`make starter_test`对我们的代码进行测试，我们还可以运行 `make check-tests` 来运行所有测试用例。 由于我们还没有实现未来的项目，因此某些测试已被禁用。 我们可以通过向测试名称添加 `DISABLED_` 前缀来禁用 `GTest` 中的测试。

我们需要在Gradescope上提交实现，只需要在压缩文件中包括以下目录即可

- `src/include/primer/p0_starter.h`

```cpp
#include <memory>
#include <stdexcept>
#include <utility>
#include "../common/exception.h"
namespace bustub {

/*
 * The base class defining a Matrix
 */
template <typename T>
class Matrix {
 protected:
  // TODO(P0): Add implementation
  Matrix(int r, int c) : rows(r), cols(c) { linear = new T[rows * cols]; }

  // # of rows in the matrix
  int rows;
  // # of Columns in the matrix
  int cols;
  // Flattened array containing the elements of the matrix
  // TODO(P0) : Allocate the array in the constructor. Don't forget to free up
  // the array in the destructor.
  T *linear;

 public:
  // Return the # of rows in the matrix
  virtual int GetRows() = 0;

  // Return the # of columns in the matrix
  virtual int GetColumns() = 0;

  // Return the (i,j)th  matrix element
  virtual T GetElem(int i, int j) = 0;

  // Sets the (i,j)th  matrix element to val
  virtual void SetElem(int i, int j, T val) = 0;

  // Sets the matrix elements based on the array arr
  virtual void MatImport(T *arr) = 0;

  // TODO(P0): Add implementation
  virtual ~Matrix() { delete[] linear; }
};

template <typename T>
class RowMatrix : public Matrix<T> {
 public:
  // TODO(P0): Add implementation
  RowMatrix(int r, int c) : Matrix<T>(r, c) {
    data_ = new T *[r];
    for (int i = 0; i < r; i++) {
      data_[i] = new T[c];
    }
  }

  // TODO(P0): Add implementation
  int GetRows() override { return this->rows; }

  // TODO(P0): Add implementation
  int GetColumns() override { return this->cols; }

  // TODO(P0): Add implementation
  T GetElem(int i, int j) override {
    if (i >= this->rows || i < 0 || j >= this->cols || j < 0) {
      return 0;
    }
    return data_[i][j];
  }

  // TODO(P0): Add implementation
  void SetElem(int i, int j, T val) override {
    if (i >= this->rows || i < 0 || j >= this->cols || j < 0) {
      return;
    }
    data_[i][j] = val;
  }

  // TODO(P0): Add implementation
  void MatImport(T *arr) override {
    for (int i = 0; i < this->rows; i++) {
      for (int j = 0; j < this->cols; j++) {
        data_[i][j] = arr[i * this->cols + j];
      }
    }
    //   for (int i = 0; i < this->cols + this->rows; i++) {
    //     this->linear[i] = arr[i];
    //   }
  }

  // TODO(P0): Add implementation
  ~RowMatrix() override {
    for (int i = 0; i < this->rows; i++) {
      delete[] data_[i];
    }
    delete[] data_;
  }

 private:
  // 2D array containing the elements of the matrix in row-major format
  // TODO(P0): Allocate the array of row pointers in the constructor. Use these pointers
  // to point to corresponding elements of the 'linear' array.
  // Don't forget to free up the array in the destructor.
  T **data_;
};

template <typename T>
class RowMatrixOperations {
 public:
  // Compute (mat1 + mat2) and return the result.
  // Return nullptr if dimensions mismatch for input matrices.
  static std::unique_ptr<RowMatrix<T>> AddMatrices(std::unique_ptr<RowMatrix<T>> mat1,
                                                   std::unique_ptr<RowMatrix<T>> mat2) {
    // TODO(P0): Add code
    int a_row_cnt = mat1->GetRows();
    int b_row_cnt = mat2->GetRows();
    int a_col_cnt = mat1->GetColumns();
    int b_col_cnt = mat2->GetColumns();
    if (a_row_cnt != b_row_cnt || a_col_cnt != b_col_cnt) {
      return std::unique_ptr<RowMatrix<T>>(nullptr);
    }
    // 使用make_unique初始化unique_ptr,用其参数来构造给定类型的对象，所以
    // 所传递的参数必须与指向类型的某个构造函数相匹配
    std::unique_ptr<RowMatrix<T>> result = std::make_unique<RowMatrix<T>>(a_row_cnt, a_col_cnt);
    // 与下面的写法结果相同，但是在使用智能指针时要尽量避免使用new和delete
    // std::unique_ptr<RowMatrix<T>> res(new RowMatrix<T>(a_row_cnt,a_col_cnt));
    for (int i = 0; i < a_row_cnt; i++) {
      for (int j = 0; j < a_col_cnt; j++) {
        // 可以通过智能指针直接操作它所指向的对象
        result->SetElem(i, j, mat1->GetElem(i, j) + mat2->GetElem(i, j));
      }
    }
    // 由于unique_ptr不能拷贝，所以它不能作为函数参数传递，但是可以作为函数返回值,
    // 返回时调用的是move构造而不是拷贝构造
    return result;
  }

  // Compute matrix multiplication (mat1 * mat2) and return the result.
  // Return nullptr if dimensions mismatch for input matrices.
  static std::unique_ptr<RowMatrix<T>> MultiplyMatrices(std::unique_ptr<RowMatrix<T>> mat1,
                                                        std::unique_ptr<RowMatrix<T>> mat2) {
    // TODO(P0): Add code
    int a_row_cnt = mat1->GetRows();
    int b_row_cnt = mat2->GetRows();
    int a_col_cnt = mat1->GetColumns();
    int b_col_cnt = mat2->GetColumns();
    // 矩阵相乘要求a的列等于b的行
    if (a_col_cnt != b_row_cnt) {
      return std::unique_ptr<RowMatrix<T>>(nullptr);
    }
    // 生成矩阵的行和列分别为a的行和b的列
    std::unique_ptr<RowMatrix<T>> result = std::make_unique<RowMatrix<T>>(a_row_cnt, b_col_cnt);
    for (int i = 0; i < a_row_cnt; i++) {
      for (int j = 0; j < a_col_cnt; j++) {
        T temp = T();
        for (int a = 0; a < a_col_cnt; a++) {
          temp += mat1->GetElem(i, a) * mat2->GetElem(a, j);
        }
        // 可以通过智能指针直接操作它所指向的对象
        result->SetElem(i, j, temp);
      }
    }
    // 由于unique_ptr不能拷贝，所以它不能作为函数参数传递，但是可以作为函数返回值
    return result;
  }

  // Simplified GEMM (general matrix multiply) operation
  // Compute (matA * matB + matC). Return nullptr if dimensions mismatch for input matrices
  static std::unique_ptr<RowMatrix<T>> GemmMatrices(std::unique_ptr<RowMatrix<T>> matA,
                                                    std::unique_ptr<RowMatrix<T>> matB,
                                                    std::unique_ptr<RowMatrix<T>> matC) {
    // TODO(P2): Add code
    return AddMatrices(MultiplyMatrices(std::move(matA), std::move(matB)), std::move(matC));
  }
};
}  // namespace bustub
```

