# flume拦截器-1.9.0
* 其他版本只需要修改pom中的版本一样
## 1. 添加Pom依赖
```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.flume</groupId>
            <artifactId>flume-ng-sdk</artifactId>
            <version>1.9.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flume</groupId>
            <artifactId>flume-ng-core</artifactId>
            <version>1.9.0</version>
        </dependency>
    </dependencies>
```
## 2. 编写拦截器类，实现`org.apache.flume.interceptor.Interceptor`接口(这里已处理kafka source数据为例）
```java
package com.owp.flumeinterceptor;

import com.google.gson.Gson;
import org.apache.commons.codec.Charsets;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.*;

public class MyInterceptor implements Interceptor {
    //打印日志，便于测试方法的执行顺序
    private static final Logger logger = LoggerFactory.getLogger(MyInterceptor.class);

    //自定义拦截器参数，用来接收自定义拦截器flume配置参数
    private static String param = "";

    @Override
    public void initialize() {
        logger.info("----------自定义拦截器initialize方法执行");
    }

    @Override
    public Event intercept(Event event) {
        logger.info("----------intercept(Event event)方法执行");
        if (event == null) return null;
        // 通过获取event数据，转化成字符串
        String line = new String(event.getBody(), Charsets.UTF_8);
        logger.info("Kafka数据，header：【{}】，数据：【{}】", event.getHeaders(), line);
        Gson gson = new Gson();
        //解析kafka读取到的数据
        LinkedHashMap<String, Object> lineInfo = gson.fromJson(line, LinkedHashMap.class);
        //处理后的新数据
        Map<String, String> cdJsonMap = new LinkedHashMap();
        //TODO 添加处理逻辑
        String kafkaData = gson.toJson(cdJsonMap);
        //重新设置event中的body（当然这里也可以设置even其他属性，例如header）
        event.setBody(kafkaData.getBytes(Charsets.UTF_8));
        return event;
    }
    @Override
    public List<Event> intercept(List<Event> list) {
        logger.info("----------intercept(List<Event> events)方法执行");
        if (list == null) return null;
        List<Event> events = new ArrayList<Event>();
        for (Event event : list) {
            Event intercept = intercept(event);
            events.add(intercept);
        }
        return events;
    }
    @Override
    public void close() {
        logger.info("----------自定义拦截器close方法执行");
    }

    /**
     * 通过该静态内部类来创建自定义对象供flume使用，实现Interceptor.Builder接口，并实现其抽象方法
     */
    public static class Builder implements Interceptor.Builder {
        /**
         * 该方法主要用来返回创建的自定义类拦截器对象
         *
         * @return
         */
        @Override
        public Interceptor build() {
            logger.info("----------build方法执行");
            return new MyInterceptor();

        }

        /**
         * 用来接收flume配置自定义拦截器参数
         *
         * @param context 通过该对象可以获取flume配置自定义拦截器的参数
         */
        @Override
        public void configure(Context context) {
            logger.info("----------configure方法执行");
            /*
            通过调用context对象的getString方法来获取flume配置自定义拦截器的参数，方法参数要和自定义拦截器配置中的参数保持一致+
             */
            param = context.getString("param");
        }
    }
}

```
## 3. 在flume配置文件中添加拦截器
```properties
#-------------------source1:interceptor------------
agent.sources.source1.interceptors = i2
agent.sources.source1.interceptors.i2.type = com.owp.flumeinterceptor.MyInterceptor$Builder
agent.sources.source1.interceptors.i2.param=parameter
```
* 说明：agent.sources.source1.interceptors.i2.param=parameter配置中等号前的param必须要代码中` context.getString("param")`中getString的参数保持一致

## 4. 打成全量jar放到flume的jar中(打包方式请百度ideal打全量jar）
## 5. 启动fulme查看结果
```shell
/home/flume/apache-flume-1.9.0-bin/bin/flume-ng agent  --conf conf --conf-file /home/flume/apache-flume-1.9.0-bin/conf/flume-conf-kafkad.properties  --name x9ec -Dflume.root.logger=INFO,console > /home/flume/logs/x9ed.log 2>&1 &
```
