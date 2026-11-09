# In-source unit tests for Ada

- **Feature ID:** rfc-in-source-tests
- **Start Date:** 2026-11-09
- **Status:** Proposed

This proposal requests language design review of an implemented approach to
in-source unit tests, as a candidate Flare extension.

Ada programmers should be able to write tests alongside their implementation,
including inside package bodies, and build and run them with one project-aware
command. The proposed extension introduces `Test`, `Fixture`, and `Cleanup`
aspects. Test code is always checked but excluded from code generation unless
explicitly enabled.

## Motivation

GNATtest's generated harness and separate test project support workflows with
strong traceability requirements. For users developing tests incrementally
alongside their code, maintaining that structure can be burdensome.

In-source tests let users choose the scenarios they need and exercise body-local
declarations without exposing them through the package interface. This
complements GNATtest's existing workflow.

## Example

```ada
with AUnit.Assertions; use AUnit.Assertions;

package body Store is

   Count : Natural := 0;

   procedure Put is
   begin
      Count := Count + 1;
   end Put;

   procedure Empty_Store (N : out Natural) with Fixture is
   begin
      Count := 0;
      N := Count;
   end Empty_Store;

   procedure Reset (Empty_Store : in out Natural) with Cleanup is
   begin
      Count := 0;
   end Reset;

   procedure Put_Adds_One (Empty_Store : Natural) with Test is
   begin
      Put;
      Assert (Count = Empty_Store + 1, "Put must add one item");
   end Put_Adds_One;

end Store;
```

Running `gprtest -P store.gpr` discovers the test, prepares its fixture, executes
it, and performs cleanup.

## Test and fixture rules

The prototype implements the following model:

- A test is a procedure declared directly in a package body. Parameterless tests
  run directly. Tests may also be separate subunits, with the aspect on the body
  stub.
- A fixture procedure has exactly one `out` parameter, which supplies its value.
  Its other parameters request dependencies.
- A cleanup procedure has exactly one `in out` parameter, identifying the fixture
  it releases. Cleanup is optional.
- Tests and fixture dependencies receive fixtures through `in` or `in out`
  parameters.
- Fixture selection uses the parameter name by default. An explicit parameter
  aspect selects another fixture, for example
  `Dir : Directory with Fixture => Log_Dir`. Explicit selection requires Ada 2022
  parameter-aspect syntax.
- Fixtures are selected by name within the same declarative part. Duplicate
  fixture names, multiple cleanups for one fixture, missing fixtures, and
  dependency cycles are rejected. Ordinary Ada compilation checks type
  compatibility.

Each test invocation receives fresh fixtures. A fixture is constructed at most
once per invocation, including when requested both directly and through another
fixture. Dependencies are constructed first, following parameter order otherwise.
Cleanup runs in reverse construction order.

Fixture values follow ordinary Ada parameter-passing rules. Limited types are
supported. The prototype requires fixture types to be definite and
default-initializable.

## Failure handling

A fixture becomes ready only after setup completes successfully, including its
assertions. A failing setup prevents the test from running. Previously prepared
fixtures are cleaned up; the failing setup must release its own partially
acquired resources.

Cleanup runs after a successful test, a failed assertion, or an unexpected
exception. Every applicable cleanup is attempted even if another cleanup fails.

Test assertions are reported as failures. Setup errors, unexpected test
exceptions, and cleanup errors are reported as errors. A test may report both an
assertion failure and a cleanup error. These failures do not prevent subsequent
tests from running.

## Compiler semantics

`Test`, `Fixture`, and `Cleanup` mark declarations as test code. The aspects
accept an omitted value or the Boolean literals `True` and `False`; other
expressions are rejected. `False` leaves the declaration as ordinary code.

Test code undergoes semantic analysis in every build. Code generation is enabled
by `-gnateU` and disabled by default. Test-only types and objects can also carry
`Test`.

Ordinary code cannot reference test entities, including through calls,
renamings, generic actuals, or access attributes. Test code may reference
production code and other test code. The compiler enforces this boundary in both
compilation modes.

Exclusion does not remove source-level dependencies: tests using
`AUnit.Assertions` still require it to be available during ordinary compilation.

## Tooling

`gprtest` uses gprbuild's command-line parser and forwards its switches. It adds
`--no-run` to build without executing.

The prototype discovers tests with Libadalang and injects registrations inside
their declaring bodies, where body-local procedures remain visible. Elaboration
registers them with an AUnit-backed runtime. Tests with fixtures receive generated
wrappers implementing the lifecycle above.

Rewritten sources are built through `--src-subdirs`; user sources remain
untouched and retain their line numbers. The generated main executes the
registered tests, ordered by source file and line.

## Prototype scope and remaining work

Regression cases cover registration, source locations, exclusion, reference
restrictions, subunits, command-line forwarding, fixture composition, lifecycle
failures, and diagnostics.

The current implementation also identifies boundaries for subsequent work:

- Fixtures are local and live for one test invocation. Shared fixtures and
  parameterized cases are unsupported.
- There is no distinct annotation for test-only helper subprograms.
- Fixture support on light runtimes remains unimplemented.
- Generics, pure and preelaborated units, and nested subunits require further
  validation.
- Discovery does not yet handle an aspect placed only on a separate subprogram
  declaration.
- Excluded tests can still affect optimization decisions; exclusion does not
  currently guarantee object code identical to deleting the tests.
- Stubbing is separate follow-up work.

The compiler currently reuses ghost-code removal machinery. Fixture resolution,
execution, and AUnit integration belong to the tooling.

The requested LD review is of the proposed aspects and their compilation and
reference rules, using the existing prototype as the concrete basis for
discussion. Product prioritization and production readiness remain separate
decisions.
