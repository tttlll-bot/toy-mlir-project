# Toy MLIR 项目

跟随 MLIR 官方 Toy 教程，从零实现一个基于 MLIR 的编译器。

## 构建

```bash
mkdir build && cd build
cmake -G Ninja .. \
    -DMLIR_DIR=$MLIR_DIR \
    -DLLVM_DIR=$LLVM_DIR
ninja
