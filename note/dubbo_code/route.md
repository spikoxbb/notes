## 路由

条件路由规则由两个条件组成，分别用于对服务消费者和提供者进行匹配。比如有这样一条规则：

```
host = 10.20.153.10 => host = 10.20.153.11
```

如果服务消费者匹配条件为空，表示不对服务消费者进行限制。如果服务提供者匹配条件为空，表示对某些服务消费者禁用服务。条件路由实现类 ConditionRouter 在进行工作前，需要先对用户配置的路由规则进行解析，得到一系列的条件。然后再根据这些条件对服务进行路由。

 ConditionRouter 的构造方法：

```java
public ConditionRouter(URL url) {
    this.url = url;
    // 获取 priority 和 force 配置
    this.priority = url.getParameter(Constants.PRIORITY_KEY, 0);
    this.force = url.getParameter(Constants.FORCE_KEY, false);
    try {
        // 获取路由规则
        String rule = url.getParameterAndDecoded(Constants.RULE_KEY);
        if (rule == null || rule.trim().length() == 0) {
            throw new IllegalArgumentException("Illegal route rule!");
        }
        rule = rule.replace("consumer.", "").replace("provider.", "");
        // 定位 => 分隔符
        int i = rule.indexOf("=>");
        // 分别获取服务消费者和提供者匹配规则
        String whenRule = i < 0 ? null : rule.substring(0, i).trim();
        String thenRule = i < 0 ? rule.trim() : rule.substring(i + 2).trim();
        // 解析服务消费者匹配规则
        Map<String, MatchPair> when = 
            StringUtils.isBlank(whenRule) || "true".equals(whenRule) 
                ? new HashMap<String, MatchPair>() : parseRule(whenRule);
        // 解析服务提供者匹配规则
        Map<String, MatchPair> then = 
            StringUtils.isBlank(thenRule) || "false".equals(thenRule) 
                ? null : parseRule(thenRule);
        // 将解析出的匹配规则分别赋值给 whenCondition 和 thenCondition 成员变量
        this.whenCondition = when;
        this.thenCondition = then;
    } catch (ParseException e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

```java
private static final class MatchPair {
    final Set<String> matches = new HashSet<String>();
    final Set<String> mismatches = new HashSet<String>();
}
```

MatchPair 内部包含了两个 Set 类型的成员变量，分别用于存放匹配和不匹配的条件。

```java
private static Map<String, MatchPair> parseRule(String rule)
        throws ParseException {
    // 定义条件映射集合
    Map<String, MatchPair> condition = new HashMap<String, MatchPair>();
    if (StringUtils.isBlank(rule)) {
        return condition;
    }
    MatchPair pair = null;
    Set<String> values = null;
    // 通过正则表达式匹配路由规则，ROUTE_PATTERN = ([&!=,]*)\s*([^&!=,\s]+)
    // 这个表达式看起来不是很好理解，第一个括号内的表达式用于匹配"&", "!", "=" 和 "," 等符号。
    // 第二括号内的用于匹配英文字母，数字等字符。举个例子说明一下：
    //    host = 2.2.2.2 & host != 1.1.1.1 & method = hello
    // 匹配结果如下：
    //     括号一      括号二
    // 1.  null       host
    // 2.   =         2.2.2.2
    // 3.   &         host
    // 4.   !=        1.1.1.1 
    // 5.   &         method
    // 6.   =         hello
    final Matcher matcher = ROUTE_PATTERN.matcher(rule);
    while (matcher.find()) {
       	// 获取括号一内的匹配结果
        String separator = matcher.group(1);
        // 获取括号二内的匹配结果
        String content = matcher.group(2);
        // 分隔符为空，表示匹配的是表达式的开始部分
        if (separator == null || separator.length() == 0) {
            // 创建 MatchPair 对象
            pair = new MatchPair();
            // 存储 <匹配项, MatchPair> 键值对，比如 <host, MatchPair>
            condition.put(content, pair); 
        } 
        
        // 如果分隔符为 &，表明接下来也是一个条件
        else if ("&".equals(separator)) {
            // 尝试从 condition 获取 MatchPair
            if (condition.get(content) == null) {
                // 未获取到 MatchPair，重新创建一个，并放入 condition 中
                pair = new MatchPair();
                condition.put(content, pair);
            } else {
                pair = condition.get(content);
            }
        } 
        
        // 分隔符为 =
        else if ("=".equals(separator)) {
            if (pair == null)
                throw new ParseException("Illegal route rule ...");

            values = pair.matches;
            // 将 content 存入到 MatchPair 的 matches 集合中
            values.add(content);
        } 
        
        //  分隔符为 != 
        else if ("!=".equals(separator)) {
            if (pair == null)
                throw new ParseException("Illegal route rule ...");

            values = pair.mismatches;
            // 将 content 存入到 MatchPair 的 mismatches 集合中
            values.add(content);
        }
        
        // 分隔符为 ,
        else if (",".equals(separator)) {
            if (values == null || values.isEmpty())
                throw new ParseException("Illegal route rule ...");
            // 将 content 存入到上一步获取到的 values 中，可能是 matches，也可能是 mismatches
            values.add(content);
        } else {
            throw new ParseException("Illegal route rule ...");
        }
    }
    return condition;
}
```

以上就是路由规则的解析逻辑，该逻辑由正则表达式和一个 while 循环以及数个条件分支组成。下面通过一个示例对解析逻辑进行演绎。示例为 `host = 2.2.2.2 & host != 1.1.1.1 & method = hello`。正则解析结果如下：

```
    括号一      括号二
1.  null       host
2.   =         2.2.2.2
3.   &         host
4.   !=        1.1.1.1
5.   &         method
6.   =         hello
```

现在线程进入 while 循环：

第一次循环：分隔符 separator = null，content = "host"。此时创建 MatchPair 对象，并存入到 condition 中，condition = {"host": MatchPair@123}

第二次循环：分隔符 separator = "="，content = "2.2.2.2"，pair = MatchPair@123。此时将 2.2.2.2 放入到 MatchPair@123 对象的 matches 集合中。

第三次循环：分隔符 separator = "&"，content = "host"。host 已存在于 condition 中，因此 pair = MatchPair@123。

第四次循环：分隔符 separator = "!="，content = "1.1.1.1"，pair = MatchPair@123。此时将 1.1.1.1 放入到 MatchPair@123 对象的 mismatches 集合中。

第五次循环：分隔符 separator = "&"，content = "method"。condition.get("method") = null，因此新建一个 MatchPair 对象，并放入到 condition 中。此时 condition = {"host": MatchPair@123, "method": MatchPair@ 456}

第六次循环：分隔符 separator = "="，content = "2.2.2.2"，pair = MatchPair@456。此时将 hello 放入到 MatchPair@456 对象的 matches 集合中。

循环结束，此时 condition 的内容如下：

```json
{
    "host": {
        "matches": ["2.2.2.2"],
        "mismatches": ["1.1.1.1"]
    },
    "method": {
        "matches": ["hello"],
        "mismatches": []
    }
}
```

服务路由的入口方法是 ConditionRouter 的 router 方法，该方法定义在 Router 接口中。实现代码如下：

```java
public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
    if (invokers == null || invokers.isEmpty()) {
        return invokers;
    }
    try {
        // 先对服务消费者条件进行匹配，如果匹配失败，表明服务消费者 url 不符合匹配规则，
        // 无需进行后续匹配，直接返回 Invoker 列表即可。比如下面的规则：
        //     host = 10.20.153.10 => host = 10.0.0.10
        // 这条路由规则希望 IP 为 10.20.153.10 的服务消费者调用 IP 为 10.0.0.10 机器上的服务。
        // 当消费者 ip 为 10.20.153.11 时，matchWhen 返回 false，表明当前这条路由规则不适用于
        // 当前的服务消费者，此时无需再进行后续匹配，直接返回即可。
        if (!matchWhen(url, invocation)) {
            return invokers;
        }
        List<Invoker<T>> result = new ArrayList<Invoker<T>>();
        // 服务提供者匹配条件未配置，表明对指定的服务消费者禁用服务，也就是服务消费者在黑名单中
        if (thenCondition == null) {
            logger.warn("The current consumer in the service blacklist...");
            return result;
        }
        // 这里可以简单的把 Invoker 理解为服务提供者，现在使用服务提供者匹配规则对 
        // Invoker 列表进行匹配
        for (Invoker<T> invoker : invokers) {
            // 若匹配成功，表明当前 Invoker 符合服务提供者匹配规则。
            // 此时将 Invoker 添加到 result 列表中
            if (matchThen(invoker.getUrl(), url)) {
                result.add(invoker);
            }
        }
        
        // 返回匹配结果，如果 result 为空列表，且 force = true，表示强制返回空列表，
        // 否则路由结果为空的路由规则将自动失效
        if (!result.isEmpty()) {
            return result;
        } else if (force) {
            logger.warn("The route result is empty and force execute ...");
            return result;
        }
    } catch (Throwable t) {
        logger.error("Failed to execute condition router rule: ...");
    }
    
    // 原样返回，此时 force = false，表示该条路由规则失效
    return invokers;
}
```

router 方法先是调用 matchWhen 对服务消费者进行匹配，如果匹配失败，直接返回 Invoker 列表。如果匹配成功，再对服务提供者进行匹配，匹配逻辑封装在了 matchThen 方法中。下面来看一下这两个方法的逻辑：

```java
boolean matchWhen(URL url, Invocation invocation) {
    // 服务消费者条件为 null 或空，均返回 true，比如：
    //     => host != 172.22.3.91
    // 表示所有的服务消费者都不得调用 IP 为 172.22.3.91 的机器上的服务
    return whenCondition == null || whenCondition.isEmpty() 
        || matchCondition(whenCondition, url, null, invocation);  // 进行条件匹配
}

private boolean matchThen(URL url, URL param) {
    // 服务提供者条件为 null 或空，表示禁用服务
    return !(thenCondition == null || thenCondition.isEmpty()) 
        && matchCondition(thenCondition, url, param, null);  // 进行条件匹配
}
```

－－未完－－