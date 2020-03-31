---
title: "Take a walk the Go AST"
description: "Deepen your understanding of Go's AST by actually walking through it."
date: 2020-03-30
tags: ["Golang"]
draft: false
images: ["/img/ast-file-tree.png"]
---

もしあなたがGoのASTについてcurious aboutした時、何を参照しますか？ドキュメント？ソースコード？
ドキュメントを読めば抽象的な理解はできますが、API同士の関連などを理解することはできません。
ソースコードを読めばそれらも理解出来ますが、全部読もうとするとかなり体力を使います。
なのでこの記事ではその中間となることを目指します。肩の力を抜いてASTを散歩することで私達が普段書いているGoのコードが内部でどのように表現されているかを理解しましょう。

この記事ではソースコードをパースする方法には触れず、ASTが構築された後から始めます。
コードがASTに変換される方法について気になる人は、 [Digging deeper into the analysis of Go-code](https://nakabonne.dev/posts/digging-deeper-into-the-analysis-of-go-code/) にnavigate toしてください。

Let’s get started.

## Interfaces
First up, let me briefly describe the interface that represents each of the AST nodes.
All AST nodes implement the [`ast.Node`](https://pkg.go.dev/go/ast?tab=doc#Node) interface, which just returns a position in the AST.  
In addtion, there are 3 main interfaces that implement `ast.Node`:

- [ast.Expr](https://pkg.go.dev/go/ast?tab=doc#Expr) — representing expressions and types node
- [ast.Stmt](https://pkg.go.dev/go/ast?tab=doc#Stmt) — representing statement node
- [ast.Decl](https://pkg.go.dev/go/ast?tab=doc#Decl) — representing declaration nodes.

{{< figure src="/img/node-interface.png" width="100%" height="auto">}}

定義を見てみても、たしかに全てのNodeはNodeインターフェイスを満たしていることがわかります:

[ast/ast.go](https://github.com/golang/go/blob/0b7c202e98949b530f7f4011efd454164356ba69/src/go/ast/ast.go#L32-L54)

```go
// All node types implement the Node interface.
type Node interface {
	Pos() token.Pos // position of first character belonging to the node
	End() token.Pos // position of first character immediately after the node
}

// All expression nodes implement the Expr interface.
type Expr interface {
	Node
	exprNode()
}

// All statement nodes implement the Stmt interface.
type Stmt interface {
	Node
	stmtNode()
}

// All declaration nodes implement the Decl interface.
type Decl interface {
	Node
	declNode()
}
```

## Getting started with walking
早速歩き始めましょう。私達がASTに変換するコードはこちらです。

```go
package hello

import "fmt"

func greet() {
	fmt.Println("Hello, World")
}
```

Nothing fancy — an overly simple Hello, World program. The AST built on this is:

<details>
  <summary>ast.File</summary>
  
```go
*ast.File {
.  Package: dummy.go:1:1
.  Name: *ast.Ident {
.  .  NamePos: dummy.go:1:9
.  .  Name: "hello"
.  }
.  Decls: []ast.Decl (len = 2) {
.  .  0: *ast.GenDecl {
.  .  .  TokPos: dummy.go:3:1
.  .  .  Tok: import
.  .  .  Lparen: -
.  .  .  Specs: []ast.Spec (len = 1) {
.  .  .  .  0: *ast.ImportSpec {
.  .  .  .  .  Path: *ast.BasicLit {
.  .  .  .  .  .  ValuePos: dummy.go:3:8
.  .  .  .  .  .  Kind: STRING
.  .  .  .  .  .  Value: "\"fmt\""
.  .  .  .  .  }
.  .  .  .  .  EndPos: -
.  .  .  .  }
.  .  .  }
.  .  .  Rparen: -
.  .  }
.  .  1: *ast.FuncDecl {
.  .  .  Name: *ast.Ident {
.  .  .  .  NamePos: dummy.go:5:6
.  .  .  .  Name: "greet"
.  .  .  .  Obj: *ast.Object {
.  .  .  .  .  Kind: func
.  .  .  .  .  Name: "greet"
.  .  .  .  .  Decl: *(obj @ 23)
.  .  .  .  }
.  .  .  }
.  .  .  Type: *ast.FuncType {
.  .  .  .  Func: dummy.go:5:1
.  .  .  .  Params: *ast.FieldList {
.  .  .  .  .  Opening: dummy.go:5:11
.  .  .  .  .  Closing: dummy.go:5:12
.  .  .  .  }
.  .  .  }
.  .  .  Body: *ast.BlockStmt {
.  .  .  .  Lbrace: dummy.go:5:14
.  .  .  .  List: []ast.Stmt (len = 1) {
.  .  .  .  .  0: *ast.ExprStmt {
.  .  .  .  .  .  X: *ast.CallExpr {
.  .  .  .  .  .  .  Fun: *ast.SelectorExpr {
.  .  .  .  .  .  .  .  X: *ast.Ident {
.  .  .  .  .  .  .  .  .  NamePos: dummy.go:6:2
.  .  .  .  .  .  .  .  .  Name: "fmt"
.  .  .  .  .  .  .  .  }
.  .  .  .  .  .  .  .  Sel: *ast.Ident {
.  .  .  .  .  .  .  .  .  NamePos: dummy.go:6:6
.  .  .  .  .  .  .  .  .  Name: "Println"
.  .  .  .  .  .  .  .  }
.  .  .  .  .  .  .  }
.  .  .  .  .  .  .  Lparen: dummy.go:6:13
.  .  .  .  .  .  .  Args: []ast.Expr (len = 1) {
.  .  .  .  .  .  .  .  0: *ast.BasicLit {
.  .  .  .  .  .  .  .  .  ValuePos: dummy.go:6:14
.  .  .  .  .  .  .  .  .  Kind: STRING
.  .  .  .  .  .  .  .  .  Value: "\"Hello, World\""
.  .  .  .  .  .  .  .  }
.  .  .  .  .  .  .  }
.  .  .  .  .  .  .  Ellipsis: -
.  .  .  .  .  .  .  Rparen: dummy.go:6:28
.  .  .  .  .  .  }
.  .  .  .  .  }
.  .  .  .  }
.  .  .  .  Rbrace: dummy.go:7:1
.  .  .  }
.  .  }
.  }
.  Scope: *ast.Scope {
.  .  Objects: map[string]*ast.Object (len = 1) {
.  .  .  "greet": *(obj @ 27)
.  .  }
.  }
.  Imports: []*ast.ImportSpec (len = 1) {
.  .  0: *(obj @ 12)
.  }
.  Unresolved: []*ast.Ident (len = 1) {
.  .  0: *(obj @ 46)
.  }
}
```
</details>

#### How to walk
All we have todo is traverse this AST node in depth-first order.
Let's print each Node one by one by calling [`ast.Inspect()`](https://pkg.go.dev/go/ast?tab=doc#Inspect) recursively.

Also, printing AST directly then we will typically see stuff that is not human readable. 
To prevent that from happening, we're going to use [`ast.Print`](https://pkg.go.dev/go/ast?tab=doc#Print), a powerful API for human reading of AST:

<details>
  <summary>walk.go</summary>
  
```go
package main

import (
	"fmt"
	"go/ast"
	"go/parser"
	"go/token"
)

func main() {
	fset := token.NewFileSet()
	f, _ := parser.ParseFile(fset, "dummy.go", src, parser.ParseComments)

	ast.Inspect(f, func(n ast.Node) bool {
        // Called recursively.
		ast.Print(fset, n)
		return true
	})
}

var src = `package hello

import "fmt"

func greet() {
	fmt.Println("Hello, World")
}
`
```
</details>
  


### ast.File
最初のNodeは、全てのルートであるast.Fileです。
ast.Nodeを実装しています。


{{< figure src="/img/ast-file-tree.png" width="100%" height="auto">}}

Fileは大まかにこれらを子ノードとして持っています。厳密にはCommentsなどもありますが、今回は省略します。まずはPackage Nameから見ていきましょう。

Note that fields with a value of nil are omitted. See [the document](https://pkg.go.dev/go/ast) for a complete list of fields for each node type.

### Package Name

#### *ast.Ident

```go
*ast.Ident {
.  NamePos: dummy.go:1:9
.  Name: "hello"
}
```

A package name can be represented by the AST node type [`*ast.Ident`](https://pkg.go.dev/go/ast?tab=doc#Ident), which implements the `ast.Expr` interface.
All identifiers are represented by this structure. It mainly contains its name and a source position within a file set.  
From the code shown above, we can see that the package name is `hello` and is declared in the first line of `dummy.go`.

We can't dive any deeper into this node, let's go back to the top level.

### Import Declarations

#### *ast.GenDecl

```go
*ast.GenDecl {
.  TokPos: dummy.go:3:1
.  Tok: import
.  Lparen: -
.  Specs: []ast.Spec (len = 1) {
.  .  0: *ast.ImportSpec {/* Omission */}
.  }
.  Rparen: -
}
```
A declaration of import is represented by the AST node type `ast.GenDecl`, which implements the `ast.Decl` interface.
`ast.GenDecl` represents all declarations except for functions; That is, import, const, var, and type.

`Tok` represents a lexical token — which is specifies what the declaration is about depending on its implementation (IMPORT or CONST or TYPE or VAR).  
This AST Node tells us that the import declaration is on line 3 in dummy.go.

Let's visit `ast.GenDecl` in depth-first order. Take a look `*ast.ImportSpec`, the next Node.

#### *ast.ImportSpec

```go
*ast.ImportSpec {
.  Path: *ast.BasicLit {/* Omission */}
.  EndPos: -
}
```
An [`*ast.ImportSpec`](https://pkg.go.dev/go/ast?tab=doc#ImportSpec) node corresponds to a single import declaration.
It implements the [`ast.Spec`](https://pkg.go.dev/go/ast?tab=doc#Spec) interface.
Visiting `Path` could make more sense about the import path. Let's go there.

#### *ast.BasicLit

```go
*ast.BasicLit {
.  ValuePos: dummy.go:3:8
.  Kind: STRING
.  Value: "\"fmt\""
}
```

A [`*ast.BasicLit`](https://pkg.go.dev/go/ast?tab=doc#BasicLit) node represents a literal of basic type.
It implements the `ast.Expr` interface.
This contains a type of token and `token.INT`, `token.FLOAT`, `token.IMAG`, `token.CHAR`, or `token.STRING` can be used.  
From `*ast.ImportSpec` and `*ast.BasicLit`, we can see it has imported package called "fmt".

We can't dive any deeper, let's get back to the top level again.

### Function Declarations

#### *ast.FuncDecl

```go
*ast.FuncDecl {
.  Name: *ast.Ident {/* Omission */}
.  Type: *ast.FuncType {/* Omission */}
.  Body: *ast.BlockStmt {/* Omission */}
}
```

A [`*ast.FuncDecl`](https://pkg.go.dev/go/ast?tab=doc#FuncDecl) node represents a function declaration.
It implements only the `ast.Node` interface.
Let's take a look at them in order from `Name`, representing a function name.

#### *ast.Ident

```go
*ast.Ident {
.  NamePos: dummy.go:5:6
.  Name: "greet"
.  Obj: *ast.Object {
.  .  Kind: func
.  .  Name: "greet"
.  .  Decl: *(obj @ 0)
.  }
}
```

The second time this has appeared, let me skip the basic explanation.

Noteworthy is the [`*ast.Object`](https://pkg.go.dev/go/ast?tab=doc#Object).
It represents the object to which the identifier refers, but why is this needed?
As you know, Go has a concept of scope, which is the extent of source text in which the identifier denotes the specified constant, type, variable, function, label, or package.

The `Decl` field indicates where the identifier was declared so that it identifies the scope of the identifier.
Identifiers that point to the identical object share the identical `*ast.Object`.

#### *ast.FuncType

```go
*ast.FuncType {
.  Func: dummy.go:5:1
.  Params: *ast.FieldList {/* Omission */}
}
```
Go back to being a parent one generation older, a [`*ast.FuncType`](https://pkg.go.dev/go/ast?tab=doc#FuncType) contains a function signature including parameters, results, and position of "func" keyword.

#### *ast.FieldList

```go
*ast.FieldList {
.  Opening: dummy.go:5:11
.  List: nil
.  Closing: dummy.go:5:12
}
```

A [`*ast.FieldList`](https://pkg.go.dev/go/ast?tab=doc#FieldList) node represents a list of Fields, enclosed by parentheses or braces.
Function parameters would be shown here if they are defined, but this time none, so no information.

`List` field is a slice of [`*ast.Field`](https://pkg.go.dev/go/ast?tab=doc#Field) that contains a pair of identifiers and types.
It is highly versatile and is used for a variety of Nodes, including [`*ast.StructType`](https://pkg.go.dev/go/ast?tab=doc#StructType), [`*ast.InterfaceType`](https://pkg.go.dev/go/ast?tab=doc#InterfaceType), and here.
That is, it's needed when mapping a type to an identifier:

```go
foo int
bar string
```

Let's loop back to `*ast.FuncDecl` again and dive a bit into `Body`, the last field.

#### *ast.BlockStmt

```go
*ast.BlockStmt {
.  Lbrace: dummy.go:5:14
.  List: []ast.Stmt (len = 1) {
.  .  0: *ast.ExprStmt {/* Omission */}
.  }
.  Rbrace: dummy.go:7:1
}
```

A [`BlockStmt`](https://pkg.go.dev/go/ast?tab=doc#BlockStmt) node represents a braced statement list.
It implements  the `ast.Stmt` interface.
It does have a list of statements. What an imaginable node!

#### *ast.ExprStmt

```go
*ast.ExprStmt {
.  X: *ast.CallExpr {/* Omission */}
}
```

[`*ast.ExprStmt`](https://pkg.go.dev/go/ast?tab=doc#ExprStmt) represents an expression in a statement list.
It implements the `ast.Stmt` interface and contains a single `ast.Expr`.

#### *ast.CallExpr

```go
*ast.CallExpr {
.  Fun: *ast.SelectorExpr {/* Omission */}
.  Lparen: dummy.go:6:13
.  Args: []ast.Expr (len = 1) {
.  .  0: *ast.BasicLit {/* Omission */}
.  }
.  Ellipsis: -
.  Rparen: dummy.go:6:28
}
```

[`*ast.CallExpr`](https://pkg.go.dev/go/ast?tab=doc#CallExpr) represents an expression that calls a function.
The fields to look at are the function to call and the list of arguments to pass to it.

#### *ast.SelectorExpr

```go
*ast.SelectorExpr {
.  X: *ast.Ident {
.  .  NamePos: dummy.go:6:2
.  .  Name: "fmt"
.  }
.  Sel: *ast.Ident {
.  .  NamePos: dummy.go:6:6
.  .  Name: "Println"
.  }
}
```

[`*ast.SelectorExpr`](https://pkg.go.dev/go/ast?tab=doc#SelectorExpr) represents an expression followed by a selector.
Simply put, it means `fmt.Println`.

#### *ast.BasicLit

```go
*ast.BasicLit {
.  ValuePos: dummy.go:6:14
.  Kind: STRING
.  Value: "\"Hello, World\""
}
```

No longer needed an explanation, Hello, World!

## Bottom Line
I've left out some of the fields in the node types I've introduced.
There are still many other node types.

Nevertheless, I'd say it's significant to actually walk the walk.
Copy and paste the code shown in section "How to walk", and try to walk around on your PC.
