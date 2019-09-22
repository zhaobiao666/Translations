错误处理
==================================

概览
----------------
Hyperledger Fabric 使用 **github.com/pkg/errors** 包来替换 Go 的标准错误类型。这个包可以通过错误信息来简化堆栈追踪的生成和展示。

使用说明
------------------

应该使用 **github.com/pkg/errors** 来替换所有 ``fmt.Errorf()`` 或 ``errors.New()`` 的调用。使用该包可以生成一个调用栈，并把错误信息追加到栈中。

这个包使用简单并且只需要简单的调整你的代码。

首先，你需要导入 **github.com/pkg/errors**。

然后，更新所有使用了错误创建方法（比如，errors.New()、errors.Errorf()、errors.WithMessage()、errors.Wrap()、errors.Wrapf()）的错误。

.. note:: 在 https://godoc.org/github.com/pkg/errors 查看所有支持的错误创建方法。也可以参考下边 Fabric 代码使用指南的章节。

最后，将所有记录器的格式或 fmt.Printf() 的调用的 ``%s`` 改为 ``%+v``，以便通过错误信息来打印调用栈。

Hyperledger Fabric 中错误处理指南
-----------------------------------------------------------

- 如果你为用户提供服务，你应该记录并返回错误。
- 如果错误来自外部，比如 Go 的依赖，使用 errors.Wrap() 来包装错误以便生成该错误的调用堆栈。
- 如果错误来自 Fabric 的其他方法，如果可以的话就使用 errors.WithMessage() 在离开调用堆栈时，在错误信息中增加更多的上下文信息。
- 致命的错误不应该传播到其他包。

示例程序
---------------

下边的示例程序清晰的展示了包的用法：

.. code:: go

  package main

  import (
    "fmt"

    "github.com/pkg/errors"
  )

  func wrapWithStack() error {
    err := createError()
    // do this when error comes from external source (go lib or vendor)
    return errors.Wrap(err, "wrapping an error with stack")
  }
  func wrapWithoutStack() error {
    err := createError()
    // do this when error comes from internal Fabric since it already has stack trace
    return errors.WithMessage(err, "wrapping an error without stack")
  }
  func createError() error {
    return errors.New("original error")
  }

  func main() {
    err := createError()
    fmt.Printf("print error without stack: %s\n\n", err)
    fmt.Printf("print error with stack: %+v\n\n", err)
    err = wrapWithoutStack()
    fmt.Printf("%+v\n\n", err)
    err = wrapWithStack()
    fmt.Printf("%+v\n\n", err)
  }

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
