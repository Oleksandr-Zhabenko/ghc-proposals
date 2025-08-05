.. author:: Matthew Pickering, Rodrigo Mesquita, Adam Gundry
.. date-accepted:: 2025-03-06
.. ticket-url:: https://gitlab.haskell.org/ghc/ghc/-/issues/25828
.. implemented::
.. highlight:: haskell
.. header:: This proposal was `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/682>`_.
.. contents::
.. sectnum::


The contents of this proposal forms the basis of

>  Pickering, M., Mesquita, R. and Gundry, A., Explicit Level Imports. Trends in Functional Programming 2025.


Explicit Level Imports
======================

Template Haskell allows *staged metaprogramming*: writing Haskell code that is
executed during the compilation stage, rather than merely compiled so that it
can be executed at the later runtime stage. In practice, most programs can be
divided into small portions that are executed at compile-time (e.g. the
definition of ``makeLenses``), and the majority of the code that is not.
However, there are currently no clear boundaries between stages, so the compiler
must pessimistically assume that anything may be needed at compile-time or
runtime.

This has a variety of negative consequences. In particular:

* Using Template Haskell causes compile-time performance to suffer due to
  unnecessary (re)compilation.  This is particularly relevant for interactive
  use of the compiler within an IDE (e.g. via Haskell Language Server).

* Cross-compilation needs to compile and execute code on the target platform
  during the build process, as there is no way to execute splices on the host.

*Levels* are the mechanism which the typechecker uses to ensure that staged evaluation
is possible. Certain program contexts require identifiers to be at specific levels.
In order to fix level-incorrect programs, programmers rely on implicit *cross-stage persistence* (CSP): the assumption that if
a definition can be used at one level, then it can also be used at a later level. Cross-stage persistence is the root
cause of the unnecessary compilation: if a module is used at compile time (e.g. as a dependency
of a module with ``TemplateHaskell`` enabled),
the compiler must assume it may be needed for runtime as well.

Thus we propose a new
pair of extensions that allow programmers to write level-correct programs
without ubiquitous cross-stage persistence:

* ``NoImplicitStagePersistence`` forbids normal top-level identifiers from
  occurring within top-level splices or quotes, and

* ``ExplicitLevelImports`` allows imports that explicitly enable the use of the
  imported identifiers within top-level splices or quotes.

This proposal draws on ideas discussed previously in
`proposal #243: Stage Hygiene for Template Haskell
<https://github.com/ghc-proposals/ghc-proposals/pull/243>`_ and
`proposal #412: Explicit Splice Imports
<https://github.com/ghc-proposals/ghc-proposals/pull/412>`_.


Motivation
==========

Level-correct programs are necessary when using staged programming so
that the program can be cleanly separated into compile-time and runtime
portions. The existing mechanism to ensure level-correctness for imported
identifiers is called *path-based cross stage persistence*: informally, it allows you to
use imported identifiers at any level.
We want to explicitly control this, because it leads to the need to compile all modules
in a project for both runtime and compile time.

This proposal introduces an explicit means to control the level at which identifiers
are imported at. Therefore instead of relying on implicit persistence of an imported
identifier, the programmer has to explicitly request for the identifier to be available
at a later or earlier level.

The result is that identifiers can be used at precisely the level they are
bound, and no other levels.
By being very precise at levels modules are needed at, there are many advantages:

1. Currently, if a module enables ``TemplateHaskell``, then code generation for all imported modules must be performed
   before name resolution can take place. This ensures that any top level splices that may be encountered are able to be fully evaluated.
   This is a pessimisation because most of the imported identifiers, which we have taken such pains to ensure we can run, will not
   actually be used in a top-level splice.
   Proposals to increase build parallelism (such as `#14095 <https://gitlab.haskell.org/ghc/ghc/-/issues/14095>`_) are far less effective
   in projects that use ``TemplateHaskell``, because name resolution depends on code generation
   for all dependencies.
   By distinguishing imported modules whose code is executed only at compile time
   (which in common cases will be a small fraction of imported modules), we are
   able to improve this pessimisation.
2. GHC offers an ``-fno-code`` flag that instructs the compiler to parse and
   typecheck Haskell modules, but not to generate code, so as to offer
   quicker feedback to the user. However, any modules imported by a module using
   ``TemplateHaskell`` must be compiled to object code,
   despite the fact that we will not generate object code for the module
   itself. By distinguishing imported modules whose code is executed only at
   compile time, we can significantly reduce this unfortunate work, and entirely eliminate it in many
   cases.
3. IDEs such as Haskell Language Server face similar problems, where they are interested only in the result of type-checking modules, but when ``TemplateHaskell`` is enabled a large
   number of modules have to be cautiously compiled to bytecode.
4. By using splice imports we can separate the dependencies during dependency analysis into those needed only at compile-time and
   those needed only at runtime. Compile-time dependencies need to be compiled to object code before the current module, but need not be linked against. Runtime dependencies need to be type-checked before the current module, but their object code only needs to be available at link time.
5. Currently, when cross-compiling modules that use ``TemplateHaskell``, all
   splices are executed on the target even though compilation takes place on a
   separate host. This is a source of significant complexity. This proposal
   takes a step towards a future in which it will be possible to properly
   distinguish dependencies that need to be compiled for and executed on the
   host from those compiled for the target. (However, making this distinction in
   GHC and Cabal is likely to require significant further work, which is out of
   scope of the present proposal.)


Example
#######

A very common pattern for using Template Haskell is the following::

  {-# LANGUAGE TemplateHaskell #-}
  module M where
    import Control.Lens.TH (makeLenses)
    import N

    data T = MkT { foo :: Int }
    $(makeLenses ''T)
    ...

Here the ``makeLenses`` function is defined in a library, and used in a
declaration splice to generate some definitions (here lens bindings, but a
similar pattern is often used where libraries provide a TH-based mechanism for
deriving instances).

At the moment, GHC must compile dependent module ``N`` before it starts
type-checking module ``M``, because as far as it knows, running the splice might
end up executing code from ``N``.

This proposal allows the programmer to be explicit about the fact that
``makeLenses`` is used only in a splice, whereas the other import is definitely
not used in splices::

  {-# LANGUAGE ExplicitLevelImports #-}
  {-# LANGUAGE TemplateHaskell #-}
  module M where
    import splice Control.Lens.TH (makeLenses)
    import N

    data T = MkT { foo :: Int }
    $(makeLenses ''T)
    ...

Not only does this make the code easier to understand, but moreover GHC can now
tell from the imports that ``M`` depends only on the interface of ``N``, not on
its implementation.  Correspondingly, it is possible to start type-checking
``M`` as soon as ``N`` has been type-checked (before code generation has been
completed), and changes to the implementation of ``N`` that do not affect its
interface do not cause recompilation of ``M``.

In practice, many Haskell programs enable ``TemplateHaskell`` solely to be able
to call functions from external packages in top-level splices.  Thus versions of
this example occur frequently, and the changes required to use
``ExplicitLevelImports`` are modest (merely adding the ``splice`` keyword to a
few imports).


Definitions
###########

**stage**
  A moment in time for which modules are compiled and at which a program can be
  executed. Typically there is one compile-time and one runtime stage.

**level**
  Levels are a concept the type-checker uses to ensure that the evaluation is
  well-staged (i.e. that the compiler can execute compile-time stages before
  runtime stages).

  Within a module, every declaration and every (sub-)expression exists at an
  integer level.  The top-level declarations in the module are at level 0.  The
  level is increased by 1 when inside a quote and decreased by 1 inside a
  splice. In short:

  * ``$(e)`` is at level ``n`` iff ``e`` is at level ``n-1``
  * ``[| e |]`` is at level ``n`` iff ``e`` is at level ``n+1``

  Therefore the level of an expression can be calculated as the number of
  quotes surrounding the expression minus the number of splices. For
  example::

    -- foo is at level 0
    foo = $(let
      -- bar is at level -1
      bar = $(let
        -- baz is at level -2
        baz = [|
        -- qux is at level -1
          qux = [|
            -- quux is at level 0
            quux = [|
              -- quuz is at level 1
              quuz = 0
            |]
          |]
        |] in baz
      ) in bar
    )

  Note that GHC uses level 1 for top-level definitions, so all numbers internal to
  ghc are offset by +1. We use 0 for the top-level here as it is more consistent with
  literature on multi-stage languages.

**cross-stage persistence**
  See `Background: Cross Stage Persistence`_.

**level-correct**
  A program where every use site of an identifier or class instance occurs at the same level
  as the level of the definition site.

**top-level splice**
  A splice whose body is at a negative level (i.e. not surrounded by any quotations), or a quasiquoter.
  A top-level splice marks in a program where compile-time evaluation will occur.
  For example::

    -- A splice in an expression context, not surrounded by any quotes. Therefore
    -- baz is at level -1 and this is a top-level splice.
    foo = $(baz)

    -- A top-level declaration splice, when evaluated will insert declarations at
    -- this point.
    $(makeLenses ''A)

    -- A quasi-quoter, looks like a quote, but is actually syntactic sugar for a
    -- top-level splice.
    qq = [quasi| my-quasi-quoter]
    ====>
    qq = $(quoteExp quasi "my-quasi-quoter")



Background: Cross Stage Persistence
###################################

GHC currently has several means to fix level-incorrect programs automatically.
These techniques are (confusingly) called **cross-stage persistence**.

At the moment, all imported definitions are assumed to be bound at level 0.

If an identifier is used at a level different from the level at which it is
bound, there are two different mechanisms that are used to attempt to fix its
level:

* **Path-based persistence**: this allows global definitions at level ``deflvl`` to be
  made available at a different level ``uselvl`` in two cases:

  - If ``uselvl > deflvl``, intuitively because all global definitions will still exist in
    the defining module even if references to them are spliced at a future
    stage. For example, this allows a module to define a top-level identifier
    and refer to it in a quote in the same module.

  - If ``uselvl < deflvl`` and the definition was *imported* rather than being defined in
    the current module, intuitively because the dependency order on modules
    ensures the definition must have been compiled already. For example, this
    allows an imported identifier to be used in a splice.

* **Serialisation-based persistence (Lift)**: locally-bound variables can't be persisted
  using path-based persistence, but provided the variable's type is serialisable, we
  can serialise its value to persist it to *future* stages. This serialisation is
  defined as the ``lift`` method of the ``Lift`` typeclass.

  The following is level-incorrect as ``x`` is bound at level 0 but used at level
  1. It is fixed by serialisation-based persistence, which transforms the program
  into one where ``x`` is used at level 0 by the compiler automatically inserting a call to ``lift``::

    tardy x = [| x |]
    =>
    tardy x = [| $(lift x) |]

  All base types such as ``Int``, ``Bool``, ``Float``, ... instantiate ``Lift``, and user
  types can instantiate it automatically with ``DeriveLift``.

For example, the following program is accepted::

    {-# LANGUAGE TemplateHaskell #-}
    module M2 where
      suc :: Int -> Int
      suc = (+1)

      one :: Q Exp
      one = [| \x -> suc x |]

      another_one :: Int -> Q Exp
      another_one y = [| suc y |]

    {-# LANGUAGE TemplateHaskell #-}
    module M3 where
      import M2 (another_one)

      two = $(another_one 1)

* *Path-based persistence* explains why the occurrence of ``suc`` in examples
  ``one`` and ``another_one`` is accepted (since it is defined at level 0 but
  used at level 1), and why ``another_one`` can be used in a top-level splice
  (since it is imported at level 0 but used at level -1)

* *Serialisation-based persistence* explains why the ``y`` in ``another_one`` can be moved from
  a value that exists at level 0 to one that exists at level 1. The
  compiler will implicitly introduce a call to ``lift``::

      another_one y = [| suc y |]
      ===>
      another_one y = [| suc $(lift y) |]

  And ``lift`` will take care of converting the compile-time ``y`` into a runtime value.

  This strategy elaborates a level-incorrect program into a level correct one, which
  the user themselves could have written. Therefore persistence by lifting does
  not impose any requirements or use any assumptions about which stages modules
  are compiled for.

It is not possible for a locally-bound variable to be used earlier than the
stage at which it is bound (e.g. GHC will report a stage error for the
expression ``[| \ x -> $x |]``). Similarly, it is not possible for a global
definition to be used in a splice in the same module as its definition.


Proposed Change Specification
=============================

The precise changes
###################

This proposal adds two language extensions, ``ExplicitLevelImports``, which is
off by default in all existing language editions, and
``ImplicitStagePersistence``, which is, in contrast, enabled by default in all
existing language editions. They have the following effects:

* ``NoImplicitStagePersistence`` disables
  **path-based cross-stage persistence** (see Section 2.3) altogether.
  That is, use of a binding at a
  level other than the level at which it was defined or imported will result in a
  type error.  In particular, bindings imported using traditional ``import``
  statements may not be used inside of top-level splices, nor within quotes.
  However **serialisation-based cross-stage persistence** is entirely unaffected.

  ``ImplicitStagePersistence`` is the default because it preserves the existing
  behaviour of allowing path-based cross-stage persistence.

* ``ExplicitLevelImports`` adds **two new import modifiers**, ``splice`` and
  ``quote``, to the import syntax, which control the level at which identifiers
  from the module are brought into scope:

  - ``import splice M(f)`` imports ``f`` at level ``-1``.
  - ``import M(f)`` imports ``f`` at level ``0``.
  - ``import quote M(f)`` imports ``f`` at level ``1``.

  When ``ExplicitLevelImports`` is enabled, a build system can inspect the module headers
  and determine precisely which modules will be needed to be executed for compile-time
  and runtime. Only modules analysed to be needed at compile time are needed to be
  executed during compilation, and only runtime modules are needed to be linked into
  the final executable.

(Side note: in GHC today, with permissive path-based persistence, import entities are made available at all levels)

``ExplicitLevelImports`` implies ``NoImplicitStagePersistence``.  Thus users
typically need only enable ``ExplicitLevelImports`` (and ``TemplateHaskell``).

It is permitted to enable both ``ExplicitLevelImports`` and
``ImplicitStagePersistence`` (provided the latter appears later than the former,
so it overrides the implied ``NoImplicitStagePersistence``). This allows
``splice`` and ``quote`` imports to be used, but ``ImplicitStagePersistence``
still allows cross-stage persistence (and thus the compiler must still be
pessimistically assume all modules are needed at all stages). This combination
is supported to allow gradual migration of code bases following the change, and
for corner cases such as programmatic code generation, where the programmer may wish to use
the syntax of ``splice`` and ``quote`` imports without obliging the whole module
to be level-correct.

Example
-------

For example, the following is accepted under the default
``ImplicitStagePersistence``, but will be rejected under
``ExplicitLevelImports`` (which implies ``NoImplicitStagePersistence``)::

    import B (foo, bar)  -- foo :: Q Exp, bar :: Int

    quoteC = [| bar |]  -- Error: bar imported at level 0 but used at level 1
    spliceC = $( foo )  -- Error: foo imported at level 0 but used at level -1

However these errors can be fixed by using import modifiers::

    import splice B (foo)  -- foo :: Q Exp
    import quote  B (bar)  -- bar :: Int
    data C = MkC

    quoteC = [| bar |]  -- OK: bar is imported at level 1, and used at level 1
    spliceC = $( foo )  -- OK: foo imported at level -1 and used at level -1

For definitions in the same module, GHC has the following behaviour::

    baz = 3 :: Int
    foo = [| 3+4 |] :: Q Exp

    quoteC = [| baz |]  -- OK: implicit lifting makes baz appear at level 0
    spliceC = $( foo )  -- Error: foo defined at level 0, and used at level -1

Definitions like ``foo``, invoked in a splice, must be put in an imported module,
which can be compiled in advance to executable code, so the splice ``$(foo)``
can be run.

With ``ExplicitLevelImports``, ``spliceC`` become illegal; instead, ``foo``
must be put in another module and splice-imported as above. On the other hand,
due to the implicit lifting, ``quoteC`` is elaborated to ``[| $(lift baz) |]``,
which correctly places ``baz`` used at level 0.

Consequences and payoff
#######################

Using ``ExplicitLevelImports`` makes programming a little less convenient:
sometimes definitions must be put in another module; and imports must be
annotated with ``splice`` or ``quote``.  The payoff concerns performance, as we now
describe.

Recall (Section 2.2) that a module ``M`` may be compiled at stage ``R`` (for runtime)
or ``C`` (for compile time), or both.  A consequence of ``ExplicitLevelImports`` is
that we can decide which modules are needed at stage ``R`` and which at stage ``C``,
**based only on their import declarations**, as follows:

- The main module is compiled for ``R``.
- A normal import does not shift the stage at which the dependent module is required.
- If a module ``M`` splice-imports module ``A``, then compiling ``M`` at stage ``R`` or at stage ``C`` requires compiling module ``A`` at stage ``C``.
- If a module ``N`` quote-imports module ``B``, then compiling ``N`` at stage ``R`` or stage ``C`` requires compiling module ``B`` at stage ``R``.

Being able to classify each module into stage ``R`` or stage ``C`` (or both) is
extremely useful: **indeed it is the main payoff of this proposal**.
Specifically:

- **Binary sizes decrease**.  A module compiled at stage ``R`` must be linked into the
  final executable; but modules compiled only at stage ``C`` need not. Only Binary
  sizes decrease, because modules used only at compile time (i.e.
  splice-imported) need not be linked into the executable.

  Hence binary sizes decrease because modules needed only at stage C are not included.

- **Compile times decrease with `-fno-code`**.  GHC's ``-fno-code`` flag tells GHC to
  stop compiling after type-checking the module and producing an interface
  file.  It is used for Haddock, for HLS, and other tools. However with
  ``TemplateHaskell`` GHC must conservatively produce executable code for all
  modules anyhow, in case the module is called by a splice

  With ``ExplicitLevelImports`` and ``-fno-code``, only modules needed at stage
  C (usually a tiny fraction) need be compiled to executable code.

- **Compile-time parallelism increases**.  Even without ``-fno-code`` compile-time
  parallelism can increase with ``ExplicitLevelImports`` because we can start
  compiling a module as soon as

  - all its imports have been typechecked
  - all its splice-imported imports (typically very few) have been compiled to executable code

  This can substantially increase compile time parallelism because the lengthy code-generation phase of most modules moves entirely off the critical path.

- **Fewer modules are linked when splicing**.  When GHC invokes a TH splice, it
  dynamically links the TH code into GHC's executable.  In GHC today that step
  conservatively links all the (by-now-compiled-to-executable-code) modules
  below the current module, which is wasteful because very few of them will be
  needed.  With ExplicitLevelImports, only the modules needed at stage ``C``
  (usually a tiny fraction) need be linked at compile time.

Moreover, under ``NoImplicitStagePersistence`` it is an error to use DeriveLift on a type
unless all its definition is imported at both level 0 and level 1. This is
discussed in more detail in the "Implicit lifting and deriving Lift instances"
section.

Syntax for imports
##################

Under ``ExplicitLevelImports``, the syntax for imports becomes::

  importdecl :: { LImportDecl GhcPs }
     : 'import' maybe_src maybe_safe optlevel optqualified maybe_pkg modid optlevel optqualified maybeas maybeimpspec

  optlevel :: { LImportLevel }
     : 'splice' { SpliceLevel }
     | 'quote'  { QuoteLevel  }
     |          { NormalLevel }


The ``splice`` or ``quote`` keyword appears before either possible position for
the ``qualified`` keyword, but after any ``SOURCE`` pragma or the ``safe``
keyword.  It is an error to specify ``splice`` or ``quote`` more than once in
the same import.

For example, the following are accepted, and do not require ``ImportQualifiedPost``::

  import splice A
  import qualified A splice
  import quote qualified B as QB
  import C splice
  import qualified C splice as SC

The following are accepted provided ``ImportQualifiedPost`` is also enabled::

  import quote B qualified as QB
  import D quote qualified as QD

The following are rejected::

  import splice quote A
  import splice A splice
  import splice A quote
  import A quote quote
  import qualified splice A
  import A qualified splice


Name resolution and ambiguous variables
#######################################

Name resolution ("renaming") does not take account of the level at which an
identifier was imported when disambiguating ambiguous names, even though this is
sometimes more conservative than necessary.  For example, the following program
is rejected::

  {-# LANGUAGE ExplicitLevelImports #-}

  import A ( x )
  import splice B ( x )

  foo = $( x ) x

In this case, there is in principle no ambiguity because ``A.x`` isn't allowed
to be used in the top-level splice, and ``B.x`` isn't allowed to be used outside
the splice.  Thus the only disambiguation that will pass the type-checker is::

  foo = $( B.x ) A.x

We choose to reject this disambiguation to keep the design simple and prevent
any confusion about what is in scope. This position is conservative, and can be
relaxed in the future if more flexibility appears worthwhile. This choice
follows the `Lexical Scoping Principle <https://github.com/ghc-proposals/ghc-proposals/blob/8ad4daecc849f435af49767864b8e61b174bf252/principles.rst#221lexical-scoping-principle-lsp>`_.

A positive consequence of the current design is that if a program is accepted
with ``ExplicitLevelImports``, it will be accepted after erasing all
``splice``/``quote`` keywords and using ``ImplicitStagePersistence`` instead of
``ExplicitLevelImports``.


Exports
#######

This section amends the `exports section <https://www.haskell.org/onlinereport/haskell2010/haskellch5.html#x11-1000005.2>`_ of the Haskell Report to account for levels.

Under ``NoImplicitStagePersistence``, modules may export bindings only if they
are available at level 0. All top-level bindings are introduced at level 0,
types, data constructors, functions and so on as well as modules imported at level 0.
These things can therefore be exported from a module.

For example, the following is rejected::

  {-# LANGUAGE ExplicitLevelImports #-}

  module M (oops) where  -- Error: oops imported at level -1 but used at level 0
    import splice N ( oops )

A binding imported at multiple levels can be exported if available at level 0::

  module M (oops) where -- Accepted: oops is available at level 0.
    import splice N ( oops )
    import N ( oops )

The export item ``T(..)`` exports the subset of ``{T}`` union ``{the methods/constructors/fields of T}`` that are available at level 0. This set may be empty::

  {-# LANGUAGE ExplicitLevelImports #-}

  module M (T(..)) where  -- Warning with -Wdodgy-exports: wildcard export doesn't export any identifiers
    import splice N ( T(t) ) -- Selector `t`, also imported to splice level
    import N (T) -- Only T, not the children imported into level 0.

The export item ``module M`` exports the set of all entities that are in scope with both an unqualified name ``e`` and a qualified name ``M.e``, and are available at level 0. This set may be empty::

  {-# LANGUAGE ExplicitLevelImports #-}

  module M (module N) where  -- Only `ok` is exported, since `oops` is imported at -1
    import splice N ( oops )
    import N ( ok )

An implicit export which doesn't export any bindings issues a warning under ``-Wdodgy-exports``::

  {-# LANGUAGE ExplicitLevelImports #-}

  module M (module N) where  -- Warning: module N doesn't export any identifiers
    import splice N ( oops )


Class instance resolution
#########################

Class instances carry a level, much like identifiers, and must be used at the
correct level.  This will be enforced by the type-checker under
``NoImplicitStagePersistence``:

* Instance resolution views the set of instances from all imports together and thus
  instances from normal and splice imports must agree with each other.

* After instance resolution has selected an instance, it is checked which levels
  the instance is available at and an error is raised if the instance is not available
  at the correct level.

* Instances defined in the current modules are at level 0, just like top-level
  variable definitions in a module.

This design for instances mirrors the situation for name resolution. As with
ambiguous names, it would in principle be possible for the type-checker to make
use of level information to accept more programs, but this seems like an
undesirable level of complexity.  Thus the following example is rejected::

  module X where
    data X = MkX

  module Normal where
    import X
    instance Show X where show _ = "normal"

  module Splice where
    import X
    instance Show X where show _ = "splice"

  module Bottom where
    import X (X(..))
    import splice X (X(..))
    import Normal ()        -- imports instance Show X at level 0
    import splice Splice () -- imports a different instance Show X at level -1

    s1 = show MkX -- Error: overlapping instances defined in ``Normal`` and ``Splice``

However the following is accepted::

  module X where
    data X = MkX deriving Show

  module Bottom where
    import X (X(..))        -- imports instance Show X at level 0
    import splice X (X(..)) -- imports the same instance Show X at level -1
    import splice Language.Haskell.TH.Lib ( stringE )

    s1 = show MkX                 -- Uses instance at level 0
    s2 = $( stringE (show MkX) )  -- Uses instance at level -1


Exports of class instances
##########################

Only instances available at level 0 are re-exported from a module.  For example,
the following is rejected::

  module X where
    data X = MkX

  module Splice where
    import X
    instance Show X where show _ = "splice"

  module Y where
    import splice Splice () -- imports instance Show X at level -1

  module Bottom where
    import X (X(..))
    import Y ()

    s1 = show MkX -- Error: no instance for Show X

Even though ``Y`` has access to the instance at level -1, it does not re-export it.  Thus ``Bottom`` does not import the instance.

This is necessary for a clean separation between stages, because instances may exist only at compile-time or only at runtime, just like identifiers.



Examples
========

Splice imports
##############

A "splice" import is prefixed with ``splice``. In this example, identifiers from
``A`` can be used only in top-level splices and identifiers from ``B``
cannot be used in quotes or splices::

  {-# LANGUAGE ExplicitLevelImports #-}
  {-# LANGUAGE TemplateHaskell #-}
  module Main where

  import splice A (foo)  -- foo :: Int -> Q Exp
  import B (bar)         -- bar :: Int -> Q Exp

  x = $(foo 25) -- Accepted
  y = $(bar 33) -- Error: bar imported at level 0 but used at level -1

Thus:

1. When compiling module ``Main``, even though ``TemplateHaskell`` is enabled,
   only identifiers from module ``A`` will be used in top-level splices so
   only ``A`` (and its dependencies) needs to compiled to object code before starting to compile ``Main``.
2. When cross-compiling, ``A`` needs to be built only for the host and ``B``
   only for the target.


Splice imports example: ``printf``
##################################

Let ``printf :: String -> Q Exp`` be defined in ``Printf``, such that the
arguments received by ``printf`` applied to a formatting string is determined at
compile time based on the format specifiers within the string::

    $(printf "Error: %s on line %d") "test" 123 :: String

The following program is rejected::

    {-# LANGUAGE ExplicitLevelImports #-}

    import Printf (printf)

    -- Error: printf imported at level 0 but used at level -1
    x = $(printf "Error: %s on line %d") "test" 123 :: String

because ``printf`` was imported "normally" at the default level 0 and thus
cannot occur within a top-level splice (at level -1). For this program to be
level-correct, ``printf`` must be imported at level -1 to be used within a
top-level splice::

    {-# LANGUAGE ExplicitLevelImports #-}

    import splice Printf (printf)

    -- accepted!
    x = $(printf "Error: %s on line %d") "test" 123 :: String

Splice-importing ``Printf`` makes it clear to both humans and compilers that
``printf`` will only be required at compile time, since it will only be used within top-level splices.



Quote imports
#############

A "quote" import is prefixed with ``quote``.  In this example, identifiers from
``A`` can be used **only** in quotes, while identifiers from ``B`` **cannot** be
used in quotes or splices::

  {-# LANGUAGE ExplicitLevelImports #-}
  {-# LANGUAGE TemplateHaskell #-}
  module Main where

  import quote A (foo)  -- foo :: Int -> Int
  import B (bar)        -- bar :: Int -> Int

  x = [| foo 25 |] -- Accepted
  y = [| bar 33 |] -- Error: bar imported at level 0 but used at level 1

When a quote such as ``x = [| foo 25 |]`` is spliced, i.e. ``z = $(x)``,
its contents will be needed to execute the program at runtime (``z = foo 25``,
so evaluating ``z`` at runtime requires ``foo`` to be available).



Module Stages
#############

Modules are compiled at a specific stage. Levels within a module are interpreted
as offsets to the specific stage at which the module is being compiled.
Stages are an application of the proposal which a levelled language makes possible,
but a levelled langauge does imply a specific stage structure which we leave
to future work.


For example, suppose we have just two stages, so a module is either compiled for
compile time (*C*) or runtime (*R*), with *C* before *R*. Then:

* The main module is compiled for ``R``.

* A normal import does not shift the stage at which the dependent module is required.

* If a module ``M`` splice imports module ``A``, then compiling ``M`` at stage
  *R* requires compiling module ``A`` at stage *C*.

* If a module ``N`` quote imports module ``B``, then compiling ``N`` at stage
  *C* requires compiling module ``B`` at stage *R*.

In general, the implementation may choose to support any number of stages. A
single stage would require that all modules must be compiled such that they can
be executed during compilation of subsequent modules, as well as at runtime.
More than two stages are possible to imagine in some cross-compilation
scenarios. By far the most common case is two stages.  However, the
specification is expressed in terms of level offsets rather than stages in order
to keep the language design abstract rather than overfitting to a particular
arrangement of stages.

The compiler can then choose appropiately how modules needed at ``C`` are compiled
and how modules needed at ``R`` are compiled.

For example:

* In ``-fno-code`` mode, ``C`` modules may be compiled in dynamic way, but ``R`` modules
  are not compiled at all.
* When using a profiled GHC. ``C`` modules must be compiled in profiled way but ``R`` modules
  will be compiled in static way.

Further level structure as needed by cross-compilation settings may require more stages.
This will be easily possible to change once the level discipline is enforced.

The order than modules are compiled depends on normal import dependencies. Before
you can compile a module, you must compile all modules you depend on for the appropiate
stages. For example, you may compile some modules for compile-time and some for runtime.
The idea of a stage relates to when the compiled code is run. Modules compiled for
compile-time will all be executed and run before any runtime modules are evaluated.


At the moment, GHC has a basic notion of stages, for example when using ``-fno-code``, only
modules which are dependencies of modules which enable ``TemplateHaskell`` are compiled but
the concept is not very precise yet.

Cabal and the rest of the ecosystem does not yet understand stages. This is left
to future work and will be necessary for Cabal to support cross-compilation properly.


Module stage offsetting example
###############################

The interaction between stages and level offsetting can be understood more clearly through an example.
Module ``A`` splices ``foo`` from module ``B`` which both quotes ``bar`` from module ``C`` and uses ``baz`` from ``D``::

    {-# LANGUAGE ExplicitLevelImports #-}
    {-# LANGUAGE TemplateHaskell #-}
    module A where
    import splice B (foo)

    -- foo can be used within a splice (level -1) because of the splice import (-1).
    x = $(foo 10)


    {-# LANGUAGE ExplicitLevelImports #-}
    {-# LANGUAGE TemplateHaskell #-}
    module B where
    import D (baz)
    import quote C (bar)

    -- bar can be used within a quote (level +1) because of the quote import (+1)
    foo x
      | baz x = [| bar * 2 |]
      | otherwise = [| bar |]

    module C where
    bar = 42

    module D where
    baz 0 = True
    baz _ = False


Now consider compiling ``A`` at stage *R*.

* ``B`` is required at stage *C*, as it is splice imported from ``A`` at *R*.
* ``C`` is required at stage *R*, as it is quote imported from ``B`` at *C*.
* ``D`` is required at stage *C*, as it is normally imported from ``B`` at *C*.

Therefore in order to compile ``A`` at *R*, we have performed
dependency resolution and require ``B`` at *C*, ``C`` at *R* and ``D`` at *C*.

The perhaps curious case is ``D``: is it needed at compile-time or runtime? It
does not use a splice import, so one could think it is needed at runtime -- but
here is where the distinction between the import level offset and base stage is
relevant. ``D`` is only being imported as a dependency of ``B``, which is at *C*
stage. This makes ``D`` *also* at the *C* stage! Note how ``baz`` is needed
at compile time just to define ``foo``, which is properly ``splice`` imported.

The levels of all modules in the transitive closure of a ``splice``-imported
module are offset by -1. Conversely, ``quote`` imports offset the levels by +1,
thereby making all the levels align correctly.

Implicit Stage Persistence and Stages
#####################################

Modules using implicit stage persistence place a set of strong requirements on itself and
immediate dependencies. Consider this example where module ``B`` uses ``ImplicitStagePersistence``::

  module A where { a = 1 :: Int }

  {-# LANGUAGE ExplicitLevelImports #-}
  {-# LANGUAGE ImplicitStagePersistence #-}
  module B where
  import A

  foo = a

  bar = [| foo |]

  {-# LANGUAGE ExplicitLevelImports #-}
  module C where
  import splice B
  c :: Int
  c = $(bar)

Consider compiling ``C @ R``, when ``bar`` from ``B`` is executed, then
it will produce a program ``B.foo``. Therefore we will also need ``B @ R``.

How could we determine from the module header that we would require ``B @ R``?

* ``C @ R`` splice imports ``B``, therefore only directly places a requirement on ``B @ C``
* However, ``B`` enables ``ImplicitStagePersistence``, and therefore is able to persist
  top-level definitions and definitions defined in ``B`` itself and all its level 0 or level 1 imports. Therefore we
  determine we also require ``C @ R``.


In this example you can observe that the ability to move a variable between
levels using cross-stage persistence places a strong set of requirements on the
stages that modules are required at. Implicit stage pesistence makes imported
identifiers available at all levels, as a consequence, they must also be available
at all stages. The introduction of the ``ImplicitStagePersistence``
extension is wholly motivated by the desire to control these requirements in an explicit
fashion.

ImplicitStagePersistence, Stages and TemplateHaskellQuotes
##########################################################

A more refined specification is possible if you observe that ``TemplateHaskellQuotes``
can only persist identifiers forwards. Therefore if you have ``ImplicitStagePersistence``
in a module where ``TemplateHaskellQuotes`` is enabled then you place a requirement
that you need the module and immediate dependencies at current and future stages
but not previous stages.

Consider this example, under the revised rule::

  {-# LANGUAGE TemplateHaskellQuotes, ImplicitStagePersistence #-}
  module M1 where
    data T = MkT Int
    instance Lift T where
      lift (MkT n) = [| MkT $(lift n) |]
  {-# LANGUAGE ExplicitSpliceImports #-}
  module M2 where
    import M1
    foo = MkT

If we require ``M2 @ R``:

* We require ``M1 @ R`` due to the ``import M1`` declaration.
* ``M2 @ R`` enables ``ImplicitStagePersitence`` and ``TemplateHaskellQuotes``
  so therefore places a requirement on compiling ``M2 @ R``.

If ``TemplateHaskell`` was enabled, we would also require ``M2 @ C`` because
``TemplateHaskell`` allows you to write a -1 context, and hence persist identifiers
to negative as well as positive levels.


Effect and Interactions
=======================

Case Study: ``pandoc``
######################

The `pandoc <https://hackage.haskell.org/package/pandoc>`_ library is a medium-sized package that
contains approximately 200 modules. It uses ``TemplateHaskell`` in a light manner in order to embed
some data files and derive some JSON instances.

Modifying the package to use ``ExplicitLevelImports`` required little effort
and involved `modifying the imports of the 5 modules <https://github.com/mpickering/pandoc/commit/ce57269b2c6ec894a2389069362ea39b06b5c413>`_ in the project which use ``TemplateHaskell``.

Now when the project is loaded into GHCi using the ``-fno-code`` option, the recompile
time is halved as no modules from the library itself need to be compiled. Before,
the ``Text.Pandoc.App.Opt`` module caused the majority of modules to be needlessly
compiled as it used ``TemplateHaskell`` and is near the root of the module graph.

It can also be easily observed from looking at the imports that

* No modules from the ``pandoc`` library are used in compile-time evaluation.
* Only a few external packages are involved in compile-time evaluation.

This information can be used by the driver in order to simplify the compilation pipeline.

Typed Template Haskell
######################

Typed Template Haskell (TTH) is an extension of Template Haskell that allows
using type-safe staged programming for program optimisation.  (Its typical use
cases are rather different from untyped TH, since in particular it does not
support declaration splices.)

The same level checks are implemented for typed brackets as untyped brackets.
In particular, when using TTH and explicit level imports, you can introduce
stage errors which you can't fix. Currently the following program is accepted::

  foo :: Show a => Code Q (a -> String)
  foo = [|| show ||]

However, there is actually a stage error introduced by this program as the
evidence for ``Show a`` is bound earlier than it is used.
The prototype correctly reports the following error::

  TTH.hs:8:11: error: [GHC-28914]
      • Stage error: ‘show’ is bound at stage {0} but used at stage 1
        From imports {imported from ‘Prelude’ at TTH.hs:3:8-11}
      • In the Template Haskell typed quotation [|| show ||]
    |
  8 | foo = [|| show ||]
    |


The language of constraints is not yet expressive enough to communicate that we
want the ``Show a`` evidence to be available at a later stage. Fixing this problem
will require
significant additional effort, and there are other known issues with TTH (see
`Staging with Class: a Specification for Typed Template Haskell
<https://dl.acm.org/doi/abs/10.1145/3498723>`_). We propose that an initial
implementation of ``NoImplicitStagePersistence`` may support untyped TH but not
TTH (i.e. the compiler may reject programs using TTH under
``NoImplicitStagePersistence``).  In the long term, we believe that implementing
Staging with Class is desirable and consistent with the direction of travel
established by this proposal, but the full details of Staging with Class are out
of scope.


Implicit lifting and deriving ``Lift`` instances
################################################
.. _lift_instances:

``Lift`` instances are used to provide serialisation-based cross-stage
persistence.  For example, a typical ``Lift`` instance looks like::

    data MInt = Some Int | None

    instance Lift MInt where
        lift :: MInt -> Q Exp
        lift None     = [| None |]
        lift (Some x) = [| Some $(lift x) |]

The presence of this instance means the following declaration will be accepted::

  foo :: MInt -> Q Exp
  foo x = [| x |]  -- implicitly becomes [| $(lift x) |]

Defining a ``Lift`` instance requires the datatype constructors to be available
both at compile-time and runtime, so defining ``Lift`` within the same module as
the datatype itself requires path-based cross-stage persistence.  Operationally,
``None`` and ``Some`` are needed both at compile-time *and*  runtime since they
are both matched on at compile time, and also persisted to be spliced in the
future into a program that can make use of them at runtime. As a result, it
isn't possible to define or derive a (non-orphan) ``Lift`` instance under
``NoImplicitStagePersistence``.

An orphan ``Lift`` instance can be defined thus::

  module M where
    data MInt = Some Int | None

  module N where
    import M
    import quote M

    instance Lift MInt where
        lift :: MInt -> Q Exp
        lift None     = [| None |]
        lift (Some x) = [| Some $(lift x) |]

This isn't technically problematic, rather it is just a result of what ``Lift``
means. However, it means some users may need to modify their use of ``Lift``
instances if they wish to benefit more from ``NoImplicitStagePersistence``.
Users are free to use ``ImplicitStagePersistence`` in selected modules to allow
defining ``Lift`` instances, but doing so means all the dependencies of the
module will need to be available both at compile-time and runtime.

As a general rule, ``Lift`` instances should be defined only for simple
datatypes near the root of the module hierarchy of an application.

Just as ``NoImplicitStagePersistence`` allows users to disable implicit
path-based cross-stage persistence, it would make sense to have an extension
flag to disable implicit lifting (serialisation-based persistence).  This would
allow the programmer to ensure they are explicit about where calls to ``lift``
occur in their programs, which is sometimes desirable when using staging for
runtime performance.  We intend to bring forward a separate proposal for this,
as it is otherwise orthogonal to the current proposal.


Implicit ``Prelude`` imports
############################

``Prelude`` does not get implicitly imported with ``splice`` or ``quote``. Therefore
if you wish to use definitions from your ``Prelude`` module at non-zero levels
then you have to explicitly import it at that level.

A ``splice`` or ``quote`` import of ``Prelude`` does not cause the implicit
``Prelude`` import to be suppressed (unlike a normal explicit import of
``Prelude``).

For example, the following is accepted, but would be rejected if the ``import
splice Prelude`` line was removed::

  {-# LANGUAGE TemplateHaskell #-}
  {-# LANGUAGE ExplicitLevelImports #-}

  import splice Prelude

  foo = null $(id [|"foo"|])

Here ``id`` is available at level -1 thanks to ``import splice Prelude``, and
``null`` is available at level 0 thanks to the implicit ``Prelude`` import.



Costs and Drawbacks
===================

* The user has to be aware of the significance of using splice imports.

  The compile-time and cross-compilation benefits only
  available if users switch on the extensions.  In simple use cases (e.g.
  ``makeLenses``) it should be easy enough for users to write ``import splice``,
  but more complex cases are more complex.


* Since the mechanism to control the levels of binders is *module-granular*,
  code in certain situations is necessary to be defined across two modules, for
  instance, the following was previously accepted under ``ImplicitStagePersistence``::

    module M where
      data B = MkB
      x = [| MkB |]

  However to be level-correct with ``NoImplicitStagePersistence`` it needs to be
  split over two modules::

    module M where
      import quote N
      x = [| MkB |]

    module N where
      data B = MkB

  This is particularly an issue for code defining ``Lift`` instances, as
  discussed above.


Backward Compatibility
======================

Since ``ImplicitStagePersistence`` is enabled by default, this proposal is
backwards compatible.  Existing programs will continue to work unchanged, though
they may not benefit from available performance improvements.

Were ``NoImplicitStagePersistence`` to become the default in a future language
edition, this would be a breaking change, but we do not propose this pending
implementation and experience with the feature.


Alternatives
============



Multiple levels within a single module
######################################

One possible design that mitigates the need for module-level granularity of
imports, inspired by the Racket and `MacoCaml <https://dl.acm.org/doi/pdf/10.1145/3607851>`_ languages, is the introduction of an
additional ``macro`` keyword that introduces bindings at a different level.
A ``macro`` annotated binding would introduce a binding at the -1 level, without
requiring it to be ``splice`` imported from a different module.

The current proposal doesn't include such a change for two reasons:

* First, our proposed design lays out the foundation for well-staged programs,
  and is forward-compatible/can be readily extended with such a ``macro``
  keyword.  Tentatively, the implementation could amount to splitting ``macro``
  bindings from non ``macro`` ones and elaborate the two sets of bindings into
  separate modules that use ``splice`` imports (and then GHC would handle them
  as described by this proposal).

* Second, a design for local modules (see `proposal #283
  <https://github.com/ghc-proposals/ghc-proposals/pull/283>`_) could provide all
  the convenience of the ``macro`` keyword without the need for additional
  language complexity.


Level-correct package dependencies
##################################

The splice and quote imports in this proposal make it possible to express which
module dependencies are required at which stages.  Ultimately, it would make
sense to expose this distinction at the level of Cabal packages, so that Cabal
could build package dependencies only for the stages at which they are required.
This would primarily be of value in cross-compilation scenarios.

In the interests of keeping the work manageable, changes to Cabal are out of
scope for the current proposal, but we believe this proposal lays a foundation
for future work to improve Cabal's cross-compilation support.


Imports with explicit level numbers
###################################

The current proposal permits imports only at levels -1, 0 or 1. This means it is
not possible to introduce a binding for use in a splice contained within another
splice, which would require it to be at level -2.  (Note that nested quotes are
in any case not supported in GHC due to a separate restriction.)

An alternative would be to allow even finer grained control of splice imports so
that usage at level -2 or lower could be distinguished. This could be useful in
some cross-compilation situations. This is the approach suggested in the `Stage
Hygiene for Template Haskell proposal
<https://github.com/ghc-proposals/ghc-proposals/pull/243>`_.

The syntax in this proposal could be extended in a natural way to allow for this by adding an optional
integer component which specifies precisely what level the imported names should be allowed at::

    -- Can be used at level -1
    import splice 1 A
    -- Can be used at level -2
    import splice 2 A

Practically, by far the most common situation is a single level of splices, so in the interests
of reducing complexity we do not propose supporting this at present.


Inferring splice/quote imports based on usage
#############################################

Since our proposed approach has the type-checker verify that usage of ``splice``
or ``quote`` imports is correct, it may be possible in principle to infer where
``splice`` or ``quote`` keywords are needed, based on usage inside a module.
However, this would compromise the principle that the build system can discover
the dependencies for a module just by looking at the import list in the module
header. Achieving the performance benefits of our proposed approach would
involve significant technical complexity (as the compiler would need to
partially type-check a module, then suspend compilation of that module while it
compiles those of its dependencies determined to be required for further
type-checking).

Given that the ``splice`` and ``quote`` annotations are useful for human readers
understanding how code is staged, it seems worthwhile to make them explicit.

Of course, nothing prevents development of a tool that helps users insert
``splice`` and ``quote`` annotations into their modules as part of a migration
to using ``ExplicitLevelImports``.


Syntactic alternatives
######################

There are several proposals for the syntax of explicit level imports:

* The current iteration of this proposal allows the splice/quote keyword to be
  placed before or after the module name, like qualified imports under
  ``ImportQualifiedPost`` (see `proposal #190
  <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0190-module-qualified-syntax.rst>`_).
  This allows users to choose their preferred position for the keywords.  We
  could be more restrictive here, but would need to agree on a single position.

* Using a pragma rather than a syntactic modifier would fit in better with
  how ``SOURCE`` imports work and make writing backwards compatible code easier::

    import {-# SPLICE #-} B

* Some have objected that the ``import splice`` suggestion is ungrammatical,
  unlike ``import qualified`` or ``import hiding``.

  One possible alternative is ``$(import Foo)`` to represent a splice import, but this
  syntax clashes with the existing syntax for declaration splices and significantly
  changes the structure of the import syntax.

  Another alternative suggested was ``import for splice``, which restores the
  grammatical nature of the import.

* The keywords ``splice`` and ``quote`` are different lengths, which interferes
  with alignment.  Alternatively ``quote`` could be replaced with ``quoted``,
  which is the same length as ``splice``.

* The syntax does not provide a way to explicitly import at level 0; this is
  indicated by the absence of a keyword. We could add a keyword for this, e.g.
  ``default`` or ``target`` (although neither of these are ideal). It would also
  be possible for a single import to refer to multiple levels simultaneously,
  e.g. ``import M default, splice`` or
  ``import Prelude qualified splice as SP (id, ($)), quote as QP (const), default (..)``.

* Modifier syntax (see `proposal #370
  <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0370-modifiers.rst>`_)
  could be used, although it would seem inconsistent with the existing syntax
  that mainly uses keywords (except for `{-# SOURCE #-}` imports).


Implicit Splice/Quote Prelude imports
#####################################

In the proposal ``Prelude`` must be imported explicitly at non-zero levels.

Another possible design would be to automatically import ``Prelude`` at all
levels rather than just level 0.

For us, it is undesirable to automatically add these additional imports and
hence dependencies on certain stages unless they were actually used.

An implicit ``Prelude`` import will require the package which provides
``Prelude`` to be compiled for all stages, whether it is used or not. This may
cause a programmer a problem if there are subtlties about compiling their Prelude
for a particular stage.

In this case, we would then also need a design about how to turn off the specific
imports. Writing ``import quote Prelude ()`` is not sufficient, because the module
will still depend on ``Prelude`` at a particular stage. The programmer would
have to enable ``NoImplicitPrelude`` in their library to turn off all ``Prelude``
imports, before manually adding them back. They would have to enable this in all
modules as well, lest a sneaky implicit import suddently adds back a ``Prelude``
dependency at an undesired stage.

Therefore it seems more in spirit with the proposal to make programmers depend
explicitly on a prelude at different levels if they want to do so.





Other alternatives
##################

* The extension could apply only to "home" modules (those from the package being compiled), because the primary benefits of
  splice imports are when using GHC's ``--make`` mode. As the proposal stands,
  for uniformity, any module used inside a top-level splice must be marked as
  a splice import, even if it's from an external package.

* Since ``ExplicitLevelImports`` is essentially useless when
  ``TemplateHaskell`` is disabled, we could have ``ExplicitLevelImports`` imply
  ``TemplateHaskell``.  There is at least one case where this would be harmful:
  users may wish to enable ``ExplicitLevelImports`` globally for their
  project, but only carefully enable ``TemplateHaskell`` for a small number of
  modules. ``TemplateHaskell`` has the effect of enabling code generation for
  a modules dependencies, so it is normally advisable to be explicit about which
  modules use the feature.

* ``NoImplicitStagePersistence`` is a "negative" extension, in that it requires
  a user to opt in but removes a feature from the language, much like
  ``NoFieldSelectors``. This could be confusing; but it seems less confusing
  than having a positive extension impose an additional restriction.

* We could consider disallowing a package quoting modules from itself and
  restrict quoting to modules imported from *different* packages. The problem
  with self quoting is that we lose some granularity regarding what exactly is
  needed at compile-time and runtime. By requiring users to specify the runtime
  dependencies in a different package we get a better compile-time vs runtime
  distinction which benefits our motivation.
  On the other hand, it's quite unfortunate to require having yet another
  package just for TH, and may drive away adoption.


Unresolved Questions
====================

None.


Implementation Plan
===================

Matthew has implemented a `prototype <https://gitlab.haskell.org/ghc/ghc/-/tree/wip/splice-imports-2024?ref_type=heads>`_.


Acknowledgements
================

Work on this proposal and its implementation was carried out by `Well-Typed
<https://well-typed.com/>`_ thanks to funding from `Mercury
<https://mercury.com>`_.
