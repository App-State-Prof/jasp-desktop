
Core - Unit Tests
==========

Guide to build Unit test project and run 
-----------------------------------------

1) use Qt creator and build the project JASP.pro

2) use qmake from terminal (in a different build directory)

```
    qmake ../jasp-desktop/JASP.pro
    make 
    ./JASPTests
```

Adding new unit tests to the project
------------------------------------

To add new test (for example Tester), include "AutomatedTests.h" in the header file and the last line in the file should be 
    DECLARE_TEST(Tester)

Update the JASP-Tests-app.pro file to include the source file and the header file

The test will run automatically when the project is built and run.


Unit Tests in the project
-------------------------

1) Opening of Text files (tests the AsyncLoader class)

2) Testing of OSF login and file opening

3) CSV importer 

4) SPSS importer


Analyses - Unit Tests
=====================

Running the tests
-----------------

Currently, the analysis unit tests are not integrated into the JASP-Tests app.  
To run the tests, the R package [jasptools](https://github.com/jasp-stats/jasptools) is required.  
Note that this package does not need to be installed manually; it is installed automatically when you clone the jasp-desktop environment.  
To load the package simply type `library(jasptools, lib.loc="path/to/jasp-desktop/Tools")`

To run all tests type
```
jasptools::testAll()
```

Any test that fails is shown in the console.  
Warnings may be ignored, but should be minimized.

It is also possible to test a specific analysis, as running all unit tests may take some time
```
jasptools::testAnalysis("Anova")
```

Fixing the tests
----------------
If a test shows up as failed, you should verify why this is and fix it before making a pull request.  
If you made a legitimate change that the test does not cover, then the unit test should be updated.  
Locate it under [JASP-Tests/R/tests/testthat](https://github.com/jasp-stats/jasp-desktop/tree/development/JASP-Tests/R/tests/testthat) and change the offending test.  
Note that if the failed test was related to plotting, then you may use
```
jasptools::inspectTestPlots("Anova")
```
to inspect differences between the saved reference plot and the failing plot.  
If you validate the failing plot (because it was a legitimate change), it will replace the reference plot in the figs folder.
Please ensure that there are no changes to figs/deps.txt as a result of your change (see the section below about "Dependencies related to plots").

Writing new tests
-----------------
The analysis tests in JASP are based on the R package testthat.  
For more information about this package see [this online book chapter by Hadley Wickam](http://r-pkgs.had.co.nz/tests.html).

The general structure of the tests is as follows:  
\- A folder titled testthat contains a number of test-analysisName.R files.  
-- Each test file has an analysis specific context and consists of tests that check that specific analysis.  
--- Each test in a file checks a specific expectation of a small portion of analysis functionality (e.g., a table, a plot, error handling, etc.)  

The testthat package offers a number of expectations useful for testing.  
JASP offers two additional expectations:
```
expect_equal_tables(test, ref, ...)
expect_equal_plots(test, name, dir)
```
`expect_equal_tables` takes the data of a JASP table list and compares it to a reference list.  
This reference list can be created by supplying a table to jasptools.
```
options <- jasptools::analysisOptions("BinomialTest")
options[["variables"]] <- "contBinom"
results <- jasptools::run("BinomialTest", "debug.csv", options, view=FALSE)
table <- results[["results"]][["binomial"]][["data"]]
jasptools::makeTestTable(table)
```
The above returns the output

`list("contBinom", 0, 58, 100, 0.58, 0.133210619207213, 0.477119195723914,  
 0.678014460645203, "TRUE", "contBinom", 1, 42, 100, 0.42, 0.133210619207213,  
 0.321985539354797, 0.522880804276086, "FALSE")`

We can now write the expectation
```
test_that("Binomial table results match", {
  options <- jasptools::analysisOptions("BinomialTest")
  options[["variables"]] <- "contBinom"
  results <- jasptools::run("BinomialTest", "debug.csv", options, view=FALSE)
  table <- results[["results"]][["binomial"]][["data"]]
  expect_equal_tables(table, 
   list("contBinom", 0, 58, 100, 0.58, 0.133210619207213, 0.477119195723914,
    0.678014460645203, "TRUE", "contBinom", 1, 42, 100, 0.42, 0.133210619207213,
    0.321985539354797, 0.522880804276086, "FALSE"))
})
```

`expect_equal_tables` is a wrapper around `testthat::expect_equal` and accepts the same arguments.

In a similar fashion `expect_equal_plots` may be used to test regression of plots.  
This function is a wrapper around `vdiffr::expect_doppelganger` (for more information about vdffir see their [github page](https://github.com/lionel-/vdiffr)).
The function takes a plot object or recorded plot and compares it to a stored .svg file.  
This results in an expectation like
```
test_that("Descriptives plot matches", {
  options <- jasptools::analysisOptions("TTestIndependentSamples")
  options$variables <- "contNormal"
  options$groupingVariable <- "contBinom"
  options$descriptivesPlots <- TRUE
  results <- jasptools::run("TTestIndependentSamples", "debug.csv", options, view=FALSE)
  testPlot <- results[["state"]][["figures"]][[1]]
  expect_equal_plots(testPlot, "descriptives", dir="TTestIndependentSamples")
})
```
To validate a plot for a newly created test, run
```
jasptools::inspectTestPlots("TTestIndependentSamples")
```
This function starts a Shiny application that allows you to view and then validate your new plot.  
Validating a plot places it in figs/analysisName.

As noted earlier, testthat offers a number of expectations as well.  
You should use whatever is most suitable for the situation.

#### Dependencies related to plots
Note that it is very important that all plots are created with equal versions of certain dependencies.  
If this is not the case, then we cannot compare plots across different systems and platforms.  
The settings you must use can be found in [figs/deps.txt](https://github.com/jasp-stats/jasp-desktop/blob/development/JASP-Tests/R/tests/figs/deps.txt)  
This file will look something like (but not necessarily the same as)
```
Fontconfig: 2.11.94
FreeType: 2.6.1
Cairo: 1.14.6
vdiffr: 0.2.1
svglite: 1.2.1
ggplot2: 2.2.1
```
You must never edit deps.txt unless the travis-CI config is updated as well.
If you have different versions of the dependencies installed the plots will be skipped.
In this case either update your configuration or rely on the automatic test that start when you make a pull request.
