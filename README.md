[![GitHub issues](https://img.shields.io/github/issues/kyonifer/koma.svg?maxAge=2592000)](https://github.com/kyonifer/koma/issues)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
[![Bintray](https://img.shields.io/bintray/v/kyonifer/maven/koma-core.svg?maxAge=2592000)](https://bintray.com/kyonifer/maven)
[![Travis](https://img.shields.io/travis/kyonifer/koma/master.svg)](https://travis-ci.org/kyonifer/koma)
[![AppVeyor](https://img.shields.io/appveyor/ci/kyonifer/koma/master.svg)](https://ci.appveyor.com/project/kyonifer/koma)
### Koma

Koma is a scientific computing environment for Kotlin that emphasizes language/platform interop, performance, and flexibility.

## Project Goals

The project aims to:

- Create a scientific programming environment for Kotlin that is familiar to people used to NumPy or
MATLAB
- Support writing numerical code once in Kotlin, and then deploy that code on JVM, JS, and native platforms
- Support polyglot usage from Kotlin, Java, other JVM languages, as well as foreign interop with legacy Python or MATLAB code
- Use pluggable back-ends to offload the actual computation to pre-existing libraries, depending on the target platform
- Use code already written by other projects where possible to avoid duplication

## Features

Koma has two components: A set of flat functions which mimic the behavior of NumPy/MATLAB,
and an underlying object oriented hierarchy which dispatch those function calls. The flat functions are in the
top-level files [here](core/src/koma/) and the underlying matrix implementations use the interfaces defined [here](core/src/koma/matrix/).
It is easiest to get up to speed on Koma by reading through the available top-level functions.

We currently have:

- Integrated plotting (via xchart)

- Pluggable matrix back-end for pure-Java and high-performance modes on the JVM (via MTJ, JBlas, and EJML)

- Many linear algebra operations supported.

- Ongoing work for js and native backends (core builds on all three, just need working backends on those platforms)

## Using the Library From a Gradle Project

Koma is hosted on bintray. First add it to your repos:

```Groovy
repositories { 
    maven { 
        url "http://dl.bintray.com/kyonifer/maven" 
        jcenter()
    } 
}
```

Then add a dependency on `core` as well as whatever backend you want to
use for doing the computation. For example, if you want to use koma with the
MTJ backend, you'd write:

```Groovy
dependencies{
    compile group: "koma", name:"backend-matrix-mtj", version: "0.10"
    compile group: "koma", name:"core", version:"0.10"
}
```
Right now koma-backend-ejml, koma-backend-mtj, and koma-backend-jblas are supported.
Koma will automatically use the backend you add to the project. If you'd like to use more
than one backend in a project at the same time, see [multiple backends](#multiple-backends)
 below.

## Example Usage

The following is an example of plotting some random noise. The koma.* namespace brings in all the
top-level functions (the only other import you may need is koma.matrix.Matrix if you need to type
  a function you write).

```Kotlin
import koma.*

fun main(args: Array<String>)
{

    // Create some normal random noise
    var a = randn(100,2)
    var b = cumsum(a)

    figure(1)
    // Second parameter is color
    plot(a, 'b', "First Run")
    plot(a+1, 'y', "First Run Offset")
    xlabel("Time (s)")
    ylabel("Magnitude")
    title("White Noise")

    figure(2)
    plot(b, 'g') // green
    xlabel("Velocity (lightweeks/minute)")
    ylabel("Intelligence")
    title("Random Walk")

}
```
![](https://raw.githubusercontent.com/kyonifer/koma/imgs/plotting.png)

## Functionality

### Math Functions
Matrices have useful map functions that return matrices for chaining operations (see [here](core/src/koma/extensions.kt) for a complete list).

```Kotlin

    // Create a 3x3 identity and then add 0.1 to all elements
    val x = eye(3) + 0.1

    // Map each element through a function that adds .01
    val y = x.mapMat { it + .01 }
    
    // Map each element through a function that adds or subtracts depending on the element index
    val z = x.mapMatIndexed { row, col, ele -> if (row > col) ele + 1 else ele - 1 }

    // Are there any elements greater than 1?
    val hasGreater = x.any { it > 1 }

    // Are all elements greater than 1?
    val allGreater = x.all { it > 1 }

    // Apply a function to a row at a time and store the outputs in a contiguous matrix
    val sins = x.mapRows { row -> sin(row) }
    
    // Print all elements greater than 1
    x.each { if (it>1) println(it) }
    
``` 

We can also do some linear algebra:

```Kotlin

    // Matrix literal syntax, see creators.kt for 
    // convenience functions like zeros(5,5)
    var A = mat[1,0,0 end
                0,3,0 end
                0,0,4]

    // Calculate the matrix inverse
    var Ainv = A.inv()

    var b = mat[2,2,4].T

    // Use overloaded operators:
    
    // * is matrix multiplication 
    var c = A*b + 1
    
    // emul is element-wise multiplication
    var d = (A emul A) + 1

    // Number of decimals to show
    format("short")

    println(c)

```

Which produces:

```
Output:

mat[ 3.00  end
     7.00  end
     17.00 ]
```

Many special functions are supported (see [the matrix interface](core/src/koma/matrix/Matrix.kt) for a complete list):

```Kotlin

    val a = 2*eye(3)+.01 // eye is identity matrix
    
    a.chol()  // Cholesky decomposition
    a.det()   // Determinant
    a.diag()  // Diagonal vector
    a.inv()   // Matrix inverse
    a.norm()  // Matrix norm

```

Scalar functions can be applied elementwise to matrices (see [here](core/src/koma/scalarfuncs.kt) for a complete list):

```Kotlin
    val x = create(0..100)/5.0  // Matrix of 0, 1/5, 2/5, ...
    val y = sin(x)              // Sin applied elementwise
    plot(y)                     // Plot of sin function
```

Matrix indexing and slicing is supported (see [here](core/src/koma/operators.kt) for a list of operators as well as the Matrix<T> type):

```Kotlin

    val x = randn(5,5)
    val y = x[0,0..4] // Grab the first row
    
    x[0..2,0..3] = zeros(3,4) // Set the upper-left 3x4 sub-matrix of x to zero

```

A Matrix is convertible to an Iterable:

```Kotlin
    val x = randn(5,5).toIterable()
    
    // Adds all elements and returns sum
    x.reduce { x, y -> x+y }
    
    // Returns list of all elements greater than 4
    x.find { it > 4 }

```
### Multiple Backends

Koma supports using multiple backends simultaneously. This is useful if e.g.
you need to work with multiple libraries which require different matrix containers
(the unfortunate reality on the JVM at the moment).

You can change the backend being used by koma's top-level functions at
any time by setting a property in the koma namespace. In Kotlin this looks
like:

```Kotlin
import koma.matrix.ejml.EJMLMatrixFactory
...

// Make subsequent function calls use the EJML backend
koma.factory = EJMLMatrixFactory()

val a = zeros(3,3) // An EJMLMatrix

// Make subsequent function calls use the MTJ backend
// (doesnt affect previous returns)
koma.factory = MTJMatrixFactory()

val b = zeros(3,3) // Now returns an MTJMatrix
```

This property can be set from Java and other languages via
`koma.Options.setFactory(...)`. If not set, koma will default
to using MTJ, EJML, and then JBlas in that order.

It is also possible to create any matrix type manually by using the
corresponding factory. For example, even if `koma.factory` is set to
MTJMatrixFactory, you could write

```kotlin
val eFac = EJMLMatrixFactory()
val a = eFac.eye(3, 3)
val b = eFac.ones(3, 3)
println(a+b) // Uses EJML's addition algorithm, not MTJ's
```

This makes it easy to use libraries requiring different matrix
containers simultaneously.

### Validation

Koma includes support for matrix dimension / attribute validation. For
example, suppose you have a function that takes in two matrices A and B.
If we know that A is a row vector (i.e. Nx1), B is a matrix with 2 rows
and the same number of columns as A has rows (i.e. 2xN), we could write

```Kotlin
import koma.matrix.Matrix
import koma.util.validation.validate

fun foo(A: Matrix<Double>, B: Matrix<Double>) {
    validate {
        A("A") { 'N'  x  1  }
        B("B") {  2   x 'N' }
    }
    // Do stuff with A and B...
}
```

The validate block will see that you used the same variable 'N' twice, 
and make sure the matrices passed in have those dimensions matching. The 
DSL supports an arbitrary number of inputs of arbitrary dimension and 
also validates attributes like symmetricity. See [the validate README](https://github.com/kyonifer/koma/blob/master/core/srcjvm/koma/util/validation/README.md)
for more information.

## Roadmap

Planned functionality:

* Implement Matrix for other primitive types, such as Int (primary difficulty is lack of support in backends)
* Add a pluggable N-D container that uses backends such as [libDyND](https://github.com/libdynd/libdynd) and [ND4j](http://nd4j.org/)
* Support arbitrary data storage for non-numerical data (e.g. string) in ND container

## Related Projects

Koma has backends that wrap several other numerical projects on the JVM:

* Pure Java linear algebra: http://ejml.org/
* Pluggable native libs: https://github.com/fommil/matrix-toolkits-java
* Blas wrapper: http://jblas.org/

For a data analysis library similar to pandas, check out https://github.com/holgerbrandl/kplyr
