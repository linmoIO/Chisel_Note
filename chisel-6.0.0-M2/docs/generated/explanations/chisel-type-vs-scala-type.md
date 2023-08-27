---
layout: docs
title:  "Chisel Type vs Scala Type"
section: "chisel3"
---

# Chisel Type vs Scala Type

The Scala compiler cannot distinguish between Chisel's representation of hardware such as `false.B`, `Reg(Bool())`
and pure Chisel types (e.g. `Bool()`). You can get runtime errors passing a Chisel type when hardware is expected, and vice versa.

## Scala Type vs Chisel Type vs Hardware


The *Scala* type of the Data is recognized by the Scala compiler, such as `Bool`, `Decoupled[UInt]` or `MyBundle` in
```scala
class MyBundle(w: Int) extends Bundle {
  val foo = UInt(w.W)
  val bar = UInt(w.W)
}
```

The *Chisel* type of a `Data` is a Scala object. It captures all the fields actually present,
by names, and their types including widths.
For example, `MyBundle(3)` creates a Chisel Type with fields `foo: UInt(3.W),  bar: UInt(3.W))`.

Hardware is `Data` that is "bound" to synthesizable hardware. For example `false.B` or `Reg(Bool())`.
The binding is what determines the actual directionality of each field, it is not a property of the Chisel type.

A literal is a `Data` that is respresentable as a literal value without being wrapped in Wire, Reg, or IO.

## Chisel Type vs Hardware vs Literals

The below code demonstrates how objects with the same Scala type (`MyBundle`) can have different properties.

```scala
import chisel3.experimental.BundleLiterals._

class MyModule(gen: () => MyBundle) extends Module {
                                                            //   Hardware   Literal
    val xType:    MyBundle     = new MyBundle(3)            //      -          -
    val dirXType: MyBundle     = Input(new MyBundle(3))     //      -          -
    val xReg:     MyBundle     = Reg(new MyBundle(3))       //      x          -
    val xIO:      MyBundle     = IO(Input(new MyBundle(3))) //      x          -
    val xRegInit: MyBundle     = RegInit(xIO)               //      x          -
    val xLit:     MyBundle     = xType.Lit(                 //      x          x
      _.foo -> 0.U(3.W),
      _.bar -> 0.U(3.W)
    )
    val y:        MyBundle = gen()                          //      ?          ?

    // Need to initialize all hardware values
    xReg := DontCare
}
```


## Chisel Type vs Hardware -- Specific Functions and Errors

`.asTypeOf` works for both hardware and Chisel type:

```scala
elaborate(new Module {
  val chiselType = new MyBundle(3)
  val hardware = Wire(new MyBundle(3))
  hardware := DontCare
  val a = 0.U.asTypeOf(chiselType)
  val b = 0.U.asTypeOf(hardware)
})
```

Can only `:=` to hardware:
```scala
// Do this...
elaborate(new Module {
  val hardware = Wire(new MyBundle(3))
  hardware := DontCare
})
```
```scala
// Not this...
elaborate(new Module {
  val chiselType = new MyBundle(3)
  chiselType := DontCare
})
// chisel3.package$ExpectedHardwareException: data to be connected 'MyBundle' must be hardware, not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$21$$anonfun$apply$21$$anon$3.<init>(chisel-type-vs-scala-type.md:90)
// 	at repl.MdocSession$MdocApp$$anonfun$21$$anonfun$apply$21.apply(chisel-type-vs-scala-type.md:88)
// 	at repl.MdocSession$MdocApp$$anonfun$21$$anonfun$apply$21.apply(chisel-type-vs-scala-type.md:88)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Can only `:=` from hardware:
```scala
// Do this...
elaborate(new Module {
  val hardware = IO(new MyBundle(3))
  val moarHardware = Wire(new MyBundle(3))
  moarHardware := DontCare
  hardware := moarHardware
})
```
```scala
// Not this...
elaborate(new Module {
  val hardware = IO(new MyBundle(3))
  val chiselType = new MyBundle(3)
  hardware := chiselType
})
// chisel3.package$ExpectedHardwareException: data to be connected 'MyBundle' must be hardware, not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$29$$anonfun$apply$27$$anon$5.<init>(chisel-type-vs-scala-type.md:115)
// 	at repl.MdocSession$MdocApp$$anonfun$29$$anonfun$apply$27.apply(chisel-type-vs-scala-type.md:112)
// 	at repl.MdocSession$MdocApp$$anonfun$29$$anonfun$apply$27.apply(chisel-type-vs-scala-type.md:112)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Have to pass hardware to `chiselTypeOf`:
```scala
// Do this...
elaborate(new Module {
  val hardware = Wire(new MyBundle(3))
  hardware := DontCare
  val chiselType = chiselTypeOf(hardware)
})
```
```scala
// Not this...
elaborate(new Module {
  val chiselType = new MyBundle(3)
  val crash = chiselTypeOf(chiselType)
})
// chisel3.package$ExpectedHardwareException: 'MyBundle' must be hardware, not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34$$anon$7$$anonfun$39$$anonfun$apply$36.apply(chisel-type-vs-scala-type.md:138)
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34$$anon$7$$anonfun$39$$anonfun$apply$36.apply(chisel-type-vs-scala-type.md:138)
// 	at chisel3.experimental.prefix$.apply(prefix.scala:50)
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34$$anon$7$$anonfun$39.apply(chisel-type-vs-scala-type.md:138)
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34$$anon$7$$anonfun$39.apply(chisel-type-vs-scala-type.md)
// 	at chisel3.internal.plugin.package$.autoNameRecursively(package.scala:33)
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34$$anon$7.<init>(chisel-type-vs-scala-type.md:138)
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34.apply(chisel-type-vs-scala-type.md:136)
// 	at repl.MdocSession$MdocApp$$anonfun$37$$anonfun$apply$34.apply(chisel-type-vs-scala-type.md:136)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Have to pass hardware to `*Init`:
```scala
// Do this...
elaborate(new Module {
  val hardware = Wire(new MyBundle(3))
  hardware := DontCare
  val moarHardware = WireInit(hardware)
})
```
```scala
// Not this...
elaborate(new Module {
  val crash = WireInit(new MyBundle(3))
})
// chisel3.package$ExpectedHardwareException: wire initializer 'MyBundle' must be hardware, not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40$$anon$9$$anonfun$45$$anonfun$apply$41.apply(chisel-type-vs-scala-type.md:160)
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40$$anon$9$$anonfun$45$$anonfun$apply$41.apply(chisel-type-vs-scala-type.md:160)
// 	at chisel3.experimental.prefix$.apply(prefix.scala:50)
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40$$anon$9$$anonfun$45.apply(chisel-type-vs-scala-type.md:160)
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40$$anon$9$$anonfun$45.apply(chisel-type-vs-scala-type.md)
// 	at chisel3.internal.plugin.package$.autoNameRecursively(package.scala:33)
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40$$anon$9.<init>(chisel-type-vs-scala-type.md:160)
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40.apply(chisel-type-vs-scala-type.md:159)
// 	at repl.MdocSession$MdocApp$$anonfun$44$$anonfun$apply$40.apply(chisel-type-vs-scala-type.md:159)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Can't pass hardware to a `Wire`, `Reg`, `IO`:
```scala
// Do this...
elaborate(new Module {
  val hardware = Wire(new MyBundle(3))
  hardware := DontCare
})
```
```scala
// Not this...
elaborate(new Module {
  val hardware = Wire(new MyBundle(3))
  val crash = Wire(hardware)
})
// chisel3.package$ExpectedChiselTypeException: wire type '_44_Anon.hardware: Wire[MyBundle]' must be a Chisel type, not hardware
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44$$anon$11$$anonfun$51$$anonfun$apply$47.apply(chisel-type-vs-scala-type.md:182)
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44$$anon$11$$anonfun$51$$anonfun$apply$47.apply(chisel-type-vs-scala-type.md:182)
// 	at chisel3.experimental.prefix$.apply(prefix.scala:50)
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44$$anon$11$$anonfun$51.apply(chisel-type-vs-scala-type.md:182)
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44$$anon$11$$anonfun$51.apply(chisel-type-vs-scala-type.md)
// 	at chisel3.internal.plugin.package$.autoNameRecursively(package.scala:33)
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44$$anon$11.<init>(chisel-type-vs-scala-type.md:182)
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44.apply(chisel-type-vs-scala-type.md:180)
// 	at repl.MdocSession$MdocApp$$anonfun$49$$anonfun$apply$44.apply(chisel-type-vs-scala-type.md:180)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

`.Lit` can only be called on Chisel type:
```scala
// Do this...
elaborate(new Module {
  val hardwareLit = (new MyBundle(3)).Lit(
    _.foo -> 0.U,
    _.bar -> 0.U
  )
})
```
```scala
//Not this...
elaborate(new Module {
  val hardware = Wire(new MyBundle(3))
  val crash = hardware.Lit(
    _.foo -> 0.U,
    _.bar -> 0.U
  )
})
// chisel3.package$ExpectedChiselTypeException: bundle literal constructor model '_52_Anon.hardware: Wire[MyBundle]' must be a Chisel type, not hardware
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52$$anon$13$$anonfun$56$$anonfun$apply$55.apply(chisel-type-vs-scala-type.md:206)
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52$$anon$13$$anonfun$56$$anonfun$apply$55.apply(chisel-type-vs-scala-type.md:206)
// 	at chisel3.experimental.prefix$.apply(prefix.scala:50)
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52$$anon$13$$anonfun$56.apply(chisel-type-vs-scala-type.md:206)
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52$$anon$13$$anonfun$56.apply(chisel-type-vs-scala-type.md)
// 	at chisel3.internal.plugin.package$.autoNameRecursively(package.scala:33)
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52$$anon$13.<init>(chisel-type-vs-scala-type.md:206)
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52.apply(chisel-type-vs-scala-type.md:204)
// 	at repl.MdocSession$MdocApp$$anonfun$54$$anonfun$apply$52.apply(chisel-type-vs-scala-type.md:204)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Can only use a Chisel type within a `Bundle` definition:
```scala
// Do this...
elaborate(new Module {
  val hardware = Wire(new Bundle {
    val nested = new MyBundle(3)
  })
  hardware := DontCare
})
```
```scala
// Not this...
elaborate(new Module {
  val crash = Wire(new Bundle {
    val nested = Wire(new MyBundle(3))
  })
})
// chisel3.package$ExpectedChiselTypeException: Bundle: AnonymousBundle contains hardware fields: nested: _60_Anon.crash_nested: Wire[MyBundle]
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60$$anon$16$$anonfun$62$$anonfun$apply$61.apply(chisel-type-vs-scala-type.md:232)
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60$$anon$16$$anonfun$62$$anonfun$apply$61.apply(chisel-type-vs-scala-type.md:232)
// 	at chisel3.experimental.prefix$.apply(prefix.scala:50)
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60$$anon$16$$anonfun$62.apply(chisel-type-vs-scala-type.md:232)
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60$$anon$16$$anonfun$62.apply(chisel-type-vs-scala-type.md)
// 	at chisel3.internal.plugin.package$.autoNameRecursively(package.scala:33)
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60$$anon$16.<init>(chisel-type-vs-scala-type.md:232)
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60.apply(chisel-type-vs-scala-type.md:231)
// 	at repl.MdocSession$MdocApp$$anonfun$61$$anonfun$apply$60.apply(chisel-type-vs-scala-type.md:231)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Can only call `directionOf` on Hardware:
```scala
import chisel3.reflect.DataMirror

class Child extends Module{
  val hardware = IO(new MyBundle(3))
  hardware := DontCare
  val chiselType = new MyBundle(3)
}
```
```scala
// Do this...
elaborate(new Module {
  val child = Module(new Child())
  child.hardware := DontCare
  val direction = DataMirror.directionOf(child.hardware)
})
```
```scala
// Not this...
elaborate(new Module {
val child = Module(new Child())
  child.hardware := DontCare
  val direction = DataMirror.directionOf(child.chiselType)
})
// chisel3.package$ExpectedHardwareException: node requested directionality on 'MyBundle' must be hardware, not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?
// 	at ... ()
// 	at repl.MdocSession$MdocApp$$anonfun$70$$anonfun$apply$68$$anon$19.<init>(chisel-type-vs-scala-type.md:271)
// 	at repl.MdocSession$MdocApp$$anonfun$70$$anonfun$apply$68.apply(chisel-type-vs-scala-type.md:268)
// 	at repl.MdocSession$MdocApp$$anonfun$70$$anonfun$apply$68.apply(chisel-type-vs-scala-type.md:268)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

Can call `specifiedDirectionOf` on hardware or Chisel type:
```scala
elaborate(new Module {
  val child = Module(new Child())
  child.hardware := DontCare
  val direction0 = DataMirror.specifiedDirectionOf(child.hardware)
  val direction1 = DataMirror.specifiedDirectionOf(child.chiselType)
})
```

## `.asInstanceOf` vs `.asTypeOf` vs `chiselTypeOf`

`.asInstanceOf` is a Scala runtime cast, usually used for telling the compiler
that you have more information than it can infer to convert Scala types:

```scala
class ScalaCastingModule(gen: () => Bundle) extends Module {
  val io = IO(Output(gen().asInstanceOf[MyBundle]))
  io.foo := 0.U
}
```

This works if we do indeed have more information than the compiler:
```scala
elaborate(new ScalaCastingModule( () => new MyBundle(3)))
```

But if we are wrong, we can get a Scala runtime exception:
```scala
class NotMyBundle extends Bundle {val baz = Bool()}
elaborate(new ScalaCastingModule(() => new NotMyBundle()))
// java.lang.ClassCastException: class repl.MdocSession$MdocApp$$anonfun$79$NotMyBundle$1 cannot be cast to class repl.MdocSession$MdocApp$MyBundle (repl.MdocSession$MdocApp$$anonfun$79$NotMyBundle$1 and repl.MdocSession$MdocApp$MyBundle are in unnamed module of loader scala.reflect.internal.util.AbstractFileClassLoader @51674951)
// 	at ... ()
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76$$anonfun$apply$71$$anonfun$apply$72$$anonfun$apply$73.apply(chisel-type-vs-scala-type.md:293)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76$$anonfun$apply$71$$anonfun$apply$72$$anonfun$apply$73.apply(chisel-type-vs-scala-type.md:293)
// 	at chisel3.SpecifiedDirection$.specifiedDirection(Data.scala:63)
// 	at chisel3.Output$.apply(Data.scala:266)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76$$anonfun$apply$71$$anonfun$apply$72.apply(chisel-type-vs-scala-type.md:293)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76$$anonfun$apply$71$$anonfun$apply$72.apply(chisel-type-vs-scala-type.md:293)
// 	at chisel3.IO$.apply(IO.scala:27)
// 	at chisel3.experimental.BaseModule.IO(Module.scala:620)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76$$anonfun$apply$71.apply(chisel-type-vs-scala-type.md:293)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76$$anonfun$apply$71.apply(chisel-type-vs-scala-type.md:293)
// 	at chisel3.experimental.prefix$.apply(prefix.scala:50)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76.apply(chisel-type-vs-scala-type.md:293)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule$$anonfun$76.apply(chisel-type-vs-scala-type.md)
// 	at chisel3.internal.plugin.package$.autoNameRecursively(package.scala:33)
// 	at repl.MdocSession$MdocApp$ScalaCastingModule.<init>(chisel-type-vs-scala-type.md:293)
// 	at repl.MdocSession$MdocApp$$anonfun$79$$anonfun$apply$75.apply(chisel-type-vs-scala-type.md:309)
// 	at repl.MdocSession$MdocApp$$anonfun$79$$anonfun$apply$75.apply(chisel-type-vs-scala-type.md:309)
// 	at ... ()
// 	at ... (Stack trace trimmed to user code only. Rerun with --full-stacktrace to see the full stack trace)
```

`.asTypeOf` is a conversion from one `Data` subclass to another.
It is commonly used to assign data to all-zeros, as described in [this cookbook recipe](https://www.chisel-lang.org/chisel3/docs/cookbooks/cookbook.html#how-can-i-tieoff-a-bundlevec-to-0), but it can
also be used (though not really recommended, as there is no checking on
width matches) to convert one Chisel type to another:

```scala
class SimilarToMyBundle(w: Int) extends Bundle{
  val foobar = UInt((2*w).W)
}

ChiselStage.emitSystemVerilog(new Module {
  val in = IO(Input(new MyBundle(3)))
  val out = IO(Output(new SimilarToMyBundle(3)))

  out := in.asTypeOf(out)
})
// res12: String = """// Generated by CIRCT firtool-1.43.0
// module _82_Anon(	// <stdin>:3:10
//   input        clock,	// <stdin>:4:11
//                reset,	// <stdin>:5:11
//   input  [2:0] in_foo,	// chisel-type-vs-scala-type.md:324:14
//                in_bar,	// chisel-type-vs-scala-type.md:324:14
//   output [5:0] out_foobar	// chisel-type-vs-scala-type.md:325:15
// );
// 
//   assign out_foobar = {in_foo, in_bar};	// <stdin>:3:10, chisel-type-vs-scala-type.md:327:21
// endmodule
// 
// """
```

In contrast to `asInstanceOf` and `asTypeOf`,
`chiselTypeOf` is not a casting operation. It returns a Scala object which
can be used as shown in the examples above to create more Chisel types and
hardware with the same Chisel type as existing hardware.
