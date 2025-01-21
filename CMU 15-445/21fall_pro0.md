```cpp
#pragma once

#include <memory>
#include <stdexcept>
#include <vector>

#include "../common/exception.h"

namespace bustub {

/**
 * The Matrix type defines a common
 * interface for matrix operations.
 */
template <typename T>
class Matrix {
 protected:
  /**
   * TODO(P0): Add implementation
   *
   * Construct a new Matrix instance.
   * @param rows The number of rows
   * @param cols The number of columns
   *
   */
  Matrix(int rows, int cols) : rows_(rows), cols_(cols) {
    // 初始化一维数组保存flatten matrix的值
    linear_ = new T[rows * cols];
  }

  /** The number of rows in the matrix */
  int rows_;
  /** The number of columns in the matrix */
  int cols_;

  /**
   * TODO(P0): Allocate the array in the constructor.
   * TODO(P0): Deallocate the array in the destructor.
   * A flattened array containing the elements of the matrix.
   */
  T *linear_;

 public:
  /** @return The number of rows in the matrix */
  virtual int GetRowCount() const = 0;

  /** @return The number of columns in the matrix */
  virtual int GetColumnCount() const = 0;

  /**
   * Get the (i,j)th matrix element.
   *
   * Throw OUT_OF_RANGE if either index is out of range.
   *
   * @param i The row index
   * @param j The column index
   * @return The (i,j)th matrix element
   * @throws OUT_OF_RANGE if either index is out of range
   */
  virtual T GetElement(int i, int j) const = 0;

  /**
   * Set the (i,j)th matrix element.
   *
   * Throw OUT_OF_RANGE if either index is out of range.
   *
   * @param i The row index
   * @param j The column index
   * @param val The value to insert
   * @throws OUT_OF_RANGE if either index is out of range
   */
  virtual void SetElement(int i, int j, T val) = 0;

  /**
   * Fill the elements of the matrix from `source`.
   *
   * Throw OUT_OF_RANGE in the event that `source`
   * does not contain the required number of elements.
   *
   * @param source The source container
   * @throws OUT_OF_RANGE if `source` is incorrect size
   */
  virtual void FillFrom(const std::vector<T> &source) = 0;

  /**
   * Destroy a matrix instance.
   * TODO(P0): Add implementation
   */
  virtual ~Matrix() { delete[] linear_; }
};

/**
 * The RowMatrix type is a concrete matrix implementation.
 * It implements the interface defined by the Matrix type.
 */

template <typename T>
class RowMatrix : public Matrix<T> {
 public:
  /**
   * TODO(P0): Add implementation
   *
   * Construct a new RowMatrix instance.
   * @param rows The number of rows
   * @param cols The number of columns
   */
  RowMatrix(int rows, int cols) : Matrix<T>(rows, cols) {
    data_ = new T *[rows];
    for (int i = 0; i < rows; i++) {
      data_[i] = new T[cols];
    }
  }

  /**
   * TODO(P0): Add implementation
   * @return The number of rows in the matrix
   */
  int GetRowCount() const override {
    // 此处必须要用this才行
    return this->rows_;
  }

  /**
   * TODO(P0): Add implementation
   * @return The number of columns in the matrix
   */
  int GetColumnCount() const override { return this->cols_; }

  /**
   * TODO(P0): Add implementation
   *
   * Get the (i,j)th matrix element.
   *
   * Throw OUT_OF_RANGE if either index is out of range.
   *
   * @param i The row index
   * @param j The column index
   * @return The (i,j)th matrix element
   * @throws OUT_OF_RANGE if either index is out of range
   */
  T GetElement(int i, int j) const override {
    if (i > this->rows_ || i < 1 || j > this->cols_ || j < 1) {
      throw Exception{"OUT_OF_RANGE"};
    }
    return data_[i - 1][j - 1];
  }

  /**
   * Set the (i,j)th matrix element.
   *
   * Throw OUT_OF_RANGE if either index is out of range.
   *
   * @param i The row index
   * @param j The column index
   * @param val The value to insert
   * @throws OUT_OF_RANGE if either index is out of range
   */
  void SetElement(int i, int j, T val) override {
    if (i > this->rows_ || i < 1 || j > this->cols_ || j < 1) {
      throw Exception{"OUT_OF_RANGE"};
    }
    data_[i - 1][j - 1] = val;
  }

  /**
   * TODO(P0): Add implementation
   *
   * Fill the elements of the matrix from `source`.
   *
   * Throw OUT_OF_RANGE in the event that `source`
   * does not contain the required number of elements.
   *
   * @param source The source container
   * @throws OUT_OF_RANGE if `source` is incorrect size
   */
  void FillFrom(const std::vector<T> &source) override {
    if (source.size() != static_cast<size_t>(this->cols_) * static_cast<size_t>(this->rows_)) {
      throw Exception{"OUT_OF_RANGE"};
    }
    for (int i = 0; i < this->rows_; i++) {
      for (int j = 0; j < this->cols_; j++) {
        data_[i][j] = source[i * this->cols_ + j];
      }
    }
  }

  /**
   * TODO(P0): Add implementation
   *
   * Destroy a RowMatrix instance.
   */
  ~RowMatrix() override {
    for (int i = 0; i < this->rows_; i++) {
      delete[] data_[i];
    }
    delete[] data_;
  };

 private:
  /**
   * A 2D array containing the elements of the matrix in row-major format.
   *
   * TODO(P0):
   * - Allocate the array of row pointers in the constructor.
   * - Use these pointers to point to corresponding elements of the `linear` array.
   * - Don't forget to deallocate the array in the destructor.
   */
  T **data_;
};

/**
 * The RowMatrixOperations class defines operations
 * that may be performed on instances of `RowMatrix`.
 */
template <typename T>
class RowMatrixOperations {
 public:
  /**
   * Compute (`matrixA` + `matrixB`) and return the result.
   * Return `nullptr` if dimensions mismatch for input matrices.
   * @param matrixA Input matrix
   * @param matrixB Input matrix
   * @return The result of matrix addition
   */
  static std::unique_ptr<RowMatrix<T>> Add(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB) {
    // TODO(P0): Add implementation
    int a_row_cnt = matrixA->GetRowCount();
    int b_row_cnt = matrixB->GetRowCount();
    int a_col_cnt = matrixA->GetColumnCount();
    int b_col_cnt = matrixB->GetColumnCount();
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
        result->SetElement(i + 1, j + 1, matrixA->GetElement(i + 1, j + 1) + matrixB->GetElement(i + 1, j + 1));
      }
    }
    // 由于unique_ptr不能拷贝，所以它不能作为函数参数传递，但是可以作为函数返回值,
    // 返回时调用的是move构造而不是拷贝构造
    return result;
  }

  /**
   * Compute the matrix multiplication (`matrixA` * `matrixB` and return the result.
   * Return `nullptr` if dimensions mismatch for input matrices.
   * @param matrixA Input matrix
   * @param matrixB Input matrix
   * @return The result of matrix multiplication
   */
  static std::unique_ptr<RowMatrix<T>> Multiply(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB) {
    // TODO(P0): Add implementation
    int a_row_cnt = matrixA->GetRowCount();
    int b_row_cnt = matrixB->GetRowCount();
    int a_col_cnt = matrixA->GetColumnCount();
    int b_col_cnt = matrixB->GetColumnCount();
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
          temp += matrixA->GetElement(i + 1, a + 1) * matrixB->GetElement(a + 1, j + 1);
        }
        // 可以通过智能指针直接操作它所指向的对象
        result->SetElement(i + 1, j + 1, temp);
      }
    }
    // 由于unique_ptr不能拷贝，所以它不能作为函数参数传递，但是可以作为函数返回值
    return result;
  }

  /**
   * Simplified General Matrix Multiply operation. Compute (`matrixA` * `matrixB` + `matrixC`).
   * Return `nullptr` if dimensions mismatch for input matrices.
   * @param matrixA Input matrix
   * @param matrixB Input matrix
   * @param matrixC Input matrix
   * @return The result of general matrix multiply
   */
  static std::unique_ptr<RowMatrix<T>> GEMM(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB,
                                            const RowMatrix<T> *matrixC) {
    // TODO(P0): Add implementation
      return Add(Multiply(matrixA,matrixB).release(),matrixC);
  //  int a_row_cnt = matrixA->GetRowCount();
  //  int b_row_cnt = matrixB->GetRowCount();
  //  int a_col_cnt = matrixA->GetColumnCount();
  //  int b_col_cnt = matrixB->GetColumnCount();
  //  int c_row_cnt = matrixC->GetRowCount();
  //  int c_col_cnt = matrixC->GetColumnCount();
  //  // 矩阵相乘要求a的列等于b的行,矩阵相加要求a的行等于c的行，b的列等于c的列
  //  if (a_col_cnt != b_row_cnt || a_row_cnt != c_row_cnt || b_col_cnt != c_col_cnt) {
  //    return std::unique_ptr<RowMatrix<T>>(nullptr);
  //  }
  //  // 生成矩阵的行和列分别为a的行和b的列
  //  std::unique_ptr<RowMatrix<T>> result = std::make_unique<RowMatrix<T>>(a_row_cnt, b_col_cnt);
  //  for (int i = 0; i < a_row_cnt; i++) {
  //    for (int j = 0; j < a_col_cnt; j++) {
  //      T temp = T();
  //      for (int a = 0; a < a_col_cnt; a++) {
  //        temp += matrixA->GetElement(i + 1, a + 1) * matrixB->GetElement(a + 1, j + 1);
  //      }
  //      // 相加
  //      result->SetElement(i + 1, j + 1, temp + matrixC->GetElement(i + 1, j + 1));
  //    }
  //  }
  //  // 由于unique_ptr不能拷贝，所以它不能作为函数参数传递，但是可以作为函数返回值
  //  return result;
  //}
};
}  // namespace bustub
```

