# Improve Syntax for Function Calls with Receivers

* Proposal: [SE-NNNN](NNNN-receiversFunctions.md)
* Authors: [Philip Wedemann](https://github.com/hfhbd)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Each function in Swift has a receiver, a list of parameters, and a return value. For class functions, this means the
receiver is the class. Because functions are first-class types, it is possible to assign a function to a variable. To
call a variable, which holds a reference to a class function, the call needs to pass the receiver, an instance of the
class.

This call syntax to hand over the receiver is ugly and hard to understand because the syntax is based on two pairs of
parentheses, one for the receiver, one for the parameters. Additionally, the generated type is not clear too.
Furthermore, using a class function as a parameter with closures is also hard to understand, because the receiver is not
switched. This proposal improves this syntax and made it clear to the developer with syntactic sugar, but keeps the
existing type system under the hood.

Swift-evolution
thread: [[Pitch] Functional - Clearer Design for Functions with Receivers](https://forums.swift.org/t/pitch-functional-clearer-design-for-functions-with-receivers)

## Motivation

Thankfully in Swift, it is possible to assign a function to a variable, see `simple` in example 1.1.

````swift
func method() { }

let simple = method // simple is () -> ()
simple()

struct Foo {
    func printing(_ i: Int) {
        print("\(Self.Type.self):\(i)")
    }
}
 
let sample = Foo.printing // sample is (Foo) -> (Int) -> Void
sample(Foo())(0) // calling sample with receiver of type Foo and 0 as parameter

// example 1.1, compiles
````

To call the referenced function the variable will be invoked as a function, with optional parameters. In the example,
the variable `simple` holds a reference to the function `method` and to call the reference function, `simple` needs to
invoked by using `simple()`.

It is also possible to assign a class function to a variable, like `sample` in the example. Here, the variable `sample`
holds a reference to the class function `printing` from the type `Foo`. But to invoke the variable, the invocation needs
also the receiver. This receiver will be passed over as a parameter in the first `()`, following by the optional
parameters in the next `()`.

To support passing the receiver, the type of the variable `sample: (Foo) -> (Int) -> Void` is a function, which needs
the receiver as a parameter and returns the "real" class function, which will be called by a second invocation, hence
the two pair of parentheses, `sample(Foo())(0)`.

Furthermore, it is also possible to use a class function as a parameter in another function. This results in this
example, see the following code extending the first sample.

````swift
struct Bar {
    func printing(_ i: Int) {
        print("\(Self.Type.self):\(i)")
    }
    
    func call(action: (Foo) -> (Int) -> Void) {
        let foo = Foo()
        action(foo)(0)
    }
}
Bar().call(action: sample)    // prints: "Foo.Type:0"

// example 2.1, compiles
````

The function `call` has the `action` parameter, which holds a reference to a class function of `Foo`. To use the current
design with the variable `sample`, the variable needs to be passed as a parameter `Bar().call(action: sample)`.

Instead of using the variable `sample`, it is also possible to use a closure, which will handle the `action` and will be
called in `call`, see the following code.

````swift
extension Bar {
    func closureCurrentDesign() {
        call(action: { (foo: Foo) -> (Int) -> Void in
            return { (i: Int) in // this second closure is needed because it is returning (Int) -> Void 
                
                self.printing(i) // although the receiver of `action` is Foo, self is Bar
                foo.printing(i) // thats why this correct call with explicit receiver is necessary
            }
        })
    }
}
Bar().closureCurrentDesign() // prints: "Bar.Type:0\nFoo.Type:0"

// example 3.1, compiles
````

Because the type of the `action` parameter is `(Foo) -> (Int) -> Void`, the closure must return an inner closure. The
outer closure has a parameter of type `Foo` and returns the inner closure of type `(Int) -> Void`. In the end, the inner
closure is the closure containing the actual programming and must be returned. Using the two closures is syntactically
correct, but it is not a clear syntax and easily hard to understand/implement.

Additionally, using `self` inside the closure will always refer to the outer `Self`, here `Bar` and will not change to
the inner receiver, which would be `Foo` inside the closure. To switch the receiver, a manual call is necessary, by
using the variable `foo`.

## Proposed solution

### Clearer type syntax

First of all, instead of assign the type of variable `sample` to `(Foo) -> (Int) -> Void`, a clearer syntax is proposed.
The new syntax contains the receiver, delimited by a `.` followed by the optional parameters, and ends with the return
type. For `sample` the new syntax would be `Foo.(Int) -> Void`. This syntax directly shows the developer the receiver,
and the needed parameters with the return type in an already known description.

````swift
struct Foo {
    func printing(_ i: Int) {
        print("\(Self.Type.self):\(i)")
    }
}
let sample: Foo.(Int) -> Void = Foo.printing

// example 2.1, wont compile
````

### Clearer invoking syntax

Second, the syntax to invoke the variable would also change to a clearer design. Calling `sample` is now possible by
using the variable directly on the receiver, like a "normal" class function, with only one pair of
parentheses, `Foo().sample(0)`.

With this two changes, the second code example would be changed to the following:

````swift
struct Bar {
    func printing(_ i: Int) {
        print("\(Self.Type.self):\(i)")
    }
    
    func call(action: Foo.(Int) -> Void) {
        let foo = Foo()
        foo.action(0)
    }
}
Bar().call(action: sample)    // prints: "Foo.Type:0"

// example 2.2, wont compile
````

### Clearer closure syntax

Third, the syntax for a closure based invocation would be clearer, only one closure would be needed, not a closure
returning a closure.

````swift
extension Bar {
    func closureCurrentDesign() {
        call(action: { Foo.(i: Int) -> Void in 
                self.printing(i) // but self is Bar
            }
        })
    }
}
Bar().closureCurrentDesign() // prints: "Bar.Type:0"

// example 2.3, wont compiles
````

### Receiver switching

Fourth, the `self` inside the closure would be automatically switched to the receiver of the inner closure.

````swift
extension Bar {
    func closureCurrentDesign() {
        call(action: { Foo.(i: Int) -> Void in 
                self.printing(i) // self is implicit switched to Foo, the receiver of this closure
                self:Bar.printing(i) // switch explicite to the outer receiver Bar
            }
        })
    }
}
Bar().closureCurrentDesign() // prints: "Foo.Type:0\nBar.Type:0"

// example 2.4, wont compiles
````

## Detailed design

### Type Syntax

The new syntax contains the receiver, delimited by a `.` followed by the optional parameters and ends with the return
type, `Receiver.(Parameters) -> Result`. As the old syntax, `(Receiver) -> (Parameters) -> Result`, all current
possibilities are still allowed, it is only a more readable syntactic sugar. It simply removes the `() ->` and replaces
it with a `.`. There are no new limits/restrictions beyond the current ones. Returning a receiver function is still
allowed, also using a receiver function as a parameter.

````swift
struct New {
    func a(action: New.() -> Void) -> Void {
        self.action()
        action() // shortcut, because self is New, the current receiver
    }
    
    func b(action: New.(Int) -> Void) -> Void {
        self.action(42)
    }
    
    func c(action: New.() -> New) -> Void {
        let a: New = self.action()
    }
    
    func d(action: New.() -> New) -> New {
        return self.action()
    }
    
    func e(action: New.() -> (New.() -> Void)) -> Void { // returning a function is still allowed
        let resultAction: New.() -> Void = self.action()
        self.resultAction()
        
        action()() // possible, but not swifty
    }
    
    func f(action: New.(Int) -> (New.(Int) -> Int)) -> Void { // returning a function is still allowed
        let resultAction: New.(Int) -> Void = self.action(42)
        let i = self.resultAction()
        
        let ii = action(42)(42) // possible shortcut, but not swifty, see Old for old possible syntax
    }
    
    var s: New.(Int) -> Void { // as always, a computed variable can return a receiver function too
    { New.(i: Int) -> Void in
           self.t(i) // call the function stored in the variable t  
        }
    }
    var t: New.(Int) -> Void // as always, a stored variable can have a receiver function as a Type too
}

struct Old {
    func a(action: (Old) -> () -> Void) -> Void {
        action(self)()
    }
    
    func b(action: (Old) -> (Int) -> Void) -> Void {
        action(self)(42)
    }
    
    func c(action: (Old) -> () -> Old) -> Void {
        let a: Old = action(self)()
    }
    
    func d(action: (Old) -> () -> Old) -> Old {
        return action(self)()
    }
    
    func e(action: (Old) -> () -> (Old) -> () -> Void) -> Void {
        let resultAction: (Old) -> () -> Void = action(self)()
        resultAction(self)()
        
        action(self)()(self)() // possible shortcut, but not swifty
    }
    
    func f(action: (Old) -> (Int) -> (Old) -> (Int) -> Int) -> Void {
        let resultAction = action(self)(42)
        let i = resultAction(self)(42)
        
        let ii = action(self)(42)(self)(42) // possible shortcut, but not swifty at all
    }
    
    var s: (Old) -> (Int) -> Void { // as always, a computed variable can return a receiver function too
        { (old: Old) -> (Int) -> Void in
        return { (i: Int) -> Void in
                self.t(self)(i) // call the function stored in the variable t 
            }
        }
    }
    var t: (Old) -> (Int) -> Void // as always, a stored variable can have a receiver function as a Type too
}
````

### Invocation Syntax

The new syntax allows to invoke a variable with a receiver as an argument directly on the receiver like a normal class
function, `let r: Result = receiverInstance.variable(Parameters)`. As the new type syntax, this is also new syntactic
sugar only for the old invocation syntax `let r: Result = variable(receiverInstance)(Parameters)`, no new
limits/restrictions are applied, except overloading, see the following example. When the name of the parameter has the
same name of an existing class function, directly or indirectly using an extension, there will be this order:

1. The directly defined class function with the same name, will always win and will be called.
2. If there is no directly defined class function with the same name, the parameter function will be called.
3. Extension functions will never be called if they have the same name. Allowing this could break code and results into
   unwanted behavior and non-deterministic code, other developers could change the internal call hierarchy with their
   own extensions.

To call the same-named extension function, the internal parameter name has to be changed.

````swift
struct Foo {
    func bar() { }
    
    func conflic(bar: Foo.() -> Void) {
        self.bar() // calling the class function
        bar() // calling the class function
        
        let baz = bar // to call the paramter, assign it to a another named variable
        baz()
        self.baz() 
    } 
}

struct Baz {
    func conflic(foo: Baz.() -> Void) {
        self.foo() // calling the paramter function
        foo() // calling the parameter function
    }
    
    func noConflic(foo bar: Baz.() -> Void) {
        self.foo() // calling the extension function foo
        bar() // calling the parameter function
    } 
}

extension Baz {
   func foo() { // will never be called from `conflict`. the parameter name has to be changed
   
   }
}
````

### Closure Syntax

With the old syntax, defining the action inside a closure is very counterintuitive and hard to read. The main action is
needed to write in a second closure, which is called from the first one as a return value. With the new type syntax, the
type of the parameter is changed. The second closure is now directly inlined with the first closure,
no `return { (Parameter) in -> Result`
is required anymore. Accessing a parameter, as before and already done in Swift, is provided by the
name, `Receiver.(name: Parameter) -> Result in`.

````swift
extension Bar {
    func closureCurrentDesign() {
        call(action: { (foo: Foo) -> (Int) -> Void in
            return { (i: Int) in // this second closure is needed because it is returning (Int) -> Void 
                // do something
            }
        })
    }
    
    func closureNewDesign() {
        call(action: { Foo.(i: Int) -> Void in
           // do something
        })
    }
}

// example 3.1 compare closures
````

### Receiver switching

With the new type and closure syntax, there is no new explicitly named parameter to access the receiver of the closure.
In example 3.1, there is no `foo` variable anymore. The idea of receiver functions is to refer to the receiver
using `self`. This behavior is well known in class functions. To archive the same expected behavior in closures, the
receiver has to switch to the receiver of the inner closure. This receiver is already known, for the developer using the
clear new syntax, as well as the compiler/lexer.

To switch to an outer receiver, an explicit label on `self` is required. The already label syntax from Swift
uses `labelName:`.

````swift
lp: while(true) {
   if(something) {
   break lp
}
````

This syntax would be also used to switch `self`, by appending the label before `labelName:self`. Although the current
common label syntax would result in `Bar:self`, it looks very similar to `Bar.self`. Furthermore the `.` and the `:`
lies on the same key, everywhere else the wrong char results in a lexer error, e.g. `let a . String`, but this mistake
would not result in a lexer error, `Bar.self` and `Bar:self` would be both lexically correct. To prevent mangling, the
syntax `self:Bar`, `self:labelName` is proposed.

````swift
extension Bar {
    func closureNewDesign() {
        call(action: { Foo.(i: Int) -> Void in
           self.printing(i) // self is foo
           self:Bar.printing(i) // self is now Bar
        })
    }
}
````

The upper proposed changes are syntactic sugar only, without changing any code/behavior. Changing the receiver of the
closure would be a semantic change. Although the old syntax is quite uncommon in Swift, there should be no semantic code
changes breaking existing code. Instead, there must be some kind of opt-in.

1. explicit writing the receiver:
   If the developer uses explicitly the new type syntax, the receiver should change automatically to the inner one.
1. Use annotation to opt-in:
   If the developer annotates the usage with the annotation `@switchSelf`, the receiver should change automatically to
   the inner one. This annotation should be allowed:
    - directly on the closure, only this closure uses the automatic receiver switching
    - directly on the function, all closures inside this function uses the automatic receiver switching
    - on the class/extension, all closures inside this class/extension uses the automatic receiver switching
1. As a compiler flag to enable this feature globally.

````swift
@switchSelf // alternative: annotate the whole extension/class to use this optin in every receiver function
extension Bar {
    func explicite() {
        call(action: { Foo.(i: Int) -> Void in // Foo.(i: Int) is explicit written by the developer
           self.printing(i) // self is Foo
        })
    }
    
    func annotateClosure() {
        @switchSelf
        call(action: { i in // Foo.(i: Int) -> Void is the implicite type
            self.printing(i) // self is Foo
        })
    }
    
    @switchSelf
    func annotateFunction() {
        call(action: { i in // Foo.(i: Int) -> Void is the implicite type
           // do something
        })
    }
}

// example 3.2 opt in
````

### Mutating support

Currently, the following code results in an error `Partial application of 'mutating' method is not allowed`:

````swift
struct F {
    nonmutating func a() { }
    mutating func b() { }
}

let s: (F) -> () -> () = F.a
let t: (inout F) -> () -> () = F.b // Partial application of 'mutating' method is not allowed
var f = F()
t(&f)()
````

The new syntax will not support `mutating` functions as well.

## Outlook

With this design, it would be possible to define the following functions:

````swift
func <T, R> with(receiver: T, action: T.() -> R) -> R {
    return receiver.action()
}

func <R> run(action: () -> R) -> R {
    return action()
}

extension T {
    mutating func apply(action: T.() -> Void) -> T {
        action() // shortcut for self.action()
        return self
    }
    func <R> run(action: T.() -> R) -> R {
        return self.action() 
    }
    func also(action: (T) -> Void) -> T {
        action(self)
        return self
    }
    func <R> let(action: (T) -> R) -> R {  // could be renamed to prevent name conflict
        return action(self)
    }
}

let s: Void = with(Foo()) {
    printing(0) // shortcut for: return self.printing(0)
}

let foo = Foo().apply {
    i = 42 // shortcut for: self.i = 42
}
foo.also {
   print($0) 
}
let foo.let { f in

}

// outlook
````

These functions are defined in the Kotlin Standard Library and are
called [scope functions](https://kotlinlang.org/docs/reference/scope-functions.html).

With this syntax and the scope functions,
[this pitch](https://forums.swift.org/t/draft-proposal-multi-property-assignment-operator/1002) could be easily
implemented too. Instead of implementing a new multi-assignment operator, in the language and the compiler, it would be
possible by simply using the `apply` function, without needing a compiler change.

````swift
var object = MyObject()
  object .= (
    property1: 1,
    property2: "a"
  )
  
var object = MyObject()
object.apply {
    property1 = 1
    property2 = "a"
}
````

## Source compatibility

Under the hood, the compiler should use the old syntax. The new syntax is syntactic sugar for the developer only. This
allows source compatibility with older Swift versions. The old syntax could be still used.

The receiver switching using the new closure syntax would be a semantic change, so this switching should be enabled
using the new closure syntax only! The compiler still uses the old syntax and switched the receiver back to the
old `self` to keep source compatibility.

````swift
extension Bar {
    func closureCurrentDesign() {
        call(action: { Foo.(i: Int) -> Void in 
                self.printing(i) // self is implicit switched to Foo, only with this closure syntax, opt in!
                self:Bar.printing(i) // switch explicite to the outer receiver Bar
            }
        })
    }
    
    func closureCurrentDesignShort() { // removed all optional parts
        call { i in 
            printing(i) // self is implicit switched to Foo, only with this closure syntax, opt in!
        }
    }
}
Bar().closureCurrentDesign() // prints: "Foo.Type:0\nBar.Type:0"

// converts under the hood to this old syntax code

extension Bar {
    func closureCurrentDesign() {
        call(action: { (foo: Foo) -> (Int) -> Void in
            return { (i: Int) in
                
                foo.printing(i) // switching the receiver using the variable `foo`, which is self in the new syntax
                self.printing(i) // self is Bar
            }
        })
    }
}
Bar().closureCurrentDesign() // prints: "Foo.Type:0\nBar.Type:0"
````

## Effect on ABI stability

This syntax change is syntactic sugar only and does not affect the ABI.

## Effect on API resilience

This syntax change is syntactic sugar only and does not affect the API.

## Alternatives considered

### Type Syntax

````swift
struct Foo {
    func printing(_ i: Int) {
        print("\(Self.Type.self):\(i)")
    }
    func something() { }
}
let sample: (Foo) -> (Int) -> Void = Foo.printing // current design
let sample: (Foo).(Int) -> Void = Foo.printing // proposal is designed to remove these parantheses
let sample: Foo:(Int) -> Void = Foo.printing // infers with the type delimiter
let sample: Foo$(Int) -> Void = Foo.printing // unique syntax from the property wrapper, but could be misleading to property wrappers
let sample: (Foo, Int) -> Void = Foo.printing // SE-0042, manually check the parentheses, which is the receiver 
let sample: Foo.(Int) -> Void = Foo.printing // new syntax, simular to invocation syntax


let sample: (Foo) -> Void = Foo.something // SE-0042, unclear, something does not take any parameters, but `Foo` is in the parantheses 
let sample: Foo.() -> Void = Foo.something // new syntax, simular to invocation syntax

// example 2.1, wont compile
````

Only the new syntax uses the already common invocation syntax on a class function from Swift, using a new syntax with
different symbols is unclear. SE-0042 reduces the variables, but the syntax is still unclear when the function takes no
additional parameters, `let sample: (Foo) -> Void`. The proposed syntax is clearer, `let sample: Foo.() -> Void`.

### Invocation Syntax

None, the `.` delimiter is already used for class functions/attributes in Swift, reusing it for the invocation syntax to
archive exactly this familiar syntax is the only way.

### Receiver switching

Beside disabling this new option completely, there could be another syntax:

````swift
struct Bar {
   func something(action: Int.(Int) -> Int) {
      action { i in
         self$Bar // unique syntax from the property wrapper, but could be misleading to property wrappers
         self@Bar // syntax in Kotlin, but unfamilar in Swift
         Bar.self // already other meaning => conflict  
         Bar:self // possible syntax from the Swift loop: label annoation syntax
         self:Bar // proposed syntax, to prevent mangling . and :, which have completly different meaning
         
         return i
      }
   }
}
````

Although the current common label syntax would result in `Bar:self`, it looks very similar to `Bar.self`. Furthermore,
the `.` and the `:` lies on the same key, everywhere else the wrong char results in a lexer error, e.g. `let a . String`
, this mistake would not result in a lexer error, `Bar.self` and `Bar:self` would be both lexically correct. To prevent
mangling, the syntax `self:Bar` is proposed.

### OptIn annotation

`````swift
@reeiverSwitching // clear, describes exactly what the compiler does, but long alternative
@switchSelf // shorter, proposed
`````

## Acknowledgments

This design is highly motivated by
the [Kotlin syntax for receivers](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver).
The accepted
proposal [SE-0042](https://github.com/apple/swift-evolution/blob/master/proposals/0042-flatten-method-types.md)
introduced a similar design, but was never implemented, due to the `mutating` breaking change. 
