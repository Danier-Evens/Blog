### 背景:

* 使用spring xml配置管理初始化HbaseTemplate，并使用HbaseTemplate API时存在bug，每次查询都会建立connection，这样会存在并发查询的时候大量时间浪费在connection建立上。
* spring-data-hadoop版本为2.2.0.RELEASE，测试对比过 `ConnectionFactory创建并使connection全局共享`和`通过spring管理的方式`，对应如下图1、图2对比，监控的指标分别为： 查询耗时、应用GC、机器与HBase服务网络情况。其中应用GC、机器与HBase服务网络情况都比较良好。查询耗时通过使用spring管理的connection查询非常不稳地。本文后面会从源码进行解析坑所在。同时对比了2.5.0.RELEASE源码都有类似的问题。2.3.0.RELEASE、2.4.0.RELEASE估计也没差~

  ![图1](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/hbase_query01.png)
  
  ![图2](https://raw.githubusercontent.com/Danier-Evens/Markdown_Image/master/image/hbase_query02.png)
 
* 查询方式使用HBase原生api的方式，rowkey范围查询，满足条件的数据条数为29条，数据2KW+。

```
@Autowired
private HbaseTemplate hbaseTemplate;
.......
    
Scan scan = new Scan();
scan.setStartRow(Bytes.toBytes(filterStartRowKey));
scan.setStopRow(Bytes.toBytes(filterEndRowKey));
List<Map<String, Object>> result = hbaseTemplate.find(tableName, scan, (resultSet, i) -> {
	List<Cell> cells = resultSet.listCells();
    Map<String, Object> valueByCloumn = cells.parallelStream()
    	.collect(Collectors.toMap(cell -> Bytes.toString(CellUtil.cloneQualifier(cell)),
                                cell -> Bytes.toString(CellUtil.cloneValue(cell))));

    return valueByCloumn;
});
```  
  
### 分析:

#### spring管理接入步骤

* 引入jar包

```
<dependency>
	<groupId>org.springframework.data</groupId>
    <artifactId>spring-data-hadoop</artifactId>
    <version>2.5.0.RELEASE</version>
    <exclusions>
    	<exclusion>
        	<groupId>org.springframework</groupId>
            <artifactId>spring-context-support</artifactId>
        </exclusion>
        <exclusion>
        	<groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
        <exclusion>
        	<groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </exclusion>
   </exclusions>
</dependency>

<dependency>
	<groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>1.1.2</version>
    	<exclusions>
        	<exclusion>
            	<artifactId>slf4j-log4j12</artifactId>
                <groupId>org.slf4j</groupId>
            </exclusion>
       </exclusions>
</dependency>
```
* xml配置

```
<hdp:hbase-configuration id="hbaseConfigurationConf"
                         zk-quorum="xxxx"
                         zk-port="xxx"/>
<bean id="hTableFactory" class="org.apache.hadoop.hbase.client.HTableFactory"/>
<bean id="hbaseTemplate" class="org.springframework.data.hadoop.hbase.HbaseTemplate" lazy-init="true">
    <property name="configuration" ref="hbaseConfigurationConf"/>
    <property name="tableFactory" ref="hTableFactory"/>
</bean>
```
* 跟踪代码 hbaseTemplate.find(....)，最终执行的方法为HbaseTemplate的execute(...)方法，获取connection的关键代码在`HTableInterface table = this.getTable(tableName);`，两个奇葩点既引起bug的原因如下：</br>
	
	*1、首先会判断传入的table是否已存在HTableInterface对象，通过HbaseSynchronizationManager类管理着这些全局的HTableInterface按照tableName分组，但是发现如果需要HbaseSynchronizationManager管理，需要调用他的bindResource方法，同时这个方法调用需要配置AOP，类似spring管理mysql事物配置，拦截器类为HbaseInterceptor，此类的作用为自动将HBase表绑定给当前线程，也就是说，每个在HBase上执行的DAO操作的类都会被HbaseInterceptor包装，因此一旦发现有在使用的表都将被绑定给当前线程，之后再使用这张表时就无需再初始化表了（理论上达到了HBase连接池的效果），调用结束后，表将被自己关闭。但是拦截器中的初始化table的时候，调用的方式还是this.getTable(tableName)逻辑一样。*
	
	*2、进入下一步，有判断`tableFactory`是否为空，此对象为spring的xml注入的实例，此处见如上的配置始终是不为空，最终会调用`createHTableInterface`创建`HTableInterface`,继续跟踪其源码发现跟不注入`tableFactory`对应是一样的，始终会new HTable(....)，HTable初始化的时候会创建`connection`，当查询完毕之后会触发`table.close`，也就将`connection`关闭了，下次在进入查询，会创建新的`connection`。*
	
	*3、结合以上两点，无论是AOP方式管理连接池，还是初始化HbaseTemplate的方式，都会触发new Ttable(.....)，当table.close()时，Table对像中的Connection都会被关闭。即spring data管理的HBase连接池并没达到想要的效果。*


```
关键代码：

> 判断hasResource 有返回没有反之获取新的HTable实例

public static HTableInterface getHTable(String tableName, Configuration configuration, Charset charset, HTableInterfaceFactory tableFactory) {
        if(HbaseSynchronizationManager.hasResource(tableName)) {
            return (HTable)HbaseSynchronizationManager.getResource(tableName);
        } else {
            Object t = null;

            try {
                if(tableFactory != null) {
                    t = tableFactory.createHTableInterface(configuration, tableName.getBytes(charset));
                } else {
                    t = new HTable(configuration, tableName.getBytes(charset));
                }

                return (HTableInterface)t;
            } catch (Exception var6) {
                throw convertHbaseException(var6);
            }
        }
}

---------------------------------------------------------------------------------------------------------
> HTableFactory.createHTableInterface

public class HTableFactory implements HTableInterfaceFactory {
    public HTableFactory() {
    }

    public HTableInterface createHTableInterface(Configuration config, byte[] tableName) {
        try {
            return new HTable(config, TableName.valueOf(tableName));
        } catch (IOException var4) {
            throw new RuntimeException(var4);
        }
    }

    public void releaseHTableInterface(HTableInterface table) throws IOException {
        table.close();
    }
}

---------------------------------------------------------------------------------------------------------
> new HTable(...)
public HTable(Configuration conf, TableName tableName) throws IOException {
        this.autoFlush = true;
        this.closed = false;
        this.defaultConsistency = Consistency.STRONG;
        this.tableName = tableName;
        this.cleanupPoolOnClose = this.cleanupConnectionOnClose = true;
        if(conf == null) {
            this.connection = null;
        } else {
            this.connection = ConnectionManager.getConnectionInternal(conf);
            this.configuration = conf;
            this.pool = getDefaultExecutor(conf);
            this.finishSetup();
        }
    }

--------------------------------------------------------------------------------------------------------
> table.close
public void close() throws IOException {
        if(!this.closed) {
            
            .....

            if(this.cleanupConnectionOnClose && this.connection != null) {
                this.connection.close();
            }

            this.closed = true;
        }
    }

```

### 后续:

* 通过以上代码分析，spring-data管理HbaseTemplate有坑，解决办法：</br>
	*1、自己实现一个tableFactory，注入进去，功能为实现管理全局的connection保证不重复创建connection。*
	
	*2、简单的直接使用ConnectionFactory方式创建，同时也测试对比过，速度比较键定，见上面截图*
	
```
public class HbaseUtils {
    private static final Logger logger = LoggerFactory.getLogger(HbaseUtils.class);
    private static Connection connection;

    static {
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "xxxxxx");
        configuration.set("zookeeper.znode.parent", "/hbase");
        try {
            connection = ConnectionFactory.createConnection(configuration);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 原生api scan操作
     *
     * @param tableName 表名
     * @param scan      scan对象
     * @return
     */
    public static List<Map<String, String>> scan(String tableName, Scan scan) {
        List<Map<String, String>> scanResult = Lists.newArrayList();
        if (connection == null) {
            logger.warn("scan error the connection is null");
            return scanResult;
        }

        Table table = null;
        ResultScanner rs = null;
        try {
            table = connection.getTable(TableName.valueOf(tableName));
            rs = table.getScanner(scan);
            Map<String, String> cellValueItem;
            for (Result r : rs) {
                cellValueItem = r.listCells()
                        .parallelStream()
                        .collect(Collectors.toMap(key -> Bytes.toString(CellUtil.cloneQualifier(key)),
                                value -> Bytes.toString(CellUtil.cloneValue(value))));
                scanResult.add(cellValueItem);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (rs != null) {
                rs.close();
            }
            if (table != null) {
                try {
                    table.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            return scanResult;
        }
    }
}
```


