## 什么是 webpack

webpack 是一个用于现代 JavaScript 应用程序的静态模块打包工具。
当 webpack 处理应用程序时，它会在内部从一个或多个入口点构建一个依赖图(dependency graph)。
然后将你项目中所需的每一个模块组合成一个或多个 bundles，它们均为静态资源，用于展示你的内容。

## 文件转换

```javascript
const fs = require('fs')
const path = require('path')
const parser = require('@babel/parser')
const traverse = require('@babel/traverse').default
const babel = require('@babel/core')

const read = (filename) => {
  const content = fs.readFileSync(filename, 'utf-8')
  const ast = parser.parse(content, {
    sourceType: 'module',
  })
  const dependencies = {}
  traverse(ast, {
    ImportDeclaration({ node }) {
      const dirname = path.dirname(filename)
      const newFile = './' + path.join(dirname, node.source.value)
      dependencies[node.source.value] = newFile
    },
  })
  const { code } = babel.transformFromAst(ast, null, {
    presets: ['@babel/preset-env'],
  })
  return {
    filename,
    dependencies,
    code,
  }
}
```

## 获取依赖图谱

```javascript
const makeDependenciesGraph = (entry) => {
  const entryModule = read(entry)
  const graphArray = [entryModule]
  for (let i = 0; i < graphArray.length; i++) {
    const item = graphArray[i]
    const { dependencies } = item
    if (dependencies) {
      for (let j in dependencies) {
        graphArray.push(read(dependencies[j]))
      }
    }
  }
  const graph = {}
  graphArray.forEach((item) => {
    graph[item.filename] = {
      dependencies: item.dependencies,
      code: item.code,
    }
  })
  return graph
}
```

## 生成代码

```javascript
const generateCode = (entry) => {
  const graph = JSON.stringify(makeDependenciesGraph(entry))
  return `
    (function(graph) {
      function require(module) {
        function localRequire(relativePath) {
          return require(graph[module].dependencies[relativePath])
        }
        const exports = {}
        (function(require, exports, code) {
          eval(code)
        })(localRequire, exports, graph[module].code)
      }
      require('${ entry }')
    }(${ graph }))
  `
}
```
