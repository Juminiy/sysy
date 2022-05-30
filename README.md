# pku-minic

## Pre

### Aim

1. SysY Lang Compiles to Koopa IR
2. Koopa IR Generates RISC-V ASM 

### What OJ Do

- Use Def IR：SysY Code (test code) -> SysY compiler(Advanced Lang Source Code writes the software)，expected output is Koopa IR ,the OJ will exam it . 
- Self design IR：SysY to RISC-V,the ir will be ignored by OJ,the OJ exam the RISC-V ASM only.
- Performance Ranking.

### Tools

Learned and Used

- ✔Linux 

- ✔Git

Need to learning ... 

- ❌GDB/LLDB

- ❌Makefile

## Lv0

> List

- Docker
- Koopa IR
- RISC-V
- Choose A Advanced Lang as  Source Lang

### Docker Env

````bash
docker pull maxxing/compiler-dev
docker run -dit --name cl-env -v /home/ubuntu/code/sysy:/root/sysy maxxing/compiler-dev 
# trap into container bash
docker exec -it cl-env /bin/bash
````

the doc says that don't recommand the ssh in container env .

### Koopa IR

compile ir to executable file

````bash
koopac hello.koopa | llc --filetype=obj -o hello.o
clang hello.o -L$CDE_LIBRARY_PATH/native -lsysy -o hello
./hello
````

### RISC-V IS

why make the jokes risc-v with intel x86 arch ?

base instruction set and extension

- RV32I
- RV64I, compatible with RV32I

Extension Include :

- `M`: multiply,division instruction
- `A`: atomic memory instruction
- `F`: single precision float instruction
- `D`: double precision float instruction 
- `C`: casual instructions 16bits zipvesion 

Example：we choose `RV32IM` arch platform IS, refer to 32bits support extension `M` and `A` RISC-V Processor

compile risc-v asm to executable file

````bash
clang hello.S -c -o hello.o -target riscv32-unknown-linux-elf -march=rv32im -mabi=ilp32
ld.lld hello.o -L$CDE_LIBRARY_PATH/riscv32 -lsysy -o hello
qemu-riscv32-static hello
````

### Advanced Lang 

1. We must choose C/C++/Rust to develop our compiler
2. We must use Make/CMake/Cargo config file,thus OJ can compile our compiler 

I choose C,Makefile, Docker mount localfiledir.

## Lv1

### Aim & Goal

the compiler can transfer code from sysy to koopa ir 

````sysy
int main() {
	// comment
	return 0 ;
}
````

the compiler can translate the code to koopa IR

````koopa
func @main(): i32 {
%entry:
	ret 0
}
````

### SysY EBNF 

IDENT ebnf:

````ebnf
identifier ::= identifier-nondigit | identifier identifier-nondigit | identifier digit;

// INT_CONST
integer-const ::= decimal-const | octal-const | hexadecimal-const;
decimal-const ::= nonzero-digit | decimal-const digit;
octal-const ::= "0" | octal-const octal-digit;
hexadecimal-const ::= hexadecimal-prefix hexadecimal-digit | hexadecimal-const hexadecimal-digit;
hexadecimal-prefix  ::= "0x" | "0X";
````

reference:

- `identifier-nondigit` [_A-Za-z]

- `digit` [0-9]

- `nonzero-digit` [1-9]

- `octal-digit` [0-7]

- `hexadecimal-digit`[0-9A-Fa-f]
- INT_CONST range [0,2<sup>31</sup>-1]

Comment:

- `// ` single line comment

- `/**/` multiple lines comment

Syntax ebnf :

````ebnf
CompUnit  ::= FuncDef;

FuncDef   ::= FuncType IDENT "(" ")" Block;
FuncType  ::= "int";

Block     ::= "{" Stmt "}";
Stmt      ::= "return" Number ";";
Number    ::= INT_CONST;
````

### Lv1.1

Compiler's Architecture 

compile from source to executable whole process:

1. compile source code to assembly
2. asm compilation to object file
3. obj link to executable file 

the whole compiler functions only need the process 1.

the whole compiler consists of :

1. Front end: do grammar and lexical analysis ,transfer `SysY` source code to AST( abstract syntax tree ),  Semantic Analysis ,scan the builed AST ,inspect the semantic errors, given the error hint called compiler output. 
2. Middle end: transfer AST to IR( Intermediate Representation ) , do some Machine independent optimization
3. Back end : transfer IR to target Platform assembly, do some Machine dependent optimization 

Lexical Analysis ：lexer read file, do from the byte stream to token，output token and pass token to parser , at same time , the lexer ignore the space tab or comment(// /* */) any lines .

````sysy
int main() {
	return 0 ; 
}
````

Lexical Analysis -> token steam is :

````
type		content
keywords	int
identifier	main
otherchar	(
otherchar	)
otherchar	{
keywords	return
constInt	0
otherchar	;
otherchar	}
````

grammar analysis -> ast is :

````
CompUnit {
  items: [
    FuncDef {
      type: "int",
      name: "main",
      params: [],
      body: Block {
        stmts: [
          Return {
            value: 0
          }
        ]
      }
    }
  ]
}
````

Semantic Analysis 

a program that obeys grammar maybe not obey the semantic .

````
int main() {
	int a = 1 ;
	int a = 2 ;
	return 0 ;
}
````

the compiler must throws the redefination error.

