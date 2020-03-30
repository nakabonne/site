---
title: "Walk the Go AST in depth-first"
description: "Deepen your understanding of Go's AST by actually walking through it."
date: 2020-03-30
tags: ["Golang"]
draft: false
images: ["/img/foo.png"]
---

ドキュメントを見ればASTの全てのNodeの説明はある。しかしイメージが付きづらいのが現実。なので、タイトルの通りサンプルコードを実際に深さ優先探索でinspectしていきながら、現れたNodeの解説をしていくことで、私達が普段書いているGoのコードが内部でどのように表現されているかを理解する。

この記事ではソースコードをパースする方法には触れず、ASTが構築された後について説明します。コードがASTに変換される方法について気になる人は、 [Digging deeper into the analysis of Go-code](https://nakabonne.dev/posts/digging-deeper-into-the-analysis-of-go-code/) にnavigate toしてください。

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

## Getting started to walk
早速歩き始めましょう。私達がASTに変換するコードはこちらです。

```go
package testpkg

import "fmt"

func foo() {
	fmt.Println("Hello, World")
}
```

Nothing fancy — an overly simple Hello, World program. The AST built on this is:

<details>
  <summary>ast.File</summary>
  
```go
*ast.File {
.  Package: foo.go:1:1
.  Name: *ast.Ident {
.  .  NamePos: foo.go:1:9
.  .  Name: "testpkg"
.  }
.  Decls: []ast.Decl (len = 2) {
.  .  0: *ast.GenDecl {
.  .  .  TokPos: foo.go:3:1
.  .  .  Tok: import
.  .  .  Lparen: -
.  .  .  Specs: []ast.Spec (len = 1) {
.  .  .  .  0: *ast.ImportSpec {
.  .  .  .  .  Path: *ast.BasicLit {
.  .  .  .  .  .  ValuePos: foo.go:3:8
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
.  .  .  .  NamePos: foo.go:5:6
.  .  .  .  Name: "foo"
.  .  .  .  Obj: *ast.Object {
.  .  .  .  .  Kind: func
.  .  .  .  .  Name: "foo"
.  .  .  .  .  Decl: *(obj @ 23)
.  .  .  .  }
.  .  .  }
.  .  .  Type: *ast.FuncType {
.  .  .  .  Func: foo.go:5:1
.  .  .  .  Params: *ast.FieldList {
.  .  .  .  .  Opening: foo.go:5:9
.  .  .  .  .  Closing: foo.go:5:10
.  .  .  .  }
.  .  .  }
.  .  .  Body: *ast.BlockStmt {
.  .  .  .  Lbrace: foo.go:5:12
.  .  .  .  List: []ast.Stmt (len = 1) {
.  .  .  .  .  0: *ast.ExprStmt {
.  .  .  .  .  .  X: *ast.CallExpr {
.  .  .  .  .  .  .  Fun: *ast.SelectorExpr {
.  .  .  .  .  .  .  .  X: *ast.Ident {
.  .  .  .  .  .  .  .  .  NamePos: foo.go:6:2
.  .  .  .  .  .  .  .  .  Name: "fmt"
.  .  .  .  .  .  .  .  }
.  .  .  .  .  .  .  .  Sel: *ast.Ident {
.  .  .  .  .  .  .  .  .  NamePos: foo.go:6:6
.  .  .  .  .  .  .  .  .  Name: "Println"
.  .  .  .  .  .  .  .  }
.  .  .  .  .  .  .  }
.  .  .  .  .  .  .  Lparen: foo.go:6:13
.  .  .  .  .  .  .  Args: []ast.Expr (len = 1) {
.  .  .  .  .  .  .  .  0: *ast.BasicLit {
.  .  .  .  .  .  .  .  .  ValuePos: foo.go:6:14
.  .  .  .  .  .  .  .  .  Kind: STRING
.  .  .  .  .  .  .  .  .  Value: "\"bar\""
.  .  .  .  .  .  .  .  }
.  .  .  .  .  .  .  }
.  .  .  .  .  .  .  Ellipsis: -
.  .  .  .  .  .  .  Rparen: foo.go:6:19
.  .  .  .  .  .  }
.  .  .  .  .  }
.  .  .  .  }
.  .  .  .  Rbrace: foo.go:7:1
.  .  .  }
.  .  }
.  }
.  Scope: *ast.Scope {
.  .  Objects: map[string]*ast.Object (len = 1) {
.  .  .  "foo": *(obj @ 27)
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

  
これを深さ優先でトラバースしていくわけですが、以下のように[`ast.Inspect()`](https://pkg.go.dev/go/ast?tab=doc#Inspect)を使って`ast.Print`を再帰的に呼ぶことで、一つひとつのNodeを表示していきます:

<details>
  <summary>How to walk</summary>
  
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
      // Prints recursively with depth first.
		ast.Print(fset, n)
		return true
	})
}

var src = `package testpkg

import "fmt"

func foo() {
	fmt.Println("bar")
}
`
```
</details>
  


### ast.File
最初のNodeは、全てのルートであるast.Fileです。
