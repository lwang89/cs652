# Smalltalk (*subset*) Bytecode Compiler

## Goal

This goal of this project is to build a full compiler for a subset of Smalltalk. For example, you will read in a small program in, say, `test.st` such as:

```
Transcript show: 'hello'.
```

and compile it to symbol table information and bytecode using the test rig included with your starter kit. That "main" program compiles down to an implied `main` method in a `MainClass` (which is of type `STClass`). The `STClass.toTestString()` method provides the following information:

```
name: MainClass
superClass: 
fields: 
literals: 'Transcript','hello','show:'
methods:
    name: main
    qualifiedName: MainClass>>main
    nargs: 0
    nlocals: 0
    0000:  push_global    'Transcript'
    0003:  push_literal   'hello'
    0006:  send           1, 'show:'
    0011:  pop              
    0012:  self             
    0013:  return   
```

You can also run the compiler from the commandline to generate JSON-based bytecode files (See the `stc` alias at the bottom of this description):

```bash
$ stc -dis test.st # gen MainClass.sto for implied main + test-teststring.txt
```

The JSON output `.sto` files look like:

```json
{
  "name": "MainClass",
  "superClassName": "Object",
  "literals": [
    "Transcript",
    "hello",
    "show:"
  ],
  "fields": [],
  "methods": [
    {
      "name": "main",
      "isClassMethod": false,
      "qualifiedName": "MainClass>>main",
      "nargs": 0,
      "nlocals": 0,
      "bytecode": [ 16, 0, 0, 15, 0, 1, 25, 0, 1, 0, 2, 20, 2, 29 ],
      "blocks": []
    }
  ]
}
```

A virtual machine that knows how to execute these .sto files is available as a jar in this project. Maven pulls the VM in automatically as part of the build and the VM is used for testing Smalltalk program execution. See package `smalltalk.compiler.test.exec`.

I have provided a suitable grammar and symbol table construction code base. Your job is to focus on the code generation aspects of the compiler.

## Smalltalk language definition

Here is the [Smalltalk ANTLR grammar](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/grammars/smalltalk/compiler/Smalltalk.g4).

Our version of Smalltalk differs from the standard in a few ways:

* no class variables but allows class methods
* subclasses can see the fields of their parent classes; in ST-80 these are private
* we disallow definition of globals variables. x:=expr will generate code for expr but not the store if x is not a valid argument, local variable, or field. There is a `push_global` instruction but it is for looking up class names and other globally-visible symbols (only one: `Transcript` object).
* We allow forward refs (refs to classes not yet defined)
* no `#(1 2 3)` array literal notation, but with dynamic array notation `{1. 2. 3}`.
* no ';' extended msg send notation
* Much of the implementation is not exposed to the programmer, such as method invocation contexts.

A useful project with implementation information is [SOM Smalltalk](http://som-st.github.io/).

Paraphrasing the [Pharo cheat sheet](http://files.pharo.org/media/flyer-cheat-sheet.pdf):

### Literals and keywords
| Syntax | Semantics |
|--------:|--------|
|  `nil` | undefined object |
|`true,false`|boolean liberals|
|`self`|receiver of current message (method call)|
|`super`|receiver of current message but with superclass scope|
|`class`|start of a class definition or class method|
|`"..."`|comment (allowed anywhere)|
|`'abc'`|string literal|
|`$a`|character literal `a`|
|`123`|integer literal|
|`1.23`|floating-point literal (single precision), no scientific notation|

### Expression syntax

| Syntax | Semantics |
|---------:|--------|
|`.`|expression separator (not terminator)|
|`x :=` *expr*|assignment to local or field (there are no global variables)|
|`^`*expr*|return expression from method, even when nested in a `[...]`block|
|&#124;`x y`&#124;|define two local variables or fields|
|`{a. 1+2. aList size}`|dynamic array constructed from three expressions separated by periods|
|`[:x `&#124;` 2*x]`|code block taking one parameter and evaluating to twice that parameter; in common usage, these are called lambdas or closures.|
|`[:x :y `&#124;` x*y]`|code block taking two parameters|
|`[99] value` |use `value` method to evaluate a block with 99|
|`[:x 	`&#124;` 2*x] value: 10` |use `value:` method to evaluate a block with parameter 10|

### Message send expressions

Smalltalk uses the concept of message sending, which is really the same thing as method calling in Java. There are three kinds of messages:

1. **Unary messages**.  These are messages with no arguments. For example, Smalltalk `myList size` is just `myList.size()` in Java. The most common unary message is probably `new`, which creates a new object. For example, `Array new`, which is more or less `new Array()` in Java.
2. **Binary operator messages**.  These messages take one argument and take the form of one or more special symbols. For example, `1+2` is the usual addition and `x->y` creates an association object used by `Dictionary`.
	```java
	/** "A binary message selector is composed of one or
	     two nonalphanumeric characters. The only restriction is
	     that the second character cannot be a minus sign."
	     BlueBlook p49 in pdf.
	 */
	bop : (opchar|'-') opchar? ;
	opchar
		:	'+'
		|	'/' | '\\'
		|	'*' | '~'
		|	'<' | '>'
		|	'=' | '@' | '%' | '|' | '&' | '?' | ','
		;
	```
	
3. **Keyword messages**. These messages take one or more arguments, but unlike Java, the argument names are used in the call. For example, here's how to execute a code block over the values from 1 to 5: `1 to: 5 do: [:i | ...]`.  Method `to:do:` passes the iteration number as an argument to the code block when it evaluates it. Here's how to take the conjunction of two booleans: `true and: false`. Here's how to map a code block across a list to get another list: 
	```   |data|
   data := { 1. 2. 3. }.   data := data collect: [ :x | 2*x ]
	```

**Precedence**.  Unary operators have higher precedence than binary operators, which have higher precedence than keyword operators.  Operators within the same type (unary, binary, keyword) group left to right. That means that `1+2*3` is `(1+2)*3` not `1+(2*3)` as we use in arithmetic and most other programming languages. Parentheses override the default precedence as usual. Here's an example that uses all three operators:

```
1+2 to: myList size do: [:i | ...]
```

The keyword message `to:do:` has lowest precedence and so `myList size` is evaluated and passed as the `to:` parameter.  Similarly, `1+2` evaluates to 3 and is the receiver of the `to:do:` message. When it's unclear to the reader what the precedence is, use parentheses.

### Class, method syntax

Smalltalk has no file format or syntax for class definitions because it was all done in a code browser. In our case, we need a file format and some very simple syntax will suffice. The following class definition for `Array` demonstrates all bits of the syntax except fields:

```
class Array : Collection [
   "An object that represents a Smalltalk array of objects backed by Java class STArray"
   class new [ ^self new: 10 ]
   class new: size <primitive:#Array_Class_NEW>

   size <primitive:#Array_SIZE>
   at: i <primitive:#Array_AT>
   at: i put: v <primitive:#Array_AT_PUT>
   do: blk [
       1 to: self size do: [:i | blk value: (self at: i)].
   ]
]
```

Classes are defined using the `class` keyword followed by the name of the class. If there is a superclass, use a colon followed by the superclass name (`Collection`, in this case). The body of the class is wrapped in square brackets.

Class methods are proceeded with the `class` keyword but are otherwise the same as other methods. Both of the `new` methods are class methods and the second one, `new:` is a primitive that has no Smalltalk implementation. Instead, the VM will look for a `Primitive` called `Array_Class_NEW` and execute the associated `perform` method. The non-primitive class method `new` invokes the primitive `new:` with a parameter of 10.  `STCompiledBlock` differentiates between class and instance methods using field:

```java
/** True if method was defined as a class method in Smalltalk code */
public final boolean isClassMethod;
```

Method `size` takes no parameters and is primitive. Method `at:` takes one parameter and is primitive.  Method `at:put:` takes two parameters and is primitive.  Method `do:` takes one parameter, a code block, and has a Smalltalk implementation.

Field syntax look like:

```
class LinkedList : Collection [
   | head tail |
   ...
]
```

You should spend some time reading [STCompiledBlock](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/src/smalltalk/compiler/symbols/STCompiledBlock.java).

## Smalltalk Bytecode

In [Bytecode.java](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/src/smalltalk/compiler/Bytecode.java), you will see the definitions of the various bytecodes. Each instruction above gets its own unique integer "op code". There is also a definition of how many operands and the operand sizes so that we can disassemble code. For example, here is a class with a simple method:

```
class T [
	|x|
	foo [|y| x := y.]
]
```

and the bytecode generated for method `foo`:

```
0000:  push_local     0, 0
0005:  store_field    0
0008:  pop
0009:  self             
0010:  return           
```

The numbers on the left are the byte addresses of the instructions. The first instruction takes five bytes because there is one byte for the [`push_local`](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/src/smalltalk/compiler/Bytecode.java#L64) instruction and [two operands](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/src/smalltalk/compiler/Bytecode.java#L99) that are each two bytes long.

### Execution state

One of the critical ideas in a Smalltalk VM is the notion of the current *state of execution* or current *context*. The virtual machine has a call stack with one node per block or method currently in flight. The method call stack is implemented with parent pointers rather than an array of contexts. To pop something from the call stack, one moves up the parent chain one node just as we do for popping scopes in the scope tree. For your reference, here is the state associated with the overall virtual machine:

```java
public class VirtualMachine {
    /** The dictionary of global objects including class meta objects.
     *  The items are looked up by name (a string).
     */
    public final SystemDictionary systemDict; // singleton

    /** "This is the active context itself. It is either a MethodContext
     *  or a BlockContext." BlueBook p 605 in pdf.
     *  "The context that represents the CompiledMethod or block currently being
     *   executed is called the active context." p583.
     *
     *  In this impl, I use BlockContext for both methods and blocks.
     */
    public BlockContext ctx;
```

As the VM loads `.sto` (compiled Smalltalk) files, it creates a `STMetaClassObject` for each Smalltalk class. This metaclass is analogous to the `STClass` in the compiler. `STMetaClassObject` tracks:

* name
* superclass name
* literals table where all string literals from the bite code are stored
* list of field names
* list of methods (a `Map<String,STCompiledBlock>`)

The overall virtual machine state is just the call stack but for each block or method execution, we need the following information to represent the state of execution:

```java
public class BlockContext {
	/** The context that was active when this context was created; i.e.,
	 *  who invoked me?
	 */
	public BlockContext invokingContext;
	
	/** The receiver of the message that resulted in this context */
	public final STObject receiver;
	
	/** The compiled code associated with this context */
	public final STCompiledBlock compiledBlock;
	
	/** All arguments and local variables associated with this block */
	public final STObject[] locals;
	
	/** The instruction pointer that points into compiledBlock.bytcodes */
	public int ip = 0;
	
	/** The operand stack for this context */
	public STObject[] stack;
	
	/** The operand stack pointer for this context; points at stack top */
	public int sp = -1;
```

Unlike the virtual machines we built in class, this VM has a separate operand stack and instruction pointer per method or block execution. 

### Bytecode semantics

<img src=images/smalltalk-bytecodes.png width=700>

### Primitive methods

Primitive Smalltalk methods are like `native` methods in java and Smalltalk code can invoke them like regular methods. Although we have a great deal of our Smalltalk implementation in Smalltalk, to bootstrap we ultimately need to execute some Java. Primitive methods are the interface between Smalltalk and the underlying implementation. For example:

```
class String : Object [
   asArray <primitive:#String_ASARRAY>
]
```

From your perspective, the only thing you care about is that the expected output has no bytecode disassembly:

```
name: String
superClass: 
fields: 
literals: 
methods:
    name: asArray
    qualifiedName: String>>asArray
    nargs: 0
    nlocals: 0
```

There is a `STCompiledBlock` for each Smalltalk-based method, such as `hash`, but there is also one for primitive methods. The difference is that `STCompiledBlock` objects for primitive methods have field `primitiveName` non-null:

```java
/** In the compiler, this is the primitive name. In the VM, the equivalent
 *  class has a 'primitive' field that points at an actual Primitive object.
 */
public final String primitiveName;
```

### Class methods

Smalltalk has class methods just like Java does and we use the `class` keyword on methods in our Smalltalk to indicate which methods are class methods. As with regular methods, class methods can also have primitive implementations:
 
```
class Object [
    class basicNew <primitive:#Object_Class_BASICNEW>
    class new [ ^self basicNew initialize ]
    initialize ["do nothing by default" ^self]
	...
]
```

Class methods are treated no differently than instance methods except that we turn on `isClassMethod` in `STMethod` and then `STCompiledBlock`.  Class methods only work on Smalltalk `Class` (Java `STMetaClassInfo`) objects but we rely on the programmer to avoid sending class messages to instances (see [testClassMessageOnInstanceError](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/test/smalltalk/compiler/test/exec/TestCore.java#L1196)) and vice versa (see [testInstanceMessageOnClassError](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler/blob/master/test/smalltalk/compiler/test/exec/TestCore.java#L1233)). For example, `Array new` makes sense because `new` is a class method but `x new` for some instance `x` does not. Similarly, `names size` makes sense but `Array size` does not.

Please note that `self` in a class method refers to the class definition object and not an instance of the class.

## Compilation

[Compiler starter kit](https://github.com/USF-CS652-starterkits/parrt-smalltalk-compiler).

For the constructs as shown below in the compilation rules, use visitor methods to compute the `Code` result for each particular construct. As a side effect, you will be tracking literals within blocks/methods. Further, you will be setting the `compiledBlock` field of each `STBlock`/`STMethod`. In a sense, the result of compilation is the scope tree (`STSymbolTable`) decorated with multiple `STCompiledBlock`s. The `STClass` objects will be installed in the `SystemDictionary` of a VM prior to execution:

<img src="images/smalltalk-blocks.png" width="700" align=middle>

<img src="images/smalltalk-expr.png" width="700" align=middle>

<img src="images/smalltalk-msgs.png" width="700" align=middle>

To help you visualize the data structures involved in compilation and how the compiled code relates to the symbol table, imagine compiling the following main method body.

```
[true] whileTrue: [ ]
```

There is an implied `MainClass` and `main` method within that class. There are also two local scopes, the two code blocks `[true]` and `[ ]`. The scope tree looks like you would expect, as on the left of the following diagram.

<img src="images/smalltalk-symtab-compiled.png" width=700 align=middle>

Your code generator needs to compile the method and the nested blocks then update pointers from the symbol table to the compiled code. Without those pointers to the compiled blocks, all of your hard work in the compilation phase would disappear.  For every `STBlock`/`STMethod` in the program, you will have a `STCompiledBlock`.  You also have a bit of bookkeeping work to do regarding the blocks. The compiled block for a method keeps an array of pointers to compiled blocks for any blocks nested anywhere in the method. Note that these references will point at the same place that the symbol table scopes point.

### DBG instructions

The DBG instructions inform the VM where in the original Smalltalk source code the subsequent bytecode instruction(s) comes from. The debug information is extremely useful when writing Smalltalk code, although it can be useful when debugging the VM itself. You need to insert DBG instructions in the following locations:

1. `visitMain`. At the end of the body, before pop, self, return.
2. `visitSmalltalkMethodBlock`.  After visiting the children, before the pop, self, ...
3. `visitAssign`

		//At the end
		if (compiler.genDbg) {
		    code = dbg(ctx.start).join(code);
		}
		//before return
4. `visitKeywordSend`

		//After getLiteralIndex()
		if(compiler.genDbg){
		    code = Code.join(code, dbg(ctx.KEYWORD(0).getSymbol()));
		}
		//Before you join code for Send
5. `visitBinaryExpression`

		//After you join code for visitUnaryExpression(1)
		if(compiler.genDbg){
		    code = Code.join(dbg(ctx.bop(i-1).getStart()), code);
		}
		//Before you join code for Send
6. `visitBlock`.

		//After you join code for visitChildren()
		if(compiler.genDbg){
		    code = Code.join(code, dbgAtEndBlock(ctx.stop));
		}
		//Before you join push_block_return
7. `visitEmptyBody`

		//At the end
		if(compiler.genDbg){
		    code = Code.join(code, dbgAtEndBlock(ctx.stop));
		}
		//Before you return
8. `visitReturn`

		//After visitChildren
		if (compiler.genDbg) {
		    e = Code.join(e, dbg(ctx.start)); // put dbg after expression as that is when it executes
		}
		//Before you join code for method_return()
9. `visitUnaryMsgSend`

		//At the end
		if (compiler.genDbg) {
		    code = Code.join(dbg(ctx.ID().getSymbol()), code);
		}
		//Before return

## Tasks

Most of the compiler is given to you, but you need to build the `CodeGenerator.java` parse tree **visitor**, which is the meat of the compiler.  Pieces of `Compiler.java` must also be filled in.

To test from the command-line, this `alias` is useful:

```bash
alias stc='java -cp "/Users/parrt/.m2/repository/edu/usfca/cs652/smalltalk-compiler/1.0/smalltalk-compiler-1.0-complete.jar:$CLASSPATH" smalltalk.compiler.STC'
```

where `parrt` should be replaced with your login name. Also, this assumes you've done `mvn install`, which creates the .jar file.

There are **136 unit tests** that your compiler should pass. Those in `exec` subdir are invoking the VM (`lib/smalltalk-vm-1.0.jar`) to execute code compiled by your compiler.