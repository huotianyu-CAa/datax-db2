# datax适配db2数据库upsert功能
# 需求描述：让rdbmswriter支持upsert功能
# 功能设计：由于db2数据库不支持on duplicate key update语句，但是支持merge into 语句，所以修改源码，在接收到update参数的时候拼接merge语句
# 实现步骤：
## 1.为了方便调试，选择本地debug datax项目
### 从git上clone下来datax源码，  
rdbms本身是不支持db2数据库的，所以需要在rdbmswriter/libs中添加db2数据库驱动，并在plugin.json中配置驱动项
```
{
"name": "rdbmswriter",

"class": "com.alibaba.datax.plugin.reader.rdbmswriter.RdbmsWriter",

"description": "useScene: prod. mechanism: Jdbc connection using the database, execute select sql, retrieve data from the ResultSet. warn: The more you know about the database, the less problems you encounter.",

"developer": "alibaba",

"drivers":["dm.jdbc.driver.DmDriver", "com.sybase.jdbc3.jdbc.SybDriver", "com.edb.Driver","com.ibm.db2.jcc.DB2Driver"]

}
```
为了防止程序运行时报错，提前修改{datax.path}/core/src/main/conf/core.json文件中的byte参数，由-1调整成2000000
```
"transport": {
"channel": {
"class": "com.alibaba.datax.core.transport.channel.memory.MemoryChannel",

"speed": {
"byte": 2000000,

"record": -1

},
"flowControlInterval": 20,

"capacity": 512,

"byteCapacity": 67108864

},
"exchanger": {
"class": "com.alibaba.datax.core.plugin.BufferedRecordExchanger",

"bufferSize": 32

}
}
```
通过main函数启动项目需要新增两个参数
```
public static void main(String[] args) throws Exception {
    int exitCode = 0;
    try {
        System.setProperty("datax.home","/Users/huotianyu/Downloads/DataX-master/target/datax/datax");
        args = new String[]{"-mode", "standalone", "-jobid", "-1", "-job", "/Users/huotianyu/Downloads/test.json"};
        Engine.entry(args);
    } catch (Throwable e) {
        exitCode = 1;
        LOG.error("\n\n经DataX智能分析,该任务最可能的错误原因是:\n" + ExceptionTracker.trace(e));


        if (e instanceof DataXException) {
            DataXException tempException = (DataXException) e;
            ErrorCode errorCode = tempException.getErrorCode();
            if (errorCode instanceof FrameworkErrorCode) {
                FrameworkErrorCode tempErrorCode = (FrameworkErrorCode) errorCode;
                exitCode = tempErrorCode.toExitValue();
            }
        }


        System.exit(exitCode);
    }
    System.exit(exitCode);
}
```
其中datax.home对应的绝对路径需要mvn打包之后才可以生成，  
mvn打包命令：  
/Users/huotianyu/developTool/apache-maven-3.6.1/bin/mvn -U clean package assembly:assembly -Dmaven.test.skip=true 

Args中的绝对路径为自定义任务json文件的路径，本次执行任务的json，本次需求就是为了支持根据主键或联合主键进行upsert操作. 
例如：update(ID),update(ID,CODE). 

明确本次修改的模块是plugin-rdbms-util包和rdbmswriter包<br>
模块：rdbmswriter<br>
文件路径：package com.alibaba.datax.plugin.reader.rdbmswriter.RdbmsWriter.java<br>

模块：plugin-rdbms-util<br>
文件路径：package com.alibaba.datax.plugin.rdbms.writer.util.WriterUtil<br>
文件路径：package com.alibaba.datax.plugin.rdbms.writer.CommonRdbmsWriter<br>
