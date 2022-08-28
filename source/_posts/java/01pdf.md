---
title: 01动态生成world
tags: java
categories: java spring
---


### ftl 转world

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
    <version>2.4.0</version>
</dependency>
```

<!--more-->


```java
package org.clxmm.pdf.util;


import freemarker.template.Configuration;
import freemarker.template.Template;
import freemarker.template.TemplateException;

import java.io.*;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * @author clxmm
 * @Description
 * @create 2022-08-07 21:10
 */
public class DocUtil {

    private Configuration configuration = null;

    public DocUtil() {
        configuration = new Configuration();
        configuration.setDefaultEncoding("utf-8");
    }


    /**
     * @param dataMap 入参数据
     * @param ftlPath ftl文件存放路径（不包括文件名）
     * @param ftlName ftl文件名
     * @param outfile 生成的docx文件
     * @throws UnsupportedEncodingException
     */
    public void createDoc(Map<String, Object> dataMap, String outfile, String ftlPath, String ftlName) throws UnsupportedEncodingException {
        //dataMap 要填入模本的数据文件
        //设置模本装置方法和路径,FreeMarker支持多种模板装载方法。可以重servlet，classpath，数据库装载，
        //这里我们的模板是放在template包下面
//        configuration.setClassForTemplateLoading(this.getClass(), ftlPath);
        Template template = null;
        try {
            //test.ftl为要装载的模板
            // 基于文件系统。 比如加载/home/user/template下的模板文件。
            configuration.setDirectoryForTemplateLoading(new File(ftlPath));
            template = configuration.getTemplate(ftlName);
        } catch (IOException e) {
            e.printStackTrace();
        }
        //输出文档路径及名称
        File outFile = new File(outfile);
        Writer out = null;
        FileOutputStream fos = null;
        try {
            fos = new FileOutputStream(outFile);
            OutputStreamWriter oWriter = new OutputStreamWriter(fos, "UTF-8");
            //这个地方对流的编码不可或缺，使用main（）单独调用时，应该可以，但是如果是web请求导出时导出后word文档就会打不开，并且包XML文件错误。主要是编码格式不正确，无法解析。
            out = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(outFile)));
//            out = new BufferedWriter(oWriter);
        } catch (FileNotFoundException e1) {
            e1.printStackTrace();
        }
        try {
            template.process(dataMap, out);
            out.close();
            fos.close();
        } catch (TemplateException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws UnsupportedEncodingException {
        String path = ClassLoader.getSystemResource("").getPath();
        String tmpFilePath = path + "temp/";
        String ftlPath = path + "pdf/";
        Map<String, Object> dataMap = new HashMap<>();
        long time = new Date().getTime();
        String ftlFile = "test.ftl";
        String outDocFile = tmpFilePath + "信息表" + time + ".docx";
//        String outPdfFile = tmpFilePath+"信息表"+time +".pdf";
        // ftl由系统自动带出的数据
        dataMap.put("name", "clxmm");
        dataMap.put("age", "24");
        DocUtil docUtil = new DocUtil();
        docUtil.createDoc(dataMap, outDocFile, ftlPath, ftlFile);
    }
}

```


### h2


