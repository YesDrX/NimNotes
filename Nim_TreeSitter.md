## What's this note about?
    To explore how to use tree-sitter in nim.
## Prepare

#### install treesitter package for [nim](https://github.com/genotrance/nimtreesitter)
```bash
  nimble install treesitter
```
#### compile a grammar using tree-sitter (python)
  assume nodejs, build-essential are installed
```zsh
  npm install -g yarn
  npm install -g node-gyp

  git clone https://github.com/tree-sitter/tree-sitter-python #grammar.js defines the grammars for python
  cd tree-sitter-python
  export PATH=$PATH:./node_modules/.bin
  yarn
  tree-sitter generate; node-gyp configure; node-gyp build; tree-sitter test # compiled .so file is saved in $HOME/.tree-sitter/bin by default
```

#### write a simple function printTree to prety print one AST

```nim
import treesitter/api
import strutils

const
    indentStr = " ".repeat(4)

proc treeSitterPython() : ptr TSLanguage {.stdcall, importc : "tree_sitter_python", dynlib : "/home/YesDrX/.tree-sitter/bin/python.so".}

proc getStartIdx(node : TSNode) : (int, int) = 
    return (node.tsNodeStartPoint.row.int, node.tsNodeStartPoint.column.int)

proc getEndIdx(node : TSNode) : (int, int) = 
    return (node.tsNodeEndPoint.row.int, node.tsNodeEndPoint.column.int)

proc getSliceIdx(lineLen : seq[int], rowIdx : int, colIdx : int) : int =
    result = 0
    for idx in 0 .. rowIdx-1:
        result += lineLen[idx]
    result += colIdx

proc getNodeText(node : TSNode, lineLen : seq[int], code : string) : string =
    var
        startRowColIdx = getStartIdx(node)
        endRowColIdx = getEndIdx(node)
        startIdx = getSliceIdx(lineLen, startRowColIdx[0], startRowColIdx[1])
        endIdx = getSliceIdx(lineLen, endRowColIdx[0], endRowColIdx[1])
    result = code[startIdx .. endIdx-1]

proc printNode(indents : int, node : TSNode, lineLen : seq[int], code : string) : string =
    result = indentStr.repeat(indents)
    result &= '"' & $node.tsNodeType & '"'
    result &= ",[" & $getStartIdx(node) & "," & $getEndIdx(node) & "]"
    result &= ": " & '"' & getNodeText(node, lineLen, code) & '"'

proc printTree(indents : int = 0, node : TSNode, lineLen : seq[int], code : string) =
    var
        cursorObj = node.tsTreeCursorNew
        cursor = cursorObj.addr
    echo printNode(indents, node, lineLen, code)
    if cursor.tsTreeCursorGotoFirstChild:
        printTree(indents + 1, cursor.tsTreeCursorCurrentNode, lineLen, code)
    while cursor.tsTreeCursorGotoNextSibling:
        printTree(indents + 1, cursor.tsTreeCursorCurrentNode, lineLen, code)

when isMainModule:
    var
        parser = tsParserNew()
        lang = treeSitterPython()
    discard parser.tsParserSetLanguage(lang)
    
    var
        code = "max(1,2,3) if a > 1 else min(2,3,4)"
        tree = parser.tsParserParseString(nil, code.cstring, code.len.uint32)
        lineLen : seq[int] = @[]
        
    for line in code.split("\n"):
        lineLen.add(line.len)
    
    printTree(0, tree.tsTreeRootNode, lineLen, code)

    parser.tsParserDelete()
```
#### output
```
"module",[(0, 0),(0, 35)]: "max(1,2,3) if a > 1 else min(2,3,4)"
    "expression_statement",[(0, 0),(0, 35)]: "max(1,2,3) if a > 1 else min(2,3,4)"
        "conditional_expression",[(0, 0),(0, 35)]: "max(1,2,3) if a > 1 else min(2,3,4)"
            "call",[(0, 0),(0, 10)]: "max(1,2,3)"
                "identifier",[(0, 0),(0, 3)]: "max"
                "argument_list",[(0, 3),(0, 10)]: "(1,2,3)"
                    "(",[(0, 3),(0, 4)]: "("
                    "integer",[(0, 4),(0, 5)]: "1"
                    ",",[(0, 5),(0, 6)]: ","
                    "integer",[(0, 6),(0, 7)]: "2"
                    ",",[(0, 7),(0, 8)]: ","
                    "integer",[(0, 8),(0, 9)]: "3"
                    ")",[(0, 9),(0, 10)]: ")"
            "if",[(0, 11),(0, 13)]: "if"
            "comparison_operator",[(0, 14),(0, 19)]: "a > 1"
                "identifier",[(0, 14),(0, 15)]: "a"
                ">",[(0, 16),(0, 17)]: ">"
                "integer",[(0, 18),(0, 19)]: "1"
            "else",[(0, 20),(0, 24)]: "else"
            "call",[(0, 25),(0, 35)]: "min(2,3,4)"
                "identifier",[(0, 25),(0, 28)]: "min"
                "argument_list",[(0, 28),(0, 35)]: "(2,3,4)"
                    "(",[(0, 28),(0, 29)]: "("
                    "integer",[(0, 29),(0, 30)]: "2"
                    ",",[(0, 30),(0, 31)]: ","
                    "integer",[(0, 31),(0, 32)]: "3"
                    ",",[(0, 32),(0, 33)]: ","
                    "integer",[(0, 33),(0, 34)]: "4"
                    ")",[(0, 34),(0, 35)]: ")"
```
