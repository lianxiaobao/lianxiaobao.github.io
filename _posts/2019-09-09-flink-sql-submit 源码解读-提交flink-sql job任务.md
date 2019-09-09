---
layout:     post
title:      apache flink
subtitle:   flink 1.9:Apache flink 使用 SQL 读取 Kafka 并写入 MySQL（源码解读-提交flink-sql job任务）
date:       2019-09-09
author:     BY xiaobao(微信：Bao1697047283)
header-img: img/post-bg-mi1.jpg
catalog: true
tags:
    - blog
    - 数据开发
    - kafka
    - flink
    
---


#### 目录
>* [使用 SQL 读取 Kafka 并写入 MySQL](https://lianxiaobao.github.io/2019/09/06/Apache-flink%E4%BD%BF%E7%94%A8SQL%E8%AF%BB%E5%8F%96Kafka%E5%B9%B6%E5%86%99%E5%85%A5MySQL/)
>* [flink-sql-submit 源码解读-模拟kafka源](https://lianxiaobao.github.io/2019/09/09/flink-sql-submit-%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB-%E6%A8%A1%E6%8B%9Fkafka%E6%BA%90/)
>* [flink-sql-submit 源码解读-提交flink-sql job任务]()


# flink-sql-submit 源码解读-提交flink-sql job任务
![](https://tva1.sinaimg.cn/large/006y8mN6ly1g6t8i04idkj318p0u04dc.jpg)

## `SqlSubmit.java`
* 任务启动、提交类

```java
import com.github.wuchong.sqlsubmit.cli.CliOptions;
import com.github.wuchong.sqlsubmit.cli.CliOptionsParser;
import com.github.wuchong.sqlsubmit.cli.SqlCommandParser;
import com.github.wuchong.sqlsubmit.cli.SqlCommandParser.SqlCommandCall;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.SqlParserException;
import org.apache.flink.table.api.TableEnvironment;

import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;

public class SqlSubmit {

    public static void main(String[] args) throws Exception {
    	// 构建自定义的启动命令
        final CliOptions options = CliOptionsParser.parseClient(args);
        SqlSubmit submit = new SqlSubmit(options);
        submit.run();
    }

    // --------------------------------------------------------------------------------------------

    private String sqlFilePath;
    private String workSpace;
    private TableEnvironment tEnv;

    private SqlSubmit(CliOptions options) {
        this.sqlFilePath = options.getSqlFilePath();
        this.workSpace = options.getWorkingSpace();
    }

    private void run() throws Exception {
        // 创建一个使用 Blink Planner 的 TableEnvironment, 并工作在流模式
        EnvironmentSettings settings = EnvironmentSettings.newInstance()
                .useBlinkPlanner()
                .inStreamingMode()
                .build();
        this.tEnv = TableEnvironment.create(settings);
        // 读取 SQL 文件
        List<String> sql = Files.readAllLines(Paths.get(workSpace + "/" + sqlFilePath));
        // 通过正则表达式匹配前缀，来区分不同的 SQL 语句
        List<SqlCommandCall> calls = SqlCommandParser.parse(sql);
        for (SqlCommandCall call : calls) {
        	// 根据不同的 SQL 语句，调用 TableEnvironment 执行
            callCommand(call);
        }
        tEnv.execute("SQL Job");
    }

    // --------------------------------------------------------------------------------------------

    private void callCommand(SqlCommandCall cmdCall) {
        switch (cmdCall.command) {
            case SET:
                callSet(cmdCall);
                break;
            case CREATE_TABLE:
                callCreateTable(cmdCall);
                break;
            case INSERT_INTO:
                callInsertInto(cmdCall);
                break;
            default:
                throw new RuntimeException("Unsupported command: " + cmdCall.command);
        }
    }

    private void callSet(SqlCommandCall cmdCall) {
        String key = cmdCall.operands[0];
        String value = cmdCall.operands[1];
        tEnv.getConfig().getConfiguration().setString(key, value);
    }

    private void callCreateTable(SqlCommandCall cmdCall) {
        String ddl = cmdCall.operands[0];
        try {
            tEnv.sqlUpdate(ddl);
        } catch (SqlParserException e) {
            throw new RuntimeException("SQL parse failed:\n" + ddl + "\n", e);
        }
    }

    private void callInsertInto(SqlCommandCall cmdCall) {
        String dml = cmdCall.operands[0];
        try {
            tEnv.sqlUpdate(dml);
        } catch (SqlParserException e) {
            throw new RuntimeException("SQL parse failed:\n" + dml + "\n", e);
        }
    }
}

```
>1. SqlSubmit.run; 
 * 获取`run.sh`脚本中的命令参数

>2. submit.run();
 * 解析`sql`脚本提交任务
>


## `run.sh`
* 任务提交shell脚本
* 在 `flink-sql-submit` 目录下运行`./run.sh q1`

```bash
source "$(dirname "$0")"/env.sh

PROJECT_DIR=`pwd`
$FLINK_DIR/bin/flink run -d -p 4 target/flink-sql-submit.jar -w "${PROJECT_DIR}"/src/main/resources/ -f "$1".sql

```

## `CliOptionsParser.java`、`CliOptions.java`

**构建`Apache Commons CLI`命令行启动**

`Apache Commons CLI`是开源的命令行解析工具，它可以帮助开发者快速构建启动命令，并且帮助你组织命令的参数、以及输出列表等。

>**CLI分为三个过程：**

* 定义阶段：在`Java`代码中定义`Optin`参数，定义参数、是否需要输入值、简单的描述等
* 解析阶段：应用程序传入参数后，`CLI`进行解析
* 询问阶段：通过查询`CommandLine`询问进入到哪个程序分支中



### `CliOptions.java`

```java
/**
 * Command line options to configure the SQL client. Arguments that have not been specified by the user are null.
 */
public class CliOptions {

    private final String sqlFilePath;
    private final String workingSpace;

    public CliOptions(String sqlFilePath, String workingSpace) {
        this.sqlFilePath = sqlFilePath;
        this.workingSpace = workingSpace;
    }

    public String getSqlFilePath() {
        return sqlFilePath;
    }

    public String getWorkingSpace() {
        return workingSpace;
    }
}


```

### `CliOptionsParser.java`



```java
import org.apache.commons.cli.CommandLine;
import org.apache.commons.cli.DefaultParser;
import org.apache.commons.cli.Option;
import org.apache.commons.cli.Options;
import org.apache.commons.cli.ParseException;

public class CliOptionsParser {
    //定义 Optin 参数，定义参数、是否需要输入值、简单的描述等
    public static final Option OPTION_WORKING_SPACE = Option
            .builder("w")
            .required(true)
            .longOpt("working_space")
            .numberOfArgs(1)
            .argName("working space dir")
            .desc("The working space dir.")
            .build();

    public static final Option OPTION_SQL_FILE = Option
            .builder("f")
            .required(true)
            .longOpt("file")
            .numberOfArgs(1)
            .argName("SQL file path")
            .desc("The SQL file path.")
            .build();

    public static final Options CLIENT_OPTIONS = getClientOptions(new Options());

    public static Options getClientOptions(Options options) {
        options.addOption(OPTION_SQL_FILE);
        options.addOption(OPTION_WORKING_SPACE);
        return options;
    }

    // --------------------------------------------------------------------------------------------
    //  Line Parsing
    // --------------------------------------------------------------------------------------------
    
    
	//将 -w -f	参数添加到 CliOptions
    public static CliOptions parseClient(String[] args) {
        if (args.length < 1) {
            throw new RuntimeException("./sql-submit -w <work_space_dir> -f <sql-file>");
        }
        try {
            DefaultParser parser = new DefaultParser();
            CommandLine line = parser.parse(CLIENT_OPTIONS, args, true);
            return new CliOptions(
                    line.getOptionValue(CliOptionsParser.OPTION_SQL_FILE.getOpt()),
                    line.getOptionValue(CliOptionsParser.OPTION_WORKING_SPACE.getOpt())
            );
        }
        catch (ParseException e) {
            throw new RuntimeException(e.getMessage());
        }
    }
}

```
