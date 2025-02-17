---
title: "CLAD Project"
layout: gridlay
excerpt: "CLAD Project"
sitemap: false
permalink: /clad/
---

# Clad
Clad enables [automatic differentiation (AD)](https://en.wikipedia.org/wiki/Automatic_differentiation) for C++. It is based on LLVM compiler infrastructure and is a plugin for [Clang compiler](http://clang.llvm.org/). Clad is based on source code transformation. Given C++ source code of a mathematical function, it can automatically generate C++ code for computing derivatives of the function. It supports both forward-mode and reverse-mode AD.
## How to use Clad
Since Clad is a Clang plugin, it must be properly attached when Clang compiler is invoked. First, the plugin must be built to get `libclad.so` (or `.dylib`). To compile `SourceFile.cpp` with Clad enabled use:
```
clang -cc1 -x c++ -std=c++11 -load /full/path/to/lib/clad.so -plugin clad SourceFile.cpp
```
Clad provides four API functions:
- `clad::differentiate` to use forward-mode AD
- `clad::gradient` to use reverse-mode AD
- `clad::hessian` to compute Hessian matrix using a combination of forward-mode and reverse-mode AD
- `clad::jacobian` to compute Jacobian matrix using reverse-mode AD

API functions are used to label an existing function for differentiation. 
Both functions return a functor object containing the generated derivative which can be called via `.execute` method, which forwards provided arguments to the generated derivative function. Example:
```cpp
#include "clad/Differentiator/Differentiator.h"
#include <iostream>

double f(double x, double y) { return x * y; }

int main() {
  auto f_dx = clad::differentiate(f, "x");
  std::cout << f_dx.execute(3, 4) << std::endl; // prints: 4
  f_dx.dump(); // prints:
  /* double f_darg0(double x, double y) {
       double _d_x = 1; double _d_y = 0;
       return _d_x * y + x * _d_y;
     } */ 
}
```
### Forward mode
For a function `f` of several inputs and single (scalar) output, forward mode AD can be used to compute (or, in case of Clad, create a function) computing a directional derivative of `f` with respect to a *single* specified input variable. Derivative function created by the forward-mode AD is guaranteed to have *at most* a constant factor (around 2-3) more arithmetical operations compared to the original function.

`clad::differentiate(f, ARGS)` takes 2 arguments:
1. `f` is a pointer to a function or a method to be differentiated
2. `ARGS` is either: 
  * a single numerical literal indicating an index of independent variable (e.g. `0` for `x`, `1` for `y`)
  * a string literal with the name of independent variable (as stated in the *definition* of `f`, e.g. `"x"` or `"y"`)
  
Generated derivative function has the same signature as the original function `f`, however its return value is the value of the derivative.

### Reverse mode
When a function has many inputs and derivatives w.r.t. every input (i.e. gradient vector) are required, reverse-mode AD is a better alternative to the forward-mode. Reverse-mode AD allows to compute the gradient of `f` using *at most* a constant factor (around 4) more arithmetical operations compared to the original function. While its constant factor and memory overhead is higher than that of the forward-mode, it is independent of the number of inputs. E.g. for a function having N inputs and consisting of T arithmetical operations, computing its gradient takes a single execution of the reverse-mode AD and around 4\*T operations, while it would take N executions of the forward-mode, this requiring up to N\*3\*T operations.

`clad::gradient(f, /*optional*/ ARGS)` takes 1 or 2 arguments:
1. `f` is a pointer to a function or a method to be differentiated
2. `ARGS` is either: 
  * not provided, then `f` is differentiated w.r.t. its every argument
  * a string literal with comma-separated names of independent variables (e.g. `"x"` or `"y"` or `"x, y"` or `"y, x"`)
  
Since a vector of derivatives must be returned from a function generated by the reverse mode, its signature is slightly different. The generated function has `void` return type and same input arguments. The function has additional, last input argument of the type `T*`, where `T` is the return type of `f`. This is the "result" argument which has to point to the beginning of the vector where the gradient will be stored. *The caller is responsible for allocating and zeroing-out the gradient storage*. Example:
```cpp
auto f_grad = clad::gradient(f);
double result1[2] = {};
f_grad.execute(x, y, result1);
std::cout << "dx: " << result1[0] << ' ' << "dy: " << result1[1] << std::endl;

auto f_dx_dy = clad::gradient(f, "x, y"); // same effect as before

auto f_dy_dx = clad::gradient(f, "y, x");
double result2[2] = {};
f_dy_dx.execute(x, y, result2);
// note that the derivatives are mapped to the "result" indices in the same order as they were specified in the argument:
std::cout << "dy: " << result2[0] << ' ' << "dx: " << result2[1] << std::endl;
```
Note: *we are working on improving the gradient interface*.
## What can be differentiated
Clad is based on compile-time analysis and transformation of C++ abstract syntax tree (Clang AST). This means that Clad must be able to see the body of a function to differentiate it (e.g. if a function is defined in an external library there is no way for Clad to get its AST).

We aim to support every piece of modern C++ syntax, however at the moment only the a subset of C++ is supported and there are some constraints on functions that can be differentiated with Clad:
* Only builtin C++ scalar numeric types (e.g. `double`, `float`, `int`) are fully supported
* Differentiated functions must return a single value of supported scalar numeric type
* Differentiated functions can have arbitrary number or inputs of supported scalar numeric types
* Clad can also differentiate `struct`/`class` methods, however at the moment there is no way to differentiate them w.r.t. member fields

Note: *we are currently working on vector inputs*

The following subset of C++ syntax is supported at the moment:
* Numerical literals, builtin arithmetic operators `+`, `-`, `*`, `/`
* Variable declarations of supported types (including local variables in `{}` blocks)
* Inside functions, builtin arrays (e.g. `double x[1][2][3];`) of supported types and subscript operator `x[i]`
* Direct assignments to variables via `=` and `+=`, `-=`, `*=`, `/=`, `++`, `--`
* Conditional operator `?:` and boolean expressions
* Comma operator `,`
* Control flow: `if` statements and `for` loops (*work on loops in the reverse-mode is in progress*)
* Calls to other functions, including recursion

## Specifying custom derivatives
Sometimes Clad may be unable to differentiate your function (e.g. if its definition is in a library and source code is not available). Alternatively, an efficient/more numerically stable expression for derivatives may be know. In such cases, it is useful to be able to specify a custom derivatives for your function.

Clad supports that functionality by allowing to specify your own derivatives in `namespace custom_derivatives`. For a function named `FNAME` you can specify:
* a custom derivative w.r.t `I`-th argument by defining a function `FNAME_dargI` inside `namespace custom_derivatives`
* a custom gradient w.r.t every argument by defining a function `FNAME_grad` inside `namespace custom_derivatives`

When Clad will encounter a function `FNAME`, it will first do a lookup inside the `custom_derivatives` namespace to try to find a suitable custom function, and only if none is found will proceed to automatically derive it.

Example:
* Suppose that you have a function `my_pow(x, y)` which computes `x` to the power of `y`. However, Clad is not able to differentiate `my_pow`'s body (e.g. it calls an external library or uses some non-differentiable approximation):
```cpp
double my_pow(double x, double y) { // something non-differentiable here... }
```
However, you know analytical formulas of its derivatives, and you can easily specify custom derivatives:
```cpp
namespace custom_derivatives {
  double my_pow_darg0(double x, double y) { return y * my_pow(x, y - 1); }
  double my_pow_darg1(dobule x, double y) { return my_pow(x, y) * std::log(x); }
}
```
You can also specify a custom gradient:
```cpp 
namespace custom_derivatives {
  void my_pow_grad(double x, double y, double* result) { 
     double t = my_pow(x, y - 1);
     result[0] = y * t;
     result[1] = x * t * std::log(x);
   }
}
```
Whenever Clad will encounter `my_pow` inside differentiated function, it will find and use provided custom funtions instead of attempting to differentiate it.

Note: Clad provides custom derivatives for some mathematical functions from `<cmath>` inside `clad/Differentiator/BuiltinDerivatives.h`.

Note: *the concept of custom_derivatives will be reviewed soon, we intend to provide a different interface and avoid function name-based specifications and by-name lookups*.

## How Clad works
Clad is a plugin for the Clang compiler. It relies on the Clang to build the AST ([Clang AST](https://clang.llvm.org/docs/IntroductionToTheClangAST.html)) of user's source code. Then, [CladPlugin](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.h#L48), implemented as `clang::ASTConsumer` analyzes the AST to find differentiation requests for clad and process those requests by building Clang AST for derivative functions. The whole clad's operation sequence is the following:
* Clang parses user's source code and builds the AST.
* [`CladPlugin`](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L63) analyzes the built AST and starts the traversal via [`HandleTopLevelDecl`](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L67).
* [`DiffCollector`](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/lib/Differentiator/DiffPlanner.cpp#L141) does the [traversal](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L79) and looks at every call expression to find all the differentiation requests (through calls to `clad::differentiate(f, ...)` or `clad::gradient(f, ...)`).
* `CladPlugin` [processes](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L82) every found request.
* Processing each request involves calling to [`DerivativeBuilder::Derive`](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L113) which then [switches](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/lib/Differentiator/DerivativeBuilder.cpp#L77) do either [`ForwardModeVisitor::Derive`](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/lib/Differentiator/DerivativeBuilder.cpp#L407) or [`ReverseModeVisitor::Derive`](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/lib/Differentiator/DerivativeBuilder.cpp#L1506), depending on which AD mode was requested.
* `ForwardModeVisitor` and `ReverseModeVisitor` are derived from `clang::StmtVisitor`. In the `Derive` method they analyze the AST of the declaration of the original function and create the AST for the declaration of derivative function. Then they proceed to recursively `Visit` every Stmt in original function's body and build the body for the derivative function. Forward/Reverse mode AD algorithm is implemented in `Visit...` methods, which are executed depending on the kind of AST node visited.
* The AST of the newly built derivative function's declaration is returned to `CladPlugin`, where the call to `clad::differentiate(f, ...)/clad::gradient(f, ...)` is [updated](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L122) and `f` is replaced by a reference to the newly created derivative function `f_...`. This effectively results in the execution of `clad::differentiate(f_...)/clad::gradient(f_...)`, which constructs `CladFunction` with a pointer to the newly created derivative. Therefore, user's calls to `.execute` method will invoke the newly generated derivative.
* Finally, derivative's AST is [passed](https://github.com/vgvassilev/clad/blob/a264195f00792feeebe63ac7a8ab815c02d20eee/tools/ClangPlugin.cpp#L145) for further processing by Clang compiler (LLVM IR generation, optimizations, machine code generation, etc.).

## Citing Clad
```latex
% Peer-Reviewed Publication
%
% 16th International workshop on Advanced Computing and Analysis Techniques
% in physics research (ACAT), 1-5 September, 2014, Prague, The Czech Republic
%
@inproceedings{Vassilev_Clad,
  author = {Vassilev,V. and Vassilev,M. and Penev,A. and Moneta,L. and Ilieva,V.},
  title = {Clad -- Automatic Differentiation Using Clang and LLVM},
  journal = {Journal of Physics: Conference Series},
  year = 2015,
  month = {may},
  volume = {608},
  number = {1},
  pages = {012055},
  doi = {10.1088/1742-6596/608/1/012055},
  url = {https://iopscience.iop.org/article/10.1088/1742-6596/608/1/012055/pdf},
  publisher = {IOP Publishing}
}
```

## Some references
[ACAT 2014 Slides](https://indico.cern.ch/event/258092/session/8/contribution/90/material/slides/0.pdf)  
[Martin's GSoC2014 Final Report](https://indico.cern.ch/event/337174/contribution/2/material/slides/0.pdf)  
[LLVM Poster](http://llvm.org/devmtg/2013-11/slides/Vassilev-Poster.pdf)  
[Violeta's GSoC2013 Final Report](http://prezi.com/g1iggppw76wl/autodiff/)  
[DIANA-HEP Meeting 2018](https://indico.cern.ch/event/760152/contributions/3153263/attachments/1730121/2795841/Clad-DIANA-HEP.pdf)  
[Demo at CERN](https://indico.cern.ch/event/808843/contributions/3368929/attachments/1817666/2971512/clad_demo.pdf)

##  Contributors
Founder of the project is Vassil Vassilev as part of his research interests and vision.
We have quite a few [contributors](https://github.com/vgvassilev/clad/blob/master/Credits.txt).
If we missed you, please contact us!

##  How to Contribute
Have a look at our [open positions](/vacancies) and [open projects](/open_projects) pages


## How to install
Find instructions at our [README](https://github.com/vgvassilev/clad/blob/master/README.md)

  

