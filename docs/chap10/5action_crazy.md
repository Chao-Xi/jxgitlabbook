
GitHub Actions 是GitHub的一个组件，可让您创建自动化工作流程。通过许多可以触发 工作流程的不同事件，您可以自由地构建您想要的任何自动化。

虽然最常见的用例是构建 CI/CD 管道，但可能性几乎是无限的。查看这份[awesome actions](https://github.com/sdras/awesome-actions) 以获得一些灵感

## If statements

乍一看文档，似乎不支持表达式中的if 语句。但是，如果语句确实存在。看看下面的工作流程

```
name: If

on: push

jobs:
  if:
    name: If
    runs-on: ubuntu-20.04
    strategy:
      matrix:
        job: [a, b]
    steps:
    - run: echo ${{ (matrix.job == 'a' && 'foo') || 'bar' }}
```

Job a将输出foo，而Job b 将输出 bar。您可以将表达式解读为以下 if 语句(pseudo code):

```
if (matrix.job == 'a') {
  echo 'foo'
} else {
  echo 'bar'
}
```

当与可用的[函数](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#functions)或[运算符](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#defining-environment-variables-for-a-single-workflow)之一结合使用时，它的功能会 更加强大。例如，您可以检查数字的值

```
name: If

on: push

jobs:
  if:
    name: If
    runs-on: ubuntu-20.04
    strategy:
      matrix:
        node: [4, 6, 8, 10, 12]
    steps:
    - run: echo ${{ (matrix.node > 8 && 'foo') || 'bar' }}
```

总结一下

只有当两个参数都为 true 时，AND 运算符才会返回true。 

如果第一个参数Not true，则无需评估第二个参数，因为它是否为真并不重要。OR 运算符的工作原理非常相似。**由于如果至少一个参数为 true 则返回 true,** 因此如果第一个参数已经为 true，则无需计算第二个参数。

因此，这种行为会"短路"对布尔运算符传递的参数的求值。这两个运算符的组合为我们提供了 GitHub Actions中的 if 功能


## Composing secret names
