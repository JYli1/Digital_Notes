
```java

GroovyExec
LoadJsExec
ProcessImplExec
ProcessBuilderExec
RuntimeExec
```

```java


// 1.### . Runtime.exec() 命令执行
// 危险示例：直接拼接用户输入
public void runtimeExec(String userInput) throws IOException {
    // 假设用户输入为 "id; rm -rf /*" 
    String command = "ping " + userInput;
    Runtime.getRuntime().exec(command);
}


```
```

```java

//2.### ProcessBuilder命令执行
// 危险示例：让用户控制命令或参数
public void processBuilderExec(String userParam) throws IOException {
    // userParam 可能是 "127.0.0.1; id"
    new ProcessBuilder("ping", "-c", "1", userParam).start();
}


//3.### ProcessImpl命令执行（反射调用）
// 通过反射调用 ProcessImpl.start() 执行命令
public void processImplReflect(String cmd) throws Exception {
    Class<?> clazz = Class.forName("java.lang.ProcessImpl");
    // 第一个参数是命令数组，第二个参数是环境变量（可为null），第三个参数是工作目录（可为null）
    Method startMethod = clazz.getDeclaredMethod("start", String[].class, java.util.Map.class, String.class, ProcessBuilder.Redirect[].class, boolean.class);
    startMethod.setAccessible(true);
    // 执行 id 命令
    startMethod.invoke(null, new String[]{"/bin/sh", "-c", cmd}, null, null, null, false);
}

//4。### Groovy 脚本引擎执行
import groovy.lang.GroovyShell;

// 危险示例：直接执行用户提供的 Groovy 代码
public void groovyExec(String userGroovyCode) {
    GroovyShell shell = new GroovyShell();
    // userGroovyCode 可以是 "Runtime.getRuntime().exec('calc')"
    shell.evaluate(userGroovyCode);
}


//5.###  LoadJsExec (使用 ScriptEngine 加载 JS)
import javax.script.ScriptEngine;
import javax.script.ScriptEngineManager;

public void loadJsExec(String userJsCode) throws Exception {
    ScriptEngineManager factory = new ScriptEngineManager();
    ScriptEngine engine = factory.getEngineByName("JavaScript");
    // userJsCode 可以是: var a = java.lang.Runtime.getRuntime().exec("calc");
    engine.eval(userJsCode);
}
```