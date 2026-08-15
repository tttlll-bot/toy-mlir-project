# Chapter 1: 输入语言及语法树（事关parser 词法分析、语法分析）

[TOC]

## 那个模型语言

toys 是基于张量的语言，允许定义函数、运行一些数学计算并打印结果。

基于此，我们想让事情简单点，代码生成被限制于秩小于等于2的张量，并且唯一的数据类型就是64bit的浮点数类型（aka double in C），由此，所有的值都是隐式的double精度。
`Values`是不可变的（每个运算的返回值都是一个新分配的变量），变量的释放你不用管，是自动管理的。
行胜于言，跑通一个全流程demo好处多多，开始吧：

```toy
def main() {
  # 张量定义的示例：
  # 直接赋值多维矩阵的字面值就好。张量的形状会自动推导，不需给出。
  var a = [[1, 2, 3], [4, 5, 6]];

  # b 和 a 完全相同，一个指定形状一个隐式推导。注意区分view和内存概念，理解# a,b两者内存的存储结构完全一致，shape仅是视图上的改变，通过声明的shape信# 息修改遍历的步长，达到与多维数组等效。
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  #  transpose()  和  print()  是 Toy 语言仅有的两个内置函数——其他所有操作#（add、mul、reshape 等）都不是内置的，而是通过 Dialect/Operation 定义#  的。
  # 转置a,b;矩阵乘法；最后打印结果
  print(transpose(a) * transpose(b));
}
```
类型检查是通过类型推导实现，由编译器静态进行的。

toy语言只有在需要指定张量形状时，才进行类型声明（如作为输入输出参数）。

函数是泛型的（不指定参数即张量的维度）。在每次调用时，根据实际传入的参数签名，自动生成特化版本——类似 C++ 模板实例化。（也是静态的，特化由编译器完成）

接下来自行定义一个函数，添加到前例中

```toy
# 定义一个泛函数，对未知形状的张量做运算
def multiply_transpose(a, b) {
  return transpose(a) * transpose(b);
}

def main() {
  # Define a variable `a` with shape <2, 3>, initialized with the literal value.
  var a = [[1, 2, 3], [4, 5, 6]];
  var b<2, 3> = [1, 2, 3, 4, 5, 6];

  # 以下均为类型推导与函数特化的讲解，相同类型的泛型函数 特化成一个函数实例

  # This call will specialize `multiply_transpose` with <2, 3> for both
  # arguments and deduce a return type of <3, 2> in initialization of `c`.
  var c = multiply_transpose(a, b);

  # A second call to `multiply_transpose` with <2, 3> for both arguments will
  # reuse the previously specialized and inferred version and return <3, 2>.
  var d = multiply_transpose(b, a);

  # A new call with <3, 2> (instead of <2, 3>) for both dimensions will
  # trigger another specialization of `multiply_transpose`.
  var e = multiply_transpose(c, d);

  # Finally, calling into `multiply_transpose` with incompatible shapes
  # (<2, 3> and <3, 2>) will trigger a shape inference error.
  var f = multiply_transpose(a, c);
}
```

## The AST语法树

上述代码生成的AST相当直观简单，下面是它的dump(转储/打印输出)
The AST from the above code is fairly straightforward; here is a dump of it:

```
Module:
  Function 
    Proto 'multiply_transpose' @test/Examples/Toy/Ch1/ast.toy:4:1
    Params: [a, b]
    Block {
      Return
        BinOp: * @test/Examples/Toy/Ch1/ast.toy:5:25
          Call 'transpose' [ @test/Examples/Toy/Ch1/ast.toy:5:10
            var: a @test/Examples/Toy/Ch1/ast.toy:5:20
          ]
          Call 'transpose' [ @test/Examples/Toy/Ch1/ast.toy:5:25
            var: b @test/Examples/Toy/Ch1/ast.toy:5:35
          ]
    } // Block
  Function 
    Proto 'main' @test/Examples/Toy/Ch1/ast.toy:8:1
    Params: []
    Block {
      VarDecl a<> @test/Examples/Toy/Ch1/ast.toy:11:3
        Literal: <2, 3>[ <3>[ 1.000000e+00, 2.000000e+00, 3.000000e+00], <3>[ 4.000000e+00, 5.000000e+00, 6.000000e+00]] @test/Examples/Toy/Ch1/ast.toy:11:11
      VarDecl b<2, 3> @test/Examples/Toy/Ch1/ast.toy:15:3
        Literal: <6>[ 1.000000e+00, 2.000000e+00, 3.000000e+00, 4.000000e+00, 5.000000e+00, 6.000000e+00] @test/Examples/Toy/Ch1/ast.toy:15:17
      VarDecl c<> @test/Examples/Toy/Ch1/ast.toy:19:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:19:11
          var: a @test/Examples/Toy/Ch1/ast.toy:19:30
          var: b @test/Examples/Toy/Ch1/ast.toy:19:33
        ]
      VarDecl d<> @test/Examples/Toy/Ch1/ast.toy:22:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:22:11
          var: b @test/Examples/Toy/Ch1/ast.toy:22:30
          var: a @test/Examples/Toy/Ch1/ast.toy:22:33
        ]
      VarDecl e<> @test/Examples/Toy/Ch1/ast.toy:25:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:25:11
          var: c @test/Examples/Toy/Ch1/ast.toy:25:30
          var: d @test/Examples/Toy/Ch1/ast.toy:25:33
        ]
      VarDecl f<> @test/Examples/Toy/Ch1/ast.toy:28:3
        Call 'multiply_transpose' [ @test/Examples/Toy/Ch1/ast.toy:28:11
          var: a @test/Examples/Toy/Ch1/ast.toy:28:30
          var: c @test/Examples/Toy/Ch1/ast.toy:28:33
        ]
    } // Block
```

自行去编译去跑提供的example的demo

You can reproduce this result and play with the example in the
`examples/toy/Ch1/` directory; try running `path/to/BUILD/bin/toyc-ch1
test/Examples/Toy/Ch1/ast.toy -emit=ast`.

这里使用的lexer\parser代码相当直观简单，都只存在一个单一的头文件中

`examples/toy/Ch1/include/toy/Lexer.h`. The parser can be found in
`examples/toy/Ch1/include/toy/Parser.h`; it is a recursive descent parser. 

如果不熟悉，可以去读LLVM Kaleidoscope  前两章 tutorial
[Kaleidoscope Tutorial](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl02.html).

The [next chapter](Ch-2.md) will demonstrate how to convert this AST into MLIR.
