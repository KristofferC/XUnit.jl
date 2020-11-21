# XUnit.jl

`XUnit.jl` is a unit-testing framework for Julia.

It is a drop-in replacement for the test macros of `Base.Test` supporting:
* Basic progress indication
* The ability to
  * run tests sequentially
  * run tests in shuffled mode (sequentially, in random order)
  * run tests in parallel mode (using multiple threads)
  * run tests in distributed mode (using multiple processes)
  * run only a subset of test sets
* XUnit/JUnit style of reporting test results

## Installation

```julia
using Pkg
Pkg.add("XUnit")
```

## Usage

`XUnit.jl` allows to organize your tests hierarchically.

There are two concepts used for creating your test hierarchy:
 - `test-suite`: is used for grouping other `test-suite`s and `test-case`s. It is important that a `test-suite` doesn't contain any code related to specific tests directly in its body. You can create a `test-suite` using `@testset` and `@testsuite` macros.
 - `test-case`: is used for declaring a unit-test. You can think of `test-case`s as leaves of your test hierarchy. You can create a `test-case` using `@testcase` macro.

Executing the tests using `XUnit.jl` happens in two phases:
 - Scheduling: when Julia processes test hierarchy macros (`@testsuite` and `@testcase`), it doesn't immediately runs them. Instead, it output a representation of the `test-suite`.
 - Running: After scheduling the tests, you have the choice of running the tests using a `test-runner`. Each `test-runner` has a different strategy for running the tests (explained below).

`XUnit.jl` rewrites/re-exports `@testset` and `@test*` macros provided by `Base.Test` (and you need to use these macros directly from `XUnit` instead of `Test`).

In addition, there are two more macros provided:

 - `@testsuite`: is used for grouping other `@testsuite`s and `@testcase`s. This macro schedules the `@testsuite`s and `@testcase`s in its body (and runs them at end). It is important that a `@testsuite` doesn't contain any code related to specific tests directly in its body. For backward compatibility with `Base.Test`, `@testset` is very similar to `@testsuite` with a distinction that it always runs the tests sequentially (and doesn't support the same keyword arguments available for `@testsuite`). The deferred execution of test-cases is a big differentiator between `XUnit.jl` and `Base.Test`. Even though you can drop-in replace `XUnit.jl` with `Base.Test`, but we strongly recommend also replace your topmost `@testset` with `@testsuite` to benefit from deferred execution in different modes (i.e., shuffled, parallel, or distributed). This is done in the example below.
 - `@testcase`: is used for encapsulating a `test-case`. You can think of `@testcase`s as leaves of your test hierarchy. The body of an `@testcase` is not executed right away. Instead, it gets scheduled for being executed later.

 **Note**: it's suggested that `@testcase`s become the leaf nodes of your test hierarchy. Even though, for now, you can have other `@testset`s, `@testsuite`s or even `@testcase`s under a `@testcase`, where in this case, all those are considered the same (like a `@testset`) and will only impact the reporting, but won't have any impact on the execution of the tests, as still the top-most `@testcase` gets scheduled for deferred execution.

**Note**:the body of a `@testsuite` always gets executed at scheduling time, as it needs to gather possible underlying `@testcase`s. Thus, it's a good practice to put your tests under a `@testcase` (instead of putting them under a `@testsuite`), as any tests defined under a `@testsuite` are executed sequentially at scheduling time.

After scheduling the tests (which happens by running the topmost `@testsuite`), the tests will run using the specified test-runner. You pass your desired test-runner as a keyword argument `runner` to the topmost `@testsuite` or `@testcase`. Here are the available test runners:
 - `SequentialTestRunner`: this is the default test-runner, which behaves similarly to `Base.Test` and runs all your tests sequentially using a single thread of execution.
 - `ShuffledTestRunner`: similar to `SequentialTestRunner`, it runs the tests using a single thread of execution, but it shuffles them and runs them in random order.
 - `ParallelTestRunner`: runs your tests in parallel using multiple threads of execution. It runs the tests with all available Julia threads. Please refer to the [Julia documentation](https://docs.julialang.org/en/v1/manual/multi-threading/#Starting-Julia-with-multiple-threads) to know about the possible ways to start Julia with multiple threads.

In the end, the test results are printed to the standard output (similar to `Base.Test`), but you can also get the XUnit/JUnit style of test reports (in `XML` format) by passing the `xml_report=true` keyword argument to the topmost `@testsuite` or `@testcase`. After calling this function on your test-suite, the results are available in a file determined by the `xml_output(testsuite)` function.

Here is an example that tries to show the above explanations in practice:

```julia
using XUnit

function fn_throws()
    throw(KeyError("Test error!"))
end

function fn_nothrows()
    return nothing
end

@testsuite runner=ParallelTestRunner() xml_report=true failure_handler=(testsuite) -> run(`open ./$(xml_output(testsuite))`) "Top Parent" begin
    @testset "XTest 1" begin
        @testcase "Child XTest 1" begin
            @test 1 == 1
            @test 2 == 2
            @test 3 == 3
        end
        @testcase "Child XTest 3" begin
            test_println("You should only see me as part of Child XTest 3 output!")
            @test 4 == 4
            @test 5 == 5
            @test 2 * 2 == 5
            @test fn_throws()
            @test 9 == 9
            @test 10 == 10
        end
    end
    @testset "XTest 2" begin
        @testcase "Child XTest 4" begin
            @test 6 == 6
            @test 7 == 7
            @test_throws KeyError fn_throws()
            @test_throws KeyError fn_nothrows()
            @test 8 == 8
            @test 11 == 11
        end
        @testcase "Child XTest 5" begin
            @test 11 == 11
            @test 12 == 12
            @test 13 == 13
        end
        @testcase "Child XTest 12" begin
            @test 1211 == 1211
            @test 1212 == 1212 * 2
            @test 1213 == 1213
        end
    end
    @testset "XTest 3" begin
        @testset "Child XTest 3" begin
            @testcase "Child XTest 3-1" begin
                x = 31
                y = 52
                @assert 5 == 9
                @test x == 31
                @test 32 == 32
                @test y == 33
            end
            @testcase "Child XTest 3-2" begin
                @test 36 == 36
                @test 37 == 37
                @test fn_throws()
                @test 38 == 38
            end
        end
    end
end
```

Sample output:

```julia
Running Top Parent tests...
  Running Top Parent/XTest 1 tests...
    Scheduling Top Parent/XTest 1/Child XTest 1 tests...
    Scheduling Top Parent/XTest 1/Child XTest 3 tests...
  Running Top Parent/XTest 2 tests...
    Scheduling Top Parent/XTest 2/Child XTest 4 tests...
    Scheduling Top Parent/XTest 2/Child XTest 5 tests...
    Scheduling Top Parent/XTest 2/Child XTest 12 tests...
  Running Top Parent/XTest 3 tests...
    Running Top Parent/XTest 3/Child XTest 3 tests...
      Scheduling Top Parent/XTest 3/Child XTest 3/Child XTest 3-1 tests...
      Scheduling Top Parent/XTest 3/Child XTest 3/Child XTest 3-2 tests...
-> Running Top Parent/XTest 2/Child XTest 12 test-case (on tid=4)...
-> Running Top Parent/XTest 3/Child XTest 3/Child XTest 3-1 test-case (on tid=3)...
-> Running Top Parent/XTest 2/Child XTest 5 test-case (on tid=2)...
-> Running Top Parent/XTest 3/Child XTest 3/Child XTest 3-2 test-case (on tid=1)...
-> Running Top Parent/XTest 2/Child XTest 4 test-case (on tid=2)...
-> Running Top Parent/XTest 1/Child XTest 3 test-case (on tid=4)...
-> Running Top Parent/XTest 1/Child XTest 1 test-case (on tid=2)...
Child XTest 3: Test Failed at ./XUnit/test/unittests.jl:22
  Expression: 2 * 2 == 5
   Evaluated: 4 == 5
Child XTest 3: Error During Test at ./XUnit/test/unittests.jl:23
  Test threw exception
  Expression: fn_throws()
  KeyError: key "Test error!" not found
  Stacktrace:
   [1] fn_throws() at ./XUnit/test/unittests.jl:4
   [2] macro expansion at ./XUnit/test/unittests.jl:23 [inlined]
   [3] (::var"#4#19")() at ./XUnit/src/XUnit.jl:557
   [4] gather_test_metrics(::var"#4#19", ::RichReportingTestSet, ::Nothing; run::Bool) at ./XUnit/src/test-metrics.jl:57
   [5] gather_test_metrics(::Function, ::XUnit._AsyncTestCase{XUnit.AsyncTestSuite}; run::Bool) at ./XUnit/src/test-metrics.jl:51
   [6] #gather_test_metrics#60 at ./XUnit/src/test-metrics.jl:47 [inlined]
   [7] run_single_testcase(::Array{XUnit.AsyncTestSuite,1}, ::XUnit._AsyncTestCase{XUnit.AsyncTestSuite}) at ./XUnit/src/test-runners.jl:287
   [8] macro expansion at ./XUnit/src/test-runners.jl:186 [inlined]
   [9] (::XUnit.var"#54#threadsfor_fun#22"{Array{XUnit.ScheduledTest,1},Base.Threads.Atomic{Int64},Bool,Nothing,UnitRange{Int64}})(::Bool) at ./threadingconstructs.jl:81
   [10] (::XUnit.var"#54#threadsfor_fun#22"{Array{XUnit.ScheduledTest,1},Base.Threads.Atomic{Int64},Bool,Nothing,UnitRange{Int64}})() at ./threadingconstructs.jl:48

Child XTest 4: Test Failed at ./XUnit/test/unittests.jl:33
  Expression: fn_nothrows()
    Expected: KeyError
  No exception thrown
Child XTest 12: Test Failed at ./XUnit/test/unittests.jl:44
  Expression: 1212 == 1212 * 2
   Evaluated: 1212 == 2424
Child XTest 3-1: Error During Test at ./XUnit/test/unittests.jl:51
  Got exception outside of a @test
  AssertionError: 5 == 9
  Stacktrace:
   [1] macro expansion at ./XUnit/test/unittests.jl:53 [inlined]
   [2] (::var"#12#27")() at ./XUnit/src/XUnit.jl:557
   [3] gather_test_metrics(::var"#12#27", ::RichReportingTestSet, ::Nothing; run::Bool) at ./XUnit/src/test-metrics.jl:57
   [4] gather_test_metrics(::Function, ::XUnit._AsyncTestCase{XUnit.AsyncTestSuite}; run::Bool) at ./XUnit/src/test-metrics.jl:51
   [5] #gather_test_metrics#60 at ./XUnit/src/test-metrics.jl:47 [inlined]
   [6] run_single_testcase(::Array{XUnit.AsyncTestSuite,1}, ::XUnit._AsyncTestCase{XUnit.AsyncTestSuite}) at ./XUnit/src/test-runners.jl:287
   [7] macro expansion at ./XUnit/src/test-runners.jl:186 [inlined]
   [8] (::XUnit.var"#54#threadsfor_fun#22"{Array{XUnit.ScheduledTest,1},Base.Threads.Atomic{Int64},Bool,Nothing,UnitRange{Int64}})(::Bool) at ./threadingconstructs.jl:81
   [9] (::XUnit.var"#54#threadsfor_fun#22"{Array{XUnit.ScheduledTest,1},Base.Threads.Atomic{Int64},Bool,Nothing,UnitRange{Int64}})() at ./threadingconstructs.jl:48

Child XTest 3-2: Error During Test at ./XUnit/test/unittests.jl:61
  Test threw exception
  Expression: fn_throws()
  KeyError: key "Test error!" not found
  Stacktrace:
   [1] fn_throws() at ./XUnit/test/unittests.jl:4
   [2] macro expansion at ./XUnit/test/unittests.jl:61 [inlined]
   [3] (::var"#14#29")() at ./XUnit/src/XUnit.jl:557
   [4] gather_test_metrics(::var"#14#29", ::RichReportingTestSet, ::Nothing; run::Bool) at ./XUnit/src/test-metrics.jl:57
   [5] gather_test_metrics(::Function, ::XUnit._AsyncTestCase{XUnit.AsyncTestSuite}; run::Bool) at ./XUnit/src/test-metrics.jl:51
   [6] #gather_test_metrics#60 at ./XUnit/src/test-metrics.jl:47 [inlined]
   [7] run_single_testcase(::Array{XUnit.AsyncTestSuite,1}, ::XUnit._AsyncTestCase{XUnit.AsyncTestSuite}) at ./XUnit/src/test-runners.jl:287
   [8] macro expansion at ./XUnit/src/test-runners.jl:186 [inlined]
   [9] (::XUnit.var"#54#threadsfor_fun#22"{Array{XUnit.ScheduledTest,1},Base.Threads.Atomic{Int64},Bool,Nothing,UnitRange{Int64}})(::Bool) at ./threadingconstructs.jl:81
   [10] (::XUnit.var"#54#threadsfor_fun#22"{Array{XUnit.ScheduledTest,1},Base.Threads.Atomic{Int64},Bool,Nothing,UnitRange{Int64}})() at ./threadingconstructs.jl:48

Test Summary:         | Pass  Fail  Error  Total
Top Parent            |   20     3      3     26
  XTest 1             |    7     1      1      9
    Child XTest 1     |    3                   3
    Child XTest 3     |    4     1      1      6
  XTest 2             |   10     2            12
    Child XTest 4     |    5     1             6
    Child XTest 5     |    3                   3
    Child XTest 12    |    2     1             3
  XTest 3             |    3            2      5
    Child XTest 3     |    3            2      5
      Child XTest 3-1 |                 1      1
      Child XTest 3-2 |    3            1      4
ERROR: LoadError: Some tests did not pass: 20 passed, 3 failed, 3 errored, 0 broken.
in expression starting at ./XUnit/test/unittests.jl:11
```

## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update the tests as appropriate.

## License
[MIT](https://choosealicense.com/licenses/mit/)

## Authors

- [Mohammad Dashti](mailto:mohammad.dashti[at]relational[dot]ai)
- [Todd J. Green](mailto:todd.green[at]relational[dot]ai)
