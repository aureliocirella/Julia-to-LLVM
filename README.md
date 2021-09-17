From Julia to LLVM
==================

Background
----------

This is the implementation of a compiler which recognizes and translates part of the Julia programming language into the LLVM IR syntax (more information about LLVM can be found [here](https://www.skenz.it/compilers/llvm "https://www.skenz.it/compilers/llvm")).

### Implemented parts

List of Julia parts supported:

-   **Data types:**

    -   Basic data types:

        -   int

        -   doubles

        -   1-dimension array 

-   (Local scope for variables is implemented)

-   **Operators:**

    -   arithmetic:

        -   sum (**+**)

        -   difference (**-**)

        -   multiplication (*)

        -   division (**/**)

    -   compound:

        -   sum (**+=**)

        -   difference (**-=**)

        -   multiplication (***=**)

        -   division (**/=**)

    -   comparison:

        -   equality(**==**)

        -   greater than (**>**)

        -   less than (**<**)

        -   greater than equal to (**>=**)

        -   less than equal to (**<=**)

    -   logic:

        -   AND (**&&**)

        -   OR (**||**)

-   **Control flow instructions:**

    -   if instruction

    -   if-then-else instruction

    -   for loop over a range

    -   while loop

-   **Function details (partial implementation):**

    -   double input parameters 

    -   double return value 

-   **Output:**

    -   print/println 

    -   with and without string interpolation ("x value is: $x")

### Compiler

The compiler is made up of two parts: a scanner and a parser.

Scanner
-------

The scanner is able to recognize and retrieve tokens (terminal symbols) to the parser coupled with an object containing a value that the token represents. It identifies integer, doubles and ids (that will be used for variables, function names, ...) and other significant Julia keywords like:

-   else

-   end

-   for

-   function

-   if

-   in

-   print

-   println

-   return

-   while

And other syntax elements like punctuation and other symbols. 

Parser
------

The parser is able to take as input the tokens that are provided by the scanner and recognize the main grammatical rules of Julia language. Consequently, the correspondend LLVM IR code is produced.


### Grammar start

The grammar defined for the parser assumes that there are function definitions first and then there are the instructions that will be included in the `@main` of the LLVM IR code.  `printBuff` is first used to store (and eventually write on `output.ll`) the recognized instructions. After that the non terminal `function_defs` is recognized, the parser write the instructions correspondent to the functions defined. Then it clears the `printBuffer` and `var_label` counter that will be then used for writing instructions of the `@main`. `BufferedWriter bwr` is the buffer used to create an `output.ll` file and write the LLVM IR code in it. 


### Some practical examples

##### Recognition of scalars and variable IDs

`val token` is used to represent any scalar entity in the parser: it represents integer, double and variable names. Each time an `INT` or `DOUBLE` token is recognized, the parser defines a new `TypeVar` and allocate the needed memory locations for that scalars and the stores them. While when an `ID` token is recognized, the parser checks if the `ID` is already defined in the `symbolTable`, if it is, then assign to the `val` token the `TypeVar` that the `ID` is coupled to.
```
{
val ::= ID:x {:
    if(!parser.symbolTable.containsKey(x))
    {
        pSemError("Variable "+x+" not declared.");
    }else{
        RESULT = parser.symbolTable.get(x);

    }

:}
| INT:x {:
    RESULT = new TypeVar(x, "i32", new [Integer](http://www.google.com/search?hl=en&q=allinurl%3Adocs.oracle.com+javase+docs+api+integer)(4));
    append("%"+RESULT.reg_id+" = alloca "+RESULT.type+", align "+RESULT.align, true);
    append("store "+RESULT.type+" "+RESULT.value+", "+RESULT.type+"* %"+RESULT.reg_id, true);
:}
| DOUBLE:x {:
    RESULT = new TypeVar(x, "double", new [Integer](http://www.google.com/search?hl=en&q=allinurl%3Adocs.oracle.com+javase+docs+api+integer)(8));
    append("%"+RESULT.reg_id+" = alloca "+RESULT.type+", align "+RESULT.align, true);
    append("store "+RESULT.type+" "+RESULT.value+", "+RESULT.type+"* %"+RESULT.reg_id, true);
:}
;
}
```

##### Array definition

This rules uses `elem_list` non terminal symbol that represents a list of square brackets enclosed `val` tokens. `elem_list` is coupled with an `ArrayList<TypeVar>` storing the values stated in the brackets. The parser allocates the needed memory for the array elements and then it stores each array element in the location at proper position within the just allocated memory locations. 

For example, given the following code: 

```
{ a = [1,2,3] }
```

Here's the correspondent LLVM IR code: 

```
{
  %1 = alloca i32, align 4
  store i32 1, i32* %1
  %2 = alloca i32, align 4
  store i32 2, i32* %2
  %3 = alloca i32, align 4
  store i32 3, i32* %3
  %4 = alloca [3 x i32], align 4
  %5 = getelementptr inbounds [3 x i32], [3 x i32]* %4, i32 0, i32 0
  %6 = load i32, i32* %1, align 4
  store i32 %6, i32* %5, align 4
  %7 = getelementptr inbounds [3 x i32], [3 x i32]* %4, i32 0, i32 1
  %8 = load i32, i32* %2, align 4
  store i32 %8, i32* %7, align 4
  %9 = getelementptr inbounds [3 x i32], [3 x i32]* %4, i32 0, i32 2
  %10 = load i32, i32* %3, align 4
  store i32 %10, i32* %9, align 4
}
```

### Function implementation

The following code shows how LLVM IR code is produced for function definition. *Remember: function definition implementation only supports doubles as input and return parameters.*

When the function definition is recognized, the parser produces the function definition in LLVM IR code. Before parsing the function statements, the input parameters are stored in new registers. 

Let's consider the following function definition in Julia:
```
{
  function division(a,b)

  ...

  end
}
```

The correspondent LLVM IR code will be: 

```
{
  define double @division(double, double) {
  %3 = alloca double, align 8
  store double %0, double* %3
  %4 = alloca double, align 8
  store double %1, double* %4
  ...
  }
}
```

### Print instruction

When a string is recognized by the parser, it calls the `CreateString` functions that defines the string declaration as a global constant. This function is also able to recognize if there's interpolation of variables within the string (e.g. "$variable_name"). Then, for simplicity and for exploit this CreateString functionality, when a variable is printed the parser encapsulates that variable in a string and pre-place a $ symbol to it. If a numerical value is printed, the operation of pre-placing a $ symbol is obviously not done. If an array element is printed, `CreateString` is not called but the parser directly produces the needed LLVM IR code. 


### Error handling

The compiler is able to recognize the following kind of errors:

-   Variable not declared 

-   Variable is not an array 

-   Function not defined

-   Generic error in assignment 

-   Missing ] in array definition

-   Missing ] in matrix definition

-   General error in while condition 

-   Missing ) in while condition 

-   General error in if condition 

-   General error in print instruction

### Missing functionalities, partial implementations

-   Matrix and touples grammar rules defined but their usage is not supported

-   Functions work only with only doubles as input and return parameters 

-   String assignment to variable not supported

-   Doubles are supported for arithmetic operations but not for other code functionalities (e.g. no support for array of double,...)

-   Print instruction doesn't support printing the result of operations (e.g. print(1+2))

-   No support for global variables

Download and installation
-------------------------

### How to run it

1.  Install the *llvm* package: `sudo apt install llvm`

2.  Download the *julia_compiler* and extract it

3.  Open the terminal, go to the folder where the compiler is extracted and run the following commands: 

    -   `jflex julia_scanner.jflex`

    -   `java java_cup.Main -expect 8 julia_parser.cup`

    -   `javac *.java`

    -   `java Main source.jl`

4.  This will produce an `output.ll` file 

5.  Run output.ll file with *lli*: `lli output.ll`
