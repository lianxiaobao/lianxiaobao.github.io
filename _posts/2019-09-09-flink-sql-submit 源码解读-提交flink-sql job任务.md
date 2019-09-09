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
>* [flink-sql-submit 源码解读-提交flink-sql job任务](https://lianxiaobao.github.io/2019/09/09/flink-sql-submit-%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB-%E6%8F%90%E4%BA%A4flink-sql-job%E4%BB%BB%E5%8A%A1/)


# flink-sql-submit 源码解读-提交flink-sql job任务
![](https://tva1.sinaimg.cn/large/006y8mN6ly1g6t8i04idkj318p0u04dc.jpg)

## 项目打包部署执行流程

### 1. `run.sh`
* 任务提交shell脚本
* 在 `flink-sql-submit` 目录下运行`./run.sh q1`

```bash
source "$(dirname "$0")"/env.sh

PROJECT_DIR=`pwd`
$FLINK_DIR/bin/flink run -d -p 4 target/flink-sql-submit.jar -w "${PROJECT_DIR}"/src/main/resources/ -f "$1".sql

```

### 2. `SqlSubmit.java`
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




## 构建`Apache Commons CLI`命令行启动

`Apache Commons CLI`是开源的命令行解析工具，它可以帮助开发者快速构建启动命令，并且帮助你组织命令的参数、以及输出列表等。

>**CLI分为三个过程：**

* 定义阶段：在`Java`代码中定义`Optin`参数，定义参数、是否需要输入值、简单的描述等
* 解析阶段：应用程序传入参数后，`CLI`进行解析
* 询问阶段：通过查询`CommandLine`询问进入到哪个程序分支中



### 1. `CliOptions.java`

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

### 2. `CliOptionsParser.java`



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


##  解析`sql` 调用`TableEnvironment`执行不同的任务


### 1. `SqlCommandParser.java`
>知识点

1. [Java 枚举(enum) 详解7种常见的用法](https://blog.csdn.net/qq_27093465/article/details/52180865)
2. [JDK8新特性-java.util.function-Function接口](https://blog.csdn.net/huo065000/article/details/78964382)


```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.function.Function;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Simple parser for determining the type of command and its parameters.
 */
public final class SqlCommandParser {

	private SqlCommandParser() {
		// private
	}

	public static List<SqlCommandCall> parse(List<String> lines) {
		List<SqlCommandCall> calls = new ArrayList<>();
		StringBuilder stmt = new StringBuilder();
		for (String line : lines) {
			if (line.trim().isEmpty() || line.startsWith("--")) {
				// skip empty line and comment line
				continue;
			}
			stmt.append("\n").append(line);
			if (line.trim().endsWith(";")) {
				Optional<SqlCommandCall> optionalCall = parse(stmt.toString());
				if (optionalCall.isPresent()) {
					calls.add(optionalCall.get());
				} else {
					throw new RuntimeException("Unsupported command '" + stmt.toString() + "'");
				}
				// clear string builder
				stmt.setLength(0);
			}
		}
		return calls;
	}

	public static Optional<SqlCommandCall> parse(String stmt) {
		// normalize
		stmt = stmt.trim();
		// remove ';' at the end
		if (stmt.endsWith(";")) {
			stmt = stmt.substring(0, stmt.length() - 1).trim();
		}

		// parse
		for (SqlCommand cmd : SqlCommand.values()) {
			final Matcher matcher = cmd.pattern.matcher(stmt);
			if (matcher.matches()) {
				final String[] groups = new String[matcher.groupCount()];
				for (int i = 0; i < groups.length; i++) {
					groups[i] = matcher.group(i + 1);
				}
				return cmd.operandConverter.apply(groups)
					.map((operands) -> new SqlCommandCall(cmd, operands));
			}
		}
		return Optional.empty();
	}

	// --------------------------------------------------------------------------------------------

	private static final Function<String[], Optional<String[]>> NO_OPERANDS =
		(operands) -> Optional.of(new String[0]);

	private static final Function<String[], Optional<String[]>> SINGLE_OPERAND =
		(operands) -> Optional.of(new String[]{operands[0]});

	private static final int DEFAULT_PATTERN_FLAGS = Pattern.CASE_INSENSITIVE | Pattern.DOTALL;

	/**
	 * Supported SQL commands.
	 */
	public enum SqlCommand {
		INSERT_INTO(
			"(INSERT\\s+INTO.*)",
			SINGLE_OPERAND),

		CREATE_TABLE(
			"(CREATE\\s+TABLE.*)",
				SINGLE_OPERAND),

		SET(
			"SET(\\s+(\\S+)\\s*=(.*))?", // whitespace is only ignored on the left side of '='
			(operands) -> {
				if (operands.length < 3) {
					return Optional.empty();
				} else if (operands[0] == null) {
					return Optional.of(new String[0]);
				}
				return Optional.of(new String[]{operands[1], operands[2]});
			});

		public final Pattern pattern;
		public final Function<String[], Optional<String[]>> operandConverter;

		SqlCommand(String matchingRegex, Function<String[], Optional<String[]>> operandConverter) {
			this.pattern = Pattern.compile(matchingRegex, DEFAULT_PATTERN_FLAGS);
			this.operandConverter = operandConverter;
		}

		@Override
		public String toString() {
			return super.toString().replace('_', ' ');
		}

		public boolean hasOperands() {
			return operandConverter != NO_OPERANDS;
		}
	}

	/**
	 * Call of SQL command with operands and command type.
	 */
	public static class SqlCommandCall {
		public final SqlCommand command;
		public final String[] operands;

		public SqlCommandCall(SqlCommand command, String[] operands) {
			this.command = command;
			this.operands = operands;
		}

		public SqlCommandCall(SqlCommand command) {
			this(command, new String[0]);
		}

		@Override
		public boolean equals(Object o) {
			if (this == o) {
				return true;
			}
			if (o == null || getClass() != o.getClass()) {
				return false;
			}
			SqlCommandCall that = (SqlCommandCall) o;
			return command == that.command && Arrays.equals(operands, that.operands);
		}

		@Override
		public int hashCode() {
			int result = Objects.hash(command);
			result = 31 * result + Arrays.hashCode(operands);
			return result;
		}

		@Override
		public String toString() {
			return command + "(" + Arrays.toString(operands) + ")";
		}
	}
}

```

### 2. `q1.sql`
* flink sql 脚本任务

```sql
-- -- 开启 mini-batch
-- SET table.exec.mini-batch.enabled=true;
-- -- mini-batch的时间间隔，即作业需要额外忍受的延迟
-- SET table.exec.mini-batch.allow-latency=1s;
-- -- 一个 mini-batch 中允许最多缓存的数据
-- SET table.exec.mini-batch.size=1000;
-- -- 开启 local-global 优化
-- SET table.optimizer.agg-phase-strategy=TWO_PHASE;
--
-- -- 开启 distinct agg 切分
-- SET table.optimizer.distinct-agg.split.enabled=true;


-- source
CREATE TABLE user_log (
    user_id VARCHAR,
    item_id VARCHAR,
    category_id VARCHAR,
    behavior VARCHAR,
    ts TIMESTAMP
) WITH (
    'connector.type' = 'kafka',
    'connector.version' = 'universal',
    'connector.topic' = 'user_behavior',
    'connector.startup-mode' = 'earliest-offset',
    'connector.properties.0.key' = 'zookeeper.connect',
    'connector.properties.0.value' = 'localhost:2181',
    'connector.properties.1.key' = 'bootstrap.servers',
    'connector.properties.1.value' = 'localhost:9092',
    'update-mode' = 'append',
    'format.type' = 'json',
    'format.derive-schema' = 'true'
);

-- sink
CREATE TABLE pvuv_sink (
    dt VARCHAR,
    pv BIGINT,
    uv BIGINT
) WITH (
    'connector.type' = 'jdbc',
    'connector.url' = 'jdbc:mysql://localhost:3306/flink-test',
    'connector.table' = 'pvuv_sink',
    'connector.username' = 'root',
    'connector.password' = 'root',
    'connector.write.flush.max-rows' = '1'
);


INSERT INTO pvuv_sink
SELECT
  DATE_FORMAT(ts, 'yyyy-MM-dd HH:00') dt,
  COUNT(*) AS pv,
  COUNT(DISTINCT user_id) AS uv
FROM user_log
GROUP BY DATE_FORMAT(ts, 'yyyy-MM-dd HH:00');
```