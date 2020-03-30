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

#### *ast.ImportSpec

```go
*ast.ImportSpec {
.  Path: *ast.BasicLit {/* Omission */}
.  EndPos: -
}
```

[`*ast.ImportSpec`](https://pkg.go.dev/go/ast?tab=doc#ImportSpec) implements the [`ast.Spec`](https://pkg.go.dev/go/ast?tab=doc#Spec) interface.

*ast.BasicLit

```go
*ast.BasicLit {
.  ValuePos: dummy.go:3:8
.  Kind: STRING
.  Value: "\"fmt\""
}
```

### Func Declarations

```go
*ast.FuncDecl {
.  Name: *ast.Ident {/* Omission */}
.  Type: *ast.FuncType {/* Omission */}
.  Body: *ast.BlockStmt {/* Omission */}
}
```

*ast.Ident

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

*ast.FuncType

```go
*ast.FuncType {
.  Func: dummy.go:5:1
.  Params: *ast.FieldList {
.  .  Opening: dummy.go:5:11
.  .  Closing: dummy.go:5:12
.  }
}
```

*ast.FieldList

```go
*ast.FieldList {
.  Opening: dummy.go:5:11
.  Closing: dummy.go:5:12
}
```


```go
*ast.BlockStmt {
.  Lbrace: dummy.go:5:14
.  List: []ast.Stmt (len = 1) {
.  .  0: *ast.ExprStmt {/* Omission */}
.  }
.  Rbrace: dummy.go:7:1
}
```

```go
*ast.ExprStmt {
.  X: *ast.CallExpr {/* Omission */}
}
```

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

```go
*ast.SelectorExpr {
.  X: *ast.Ident {/* Omission */}
.  Sel: *ast.Ident {/* Omission */}
}
```


```go
*ast.Ident {
.  NamePos: dummy.go:6:2
.  Name: "fmt"
}
```

```go
*ast.Ident {
.  NamePos: dummy.go:6:6
.  Name: "Println"
}
```

```go
*ast.BasicLit {
.  ValuePos: dummy.go:6:14
.  Kind: STRING
.  Value: "\"Hello, World\""
}
```
