---
layout: docs
title:  "Hierarchy Cookbook"
section: "chisel3"
---

# Hierarchy Cookbook

* [How do I instantiate multiple instances with the same module parameterization, but avoid re-elaboration?](#how-do-i-instantiate-multiple-instances-with-the-same-module-parameterization)
* [How do I access internal fields of an instance?](#how-do-i-access-internal-fields-of-an-instance)
* [How do I make my parameters accessable from an instance?](#how-do-i-make-my-parameters-accessable-from-an-instance)
* [How do I reuse a previously elaborated module, if my new module has the same parameterization?](#how-do-i-reuse-a-previously-elaborated-module-if-my-new-module-has-the-same-parameterization)
* [How do I parameterize a module by its children instances?](#how-do-I-parameterize-a-module-by-its-children-instances)
* [How do I use the new hierarchy-specific Select functions?](#how-do-I-use-the-new-hierarchy-specific-Select-functions)

## How do I instantiate multiple instances with the same module parameterization?

Prior to this package, Chisel users relied on deduplication in a FIRRTL compiler to combine
structurally equivalent modules into one module (aka "deduplication").
This package introduces the following new APIs to enable multiply-instantiated modules directly in Chisel.

`Definition(...)` enables elaborating a module, but does not actually instantiate that module.
Instead, it returns a `Definition` class which represents that module's definition.

`Instance(...)` takes a `Definition` and instantiates it, returning an `Instance` object.

`Instantiate(...)` provides an API similar to `Module(...)`, except it uses
`Definition` and `Instance` to only elaborate modules once for a given set of
parameters. It returns an `Instance` object.

Modules (classes or traits) which will be used with the `Definition`/`Instance` api should be marked
with the `@instantiable` annotation at the class/trait definition.

To make a Module's members variables accessible from an `Instance` object, they must be annotated
with the `@public` annotation. Note that this is only accessible from a Scala sense—this is not
in and of itself a mechanism for cross-module references.

### Using Definition and Instance

In the following example, use `Definition`, `Instance`, `@instantiable` and `@public` to create
multiple instances of one specific parameterization of a module, `AddOne`.

```scala
import chisel3._
import chisel3.experimental.hierarchy.{Definition, Instance, instantiable, public}

@instantiable
class AddOne(width: Int) extends Module {
  @public val in  = IO(Input(UInt(width.W)))
  @public val out = IO(Output(UInt(width.W)))
  out := in + 1.U
}

class AddTwo(width: Int) extends Module {
  val in  = IO(Input(UInt(width.W)))
  val out = IO(Output(UInt(width.W)))
  val addOneDef = Definition(new AddOne(width))
  val i0 = Instance(addOneDef)
  val i1 = Instance(addOneDef)
  i0.in := in
  i1.in := i0.out
  out   := i1.out
}
```
```verilog
// Generated by CIRCT firtool-1.43.0
module AddOne(	// <stdin>:3:10
  input  [9:0] in,	// hierarchy.md:16:23
  output [9:0] out	// hierarchy.md:17:23
);

  assign out = in + 10'h1;	// <stdin>:3:10, hierarchy.md:18:13
endmodule

module AddTwo(	// <stdin>:13:10
  input        clock,	// <stdin>:14:11
               reset,	// <stdin>:15:11
  input  [9:0] in,	// hierarchy.md:23:15
  output [9:0] out	// hierarchy.md:24:15
);

  wire [9:0] _i0_out;	// hierarchy.md:26:20
  AddOne i0 (	// hierarchy.md:26:20
    .in  (in),
    .out (_i0_out)
  );
  AddOne i1 (	// hierarchy.md:27:20
    .in  (_i0_out),	// hierarchy.md:26:20
    .out (out)
  );
endmodule

```

### Using Instantiate

Similar to the above, the following example uses `Instantiate` to create
multiple instances of `AddOne`.

```scala
import chisel3.experimental.hierarchy.Instantiate

class AddTwoInstantiate(width: Int) extends Module {
  val in  = IO(Input(UInt(width.W)))
  val out = IO(Output(UInt(width.W)))
  val i0 = Instantiate(new AddOne(width))
  val i1 = Instantiate(new AddOne(width))
  i0.in := in
  i1.in := i0.out
  out   := i1.out
}
```
```verilog
// Generated by CIRCT firtool-1.43.0
module AddOne(	// <stdin>:3:10
  input  [15:0] in,	// hierarchy.md:16:23
  output [15:0] out	// hierarchy.md:17:23
);

  assign out = in + 16'h1;	// <stdin>:3:10, hierarchy.md:18:13
endmodule

module AddTwoInstantiate(	// <stdin>:13:10
  input         clock,	// <stdin>:14:11
                reset,	// <stdin>:15:11
  input  [15:0] in,	// hierarchy.md:47:15
  output [15:0] out	// hierarchy.md:48:15
);

  wire [15:0] _i0_out;	// hierarchy.md:49:23
  AddOne i0 (	// hierarchy.md:49:23
    .in  (in),
    .out (_i0_out)
  );
  AddOne i1 (	// hierarchy.md:50:23
    .in  (_i0_out),	// hierarchy.md:49:23
    .out (out)
  );
endmodule

```

## How do I access internal fields of an instance?

You can mark internal members of a class or trait marked with `@instantiable` with the `@public` annotation.
The requirements are that the field is publicly accessible, is a `val` or `lazy val`, and is a valid type.
The list of valid types are:

1. `IsInstantiable`
2. `IsLookupable`
3. `Data`
4. `BaseModule`
5. `Iterable`/`Option `containing a type that meets these requirements
6. Basic type like `String`, `Int`, `BigInt` etc.

To mark a superclass's member as `@public`, use the following pattern (shown with `val clock`).

```scala
import chisel3._
import chisel3.experimental.hierarchy.{instantiable, public}

@instantiable
class MyModule extends Module {
  @public val clock = clock
}
```

You'll get the following error message for improperly marking something as `@public`:

```scala
import chisel3._
import chisel3.experimental.hierarchy.{instantiable, public}

object NotValidType

@instantiable
class MyModule extends Module {
  @public val x = NotValidType
}
// error: @public is only legal within a class or trait marked @instantiable, and only on vals of type Data, BaseModule, MemBase, IsInstantiable, IsLookupable, or Instance[_], or in an Iterable, Option, Either, or Tuple2
// val x = circt.stage.ChiselStage.emitCHIRRTL(new Top)
//     ^
```

## How do I make my parameters accessible from an instance?

If an instance's parameters are simple (e.g. `Int`, `String` etc.) they can be marked directly with `@public`.

Often, parameters are more complicated and are contained in case classes.
In such cases, mark the case class with the `IsLookupable` trait.
This indicates to Chisel that instances of the `IsLookupable` class may be accessed from within instances.

However, ensure that these parameters are true for **all** instances of a definition.
For example, if our parameters contained an id field which was instance-specific but defaulted to zero,
then the definition's id would be returned for all instances.
This change in behavior could lead to bugs if other code presumed the id field was correct.

Thus, it is important that when converting normal modules to use this package,
you are careful about what you mark as `IsLookupable`.

In the following example, we added the trait `IsLookupable` to allow the member to be marked `@public`.

```scala
import chisel3._
import chisel3.experimental.hierarchy.{Definition, Instance, instantiable, IsLookupable, public}

case class MyCaseClass(width: Int) extends IsLookupable

@instantiable
class MyModule extends Module {
  @public val x = MyCaseClass(10)
}

class Top extends Module {
  val inst = Instance(Definition(new MyModule))
  println(s"Width is ${inst.x.width}")
}
```
```
Width is 10
Circuit(Top,List(DefModule(repl.MdocSession$MdocApp5$MyModule@47251468,MyModule,List(Port(MyModule.clock: IO[Clock],Input,UnlocatableSourceInfo), Port(MyModule.reset: IO[Reset],Input,UnlocatableSourceInfo)),Vector()), DefModule(repl.MdocSession$MdocApp5$Top@65305369,Top,List(Port(Top.clock: IO[Clock],Input,UnlocatableSourceInfo), Port(Top.reset: IO[Bool],Input,UnlocatableSourceInfo)),Vector(DefInstance(SourceLine(hierarchy.md,112,22),ModuleClone(repl.MdocSession$MdocApp5$MyModule@47251468),List(Port(MyModule.clock: IO[Clock],Input,UnlocatableSourceInfo), Port(MyModule.reset: IO[Reset],Input,UnlocatableSourceInfo))), Connect(SourceLine(hierarchy.md,112,22),Node(MyModule.inst.clock: IO[Clock]),Node(Top.clock: IO[Clock])), Connect(SourceLine(hierarchy.md,112,22),Node(MyModule.inst.reset: IO[Reset]),Node(Top.reset: IO[Bool]))))),List(),firrtl.renamemap.package$MutableRenameMap@7c8288c7,List())
```

## How do I look up parameters from a Definition, if I don't want to instantiate it?

Just like `Instance`s, `Definition`'s also contain accessors for `@public` members.
As such, you can directly access them:

```scala
import chisel3._
import chisel3.experimental.hierarchy.{Definition, instantiable, public}

@instantiable
class AddOne(val width: Int) extends RawModule {
  @public val width = width
  @public val in  = IO(Input(UInt(width.W)))
  @public val out = IO(Output(UInt(width.W)))
  out := in + 1.U
}

class Top extends Module {
  val definition = Definition(new AddOne(10))
  println(s"Width is: ${definition.width}")
}
```

```verilog
// Generated by CIRCT firtool-1.43.0
module Top(	// <stdin>:11:10
  input clock,	// <stdin>:12:11
        reset	// <stdin>:13:11
);

endmodule

```

## How do I parameterize a module by its children instances?

Prior to the introduction of this package, a parent module would have to pass all necessary parameters
when instantiating a child module.
This had the unfortunate consequence of requiring a parent's parameters to always contain the child's
parameters, which was an unnecessary coupling which lead to some anti-patterns.

Now, a parent can take a child `Definition` as an argument, and instantiate it directly.
In addition, it can analyze the parameters used in the definition to parameterize itself.
In a sense, now the child can actually parameterize the parent.

In the following example, we create a definition of `AddOne`, and pass the definition to `AddTwo`.
The width of the `AddTwo` ports are now derived from the parameterization of the `AddOne` instance.

```scala
import chisel3._
import chisel3.experimental.hierarchy.{Definition, Instance, instantiable, public}

@instantiable
class AddOne(val width: Int) extends Module {
  @public val width = width
  @public val in  = IO(Input(UInt(width.W)))
  @public val out = IO(Output(UInt(width.W)))
  out := in + 1.U
}

class AddTwo(addOneDef: Definition[AddOne]) extends Module {
  val i0 = Instance(addOneDef)
  val i1 = Instance(addOneDef)
  val in  = IO(Input(UInt(addOneDef.width.W)))
  val out = IO(Output(UInt(addOneDef.width.W)))
  i0.in := in
  i1.in := i0.out
  out   := i1.out
}
```
```verilog
// Generated by CIRCT firtool-1.43.0
module AddOne(	// <stdin>:3:10
  input  [9:0] in,	// hierarchy.md:184:23
  output [9:0] out	// hierarchy.md:185:23
);

  assign out = in + 10'h1;	// <stdin>:3:10, hierarchy.md:186:13
endmodule

module AddTwo(	// <stdin>:13:10
  input        clock,	// <stdin>:14:11
               reset,	// <stdin>:15:11
  input  [9:0] in,	// hierarchy.md:193:15
  output [9:0] out	// hierarchy.md:194:15
);

  wire [9:0] _i0_out;	// hierarchy.md:191:20
  AddOne i0 (	// hierarchy.md:191:20
    .in  (in),
    .out (_i0_out)
  );
  AddOne i1 (	// hierarchy.md:192:20
    .in  (_i0_out),	// hierarchy.md:191:20
    .out (out)
  );
endmodule

```

## How do I use the new hierarchy-specific Select functions?

Select functions can be applied after a module has been elaborated, either in a Chisel Aspect or in a parent module applied to a child module.

There are seven hierarchy-specific functions, which (with the exception of `ios`) either return `Instance`'s or `Definition`'s:
 - `instancesIn(parent)`: Return all instances directly instantiated locally within `parent`
 - `instancesOf[type](parent)`: Return all instances of provided `type` directly instantiated locally within `parent`
 - `allInstancesOf[type](root)`: Return all instances of provided `type` directly and indirectly instantiated, locally and deeply, starting from `root`
 - `definitionsIn`: Return definitions of all instances directly instantiated locally within `parent`
 - `definitionsOf[type]`: Return definitions of all instances of provided `type` directly instantiated locally within `parent`
 - `allDefinitionsOf[type]`: Return all definitions of instances of provided `type` directly and indirectly instantiated, locally and deeply, starting from `root`
 - `ios`: Returns all the I/Os of the provided definition or instance.

To demonstrate this, consider the following. We mock up an example where we are using the `Select.allInstancesOf` and `Select.allDefinitionsOf` to annotate instances and the definition of `EmptyModule`. When converting the `ChiselAnnotation` to firrtl's `Annotation`, we print out the resulting `Target`. As shown, despite `EmptyModule` actually only being elaborated once, we still provide different targets depending on how the instance or definition is selected.

```scala
import chisel3._
import chisel3.experimental.hierarchy.{Definition, Instance, Hierarchy, instantiable, public}
import firrtl.annotations.{IsModule, NoTargetAnnotation}
case object EmptyAnnotation extends NoTargetAnnotation
case class MyChiselAnnotation(m: Hierarchy[RawModule], tag: String) extends experimental.ChiselAnnotation {
  def toFirrtl = {
    println(tag + ": " + m.toTarget)
    EmptyAnnotation
  }
}

@instantiable
class EmptyModule extends Module {
  println("Elaborating EmptyModule!")
}

@instantiable
class TwoEmptyModules extends Module {
  val definition = Definition(new EmptyModule)
  val i0         = Instance(definition)
  val i1         = Instance(definition)
}

class Top extends Module {
  val definition = Definition(new TwoEmptyModules)
  val instance   = Instance(definition)
  aop.Select.allInstancesOf[EmptyModule](instance).foreach { i =>
    experimental.annotate(MyChiselAnnotation(i, "instance"))
  }
  aop.Select.allDefinitionsOf[EmptyModule](instance).foreach { d =>
    experimental.annotate(MyChiselAnnotation(d, "definition"))
  }
}
```
```
Elaborating EmptyModule!
instance: ~Top|Top/instance:TwoEmptyModules/i0:EmptyModule
instance: ~Top|Top/instance:TwoEmptyModules/i1:EmptyModule
definition: ~Top|EmptyModule
```

You can also use `Select.ios` on either a `Definition` or an `Instance` to annotate the I/Os appropriately:

```scala
case class MyIOAnnotation(m: Data, tag: String) extends experimental.ChiselAnnotation {
  def toFirrtl = {
    println(tag + ": " + m.toTarget)
    EmptyAnnotation
  }
}

@instantiable
class InOutModule extends Module {
  @public val in = IO(Input(Bool()))
  @public val out = IO(Output(Bool()))
  out := in
}

@instantiable
class TwoInOutModules extends Module {
  val in = IO(Input(Bool()))
  val out = IO(Output(Bool()))
  val definition = Definition(new InOutModule)
  val i0         = Instance(definition)
  val i1         = Instance(definition)
  i0.in := in
  i1.in := i0.out
  out := i1.out
}

class InOutTop extends Module {
  val definition = Definition(new TwoInOutModules)
  val instance   = Instance(definition)
  aop.Select.allInstancesOf[InOutModule](instance).foreach { i =>
    aop.Select.ios(i).foreach { io =>
      experimental.annotate(MyIOAnnotation(io, "instance io"))
  }}
  aop.Select.allDefinitionsOf[InOutModule](instance).foreach { d =>
    aop.Select.ios(d).foreach {io =>
      experimental.annotate(MyIOAnnotation(io, "definition io"))
  }}
}
```
```
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i0:InOutModule>clock
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i0:InOutModule>reset
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i0:InOutModule>in
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i0:InOutModule>out
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i1:InOutModule>clock
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i1:InOutModule>reset
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i1:InOutModule>in
instance io: ~InOutTop|InOutTop/instance:TwoInOutModules/i1:InOutModule>out
definition io: ~InOutTop|InOutModule>clock
definition io: ~InOutTop|InOutModule>reset
definition io: ~InOutTop|InOutModule>in
definition io: ~InOutTop|InOutModule>out
```