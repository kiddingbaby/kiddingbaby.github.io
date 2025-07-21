---
title: "理解 Jenkins Pipeline 中的序列化与暂停恢复逻辑"
date: 2025-06-17T10:00:00+08:00
lastmod: 2025-06-17T10:00:00+08:00
categories: ["DevOps", "CI/CD"]
tags: ["Jenkins", "Pipeline", "Groovy", "序列化", "持续集成"]
series: ["DevOps"]
keywords: ["Jenkins Pipeline", "Groovy CPS", "流水线序列化", "脚本恢复", "bindScript"]
ShowToc: true
TocOpen: true
draft: false
---

## Groovy CPS 简介

Groovy CPS 中的 CPS 全称是 Continuation Passing Style，翻译过来就是**延续传递风格**。它允许流水线脚本在任意位置暂停执行，并保存当前执行状态，待外部事件（如审批、异步触发）完成后，再恢复执行。这种机制使得 Jenkins 能够支持复杂的流水线控制和分布式执行。

## Jenkins Pipeline 执行模型

Jenkins pipeline 通过 Groovy CPS 实现流水线的暂停与恢复能力。

- 暂停执行：当遇到等待节点（如输入审批、异步事件）时暂停执行，脚本状态序列化保存到本地。
- 恢复执行：Jenkins 反序列化脚本状态，并从中断点继续运行。

## Groovy CPS 的限制

- 代码必须是 CPS 可转换的：比如部分 toString() 标准库方法不兼容，I/O、线程、闭包嵌套、反射等特性都不受支持。
- 不能直接持有复杂的非序列化对象：比如 pipeline 脚本上下文对象 Script，想要持有必须用 transient 标记，并实现恢复方法。
- 调试和异常栈会比较复杂：pipeline 中的 Groovy 代码，需通过 Groovy CPS 编译器转成 CPS 代码，而非普通顺序代码，导致异常栈里可能会被 CPS 重写。

## @NonCPS 方法限制

@NoeCPS 会告诉 CPS 不去转换某个方法。但它的使用也有明确限制：

- 方法不能访问 script（会序列化失败或运行时报错）。
- 方法必须是纯逻辑，不能依赖任何 Pipeline DSL，比如 sh(), echo(), input(), checkout() 等。
- 方法返回值必须是基本类型或可序列化的结构（不能返回闭包、不可序列化对象）。

## 使用 `transient` 修饰与动态绑定

想要避免 Jenkins Pipeline 在暂停恢复时因上下文丢失导致的空指针异常和流水线失败，可以使用 `transient` 修饰与动态绑定方案，提高共享库代码的健壮性和用户体验。

下面是一个自定义 Logger 的实践示例：

```groovy
class Logger implements Serializable {
    private transient def script

    Logger(def script) {
        this.script = script
    }

    Logger bindScript(def script) {
        this.script = script
        return this
    }

    void info(String msg) {
        script.echo "[INFO] ${msg}"
    }
}

class ContextResolver implements Serializable {
    private transient def script
    private Logger log

    ContextResolver(def script) {
        bindScript(script)
    }

    void bindScript(def script) {
        this.script = script
        if (log != null) {
            log.bindScript(script)
        } else {
            this.log = new Logger(script)
        }
    }
}
```

### 实践注意

持有 script 对象的逻辑，尽量放在 `vars/`。`src/` 只写纯逻辑代码，避免持有 script 对象。尽量避免 `@NonCPS`，保证流水线的可序列化和稳定性。如果需要持有 script 对象，应注意以下事项：

- 必须为不可序列化的字段添加 transient 修饰，避免序列化异常。
- 持有 script，应实现 bindScript() 进行动态绑定。
- 流水线恢复后，调用 bindScript(this) 确保上下文可用。

