---
layout: fragment
title: HDFS Java-API
tags: [HDFS, Java]
description: HDFS Java-API
keywords: Hadoop, HDFS
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
package HDFS;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.apache.hadoop.fs.permission.FsAction;
import org.apache.hadoop.fs.permission.FsPermission;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.util.Progressable;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import java.io.*;
import java.net.URI;
import java.net.URISyntaxException;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/6
 * Created by fujiahao
 */

/**
 * 需求：Hadoop的HDFS-JAVA-API
 *      1.FileSystem
 *      2.创建目录
 *      3.创建指定权限的目录
 *      4.创建文件，并写入内容
 *      5.判断文件是否存在
 *      6.查看文件内容
 *      7.文件重命名
 *      8.删除目录或文件
 *      9.上传文件到HDFS
 *      10.上传大文件并显示上传进度
 *      11.从HDFS上下载文件
 *      12.查看指定目录下所有文件的信息
 *      13.递归查看指定目录下所有文件的信息
 *      14.查看文件的块信息
 */


public class HdfsClient {

    /**
     * FileSystem是所有HDFS的操作主入口
     * 使用@Before进行标注
     */

    private static FileSystem fileSystem;
    @Before
    public void prepare() {
        try {
            Configuration configuration = new Configuration();
            // 单节点Hadoop，副本系数设置为1，默认为3
            configuration.set("dfs.replication", "1");
            fileSystem = FileSystem.get(new URI("hdfs://master:9000"), configuration, "root");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (URISyntaxException e) {
            e.printStackTrace();
        }
    }

    @After
    public void destroy() {
        fileSystem = null;
    }


    /**
     * 创建目录
     */
    @Test
    public void mkDir() throws Exception {
        fileSystem.mkdirs(new Path(""));
    }


    /**
     * 创建指定权限的目录
     */
    public void mkDirWithPermission() throws Exception {
        fileSystem.mkdirs(new Path(""),
                           new FsPermission(FsAction.READ_WRITE, FsAction.READ, FsAction.READ));
    }


    /**
     * 创建文件，并写入内容
     */
    @Test
    public void create() throws Exception {
        // 如果文件存在，默认会覆盖，可以通过第二个参数进行控制
        // 第三个参数可以控制使用缓冲区的大小
        FSDataOutputStream out = fileSystem.create(new Path(""), true, 4096);
        out.write("hello hadoop!".getBytes());
        out.write("hello spark!".getBytes());
        out.write("hello flink!".getBytes());
        // 强制将缓冲区中内容刷出
        out.flush();
        out.close();
    }


    /**
     * 判断文件是否存在
     */
    @Test
    public void exist() throws Exception {
        boolean exists = fileSystem.exists(new Path(""));
        System.out.println(exists);
    }


    /**
     * 查看文件内容(小文本文件内容，直接转换成字符串后输出)
     */

    @Test
    public void readToString() throws Exception {
        FSDataInputStream inputStream = fileSystem.open(new Path(""));
        String context = inputStreamToString(inputStream, "utf-8");
        System.out.println(context);
    }

    /**
     * 自定义inputStreamToString方法
     * @param inputStream 输入流
     * @param encode 指定编码类型
     * @return 返回内容
     */
    private static String inputStreamToString(InputStream inputStream, String encode) {
        try {
            if (encode == null || ("".equals(encode))) {
                encode = "utf-8";
            }
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, encode));
            StringBuilder builder = new StringBuilder();
            String str = "";
            while ((str = reader.readLine()) != null) {
                builder.append(str).append("\n");
            }
            return builder.toString();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 文件重命名
     */
    @Test
    public void rename() throws Exception {
        Path oldPath = new Path("");
        Path newPath = new Path("");
        boolean result = fileSystem.rename(oldPath, newPath);
        System.out.println(result);
    }


    /**
     * 删除目录或文件
     */
    @Test
    public void delete() throws Exception {
        // 第二个参数代表是否递归删除
        // 如果path是一个目录，递归删除为true，则删除该目录及所有文件
        // 如果path是一个目录，递归删除为false，则会抛出异常
        boolean result = fileSystem.delete(new Path(""), true);
        System.out.println(result);
    }


    /**
     * 上传文件到HDFS
     */
    @Test
    public void copyFromLocalFile() throws Exception {
        // 如果指定的是目录，则会把目录及其中的文件都复制到指定目录下
        Path src = new Path("");
        Path dst = new Path("");
        fileSystem.copyFromLocalFile(src, dst);
    }


    /**
     * 上传大文件并显示上传进度
     */
    @Test
    public void copyFromLocalBigFile() throws Exception {
        File file = new File("");
        final float fileSize = file.length();
        InputStream in = new BufferedInputStream(new FileInputStream(file));

        FSDataOutputStream out = fileSystem.create(new Path(""),
                new Progressable() {
                    long fileCount = 0;

                    @Override
                    public void progress() {
                        fileCount++;
                        // progress：每次上传64KB数据后就会被调用一次
                        System.out.println("上传进度：" + (fileCount * 64 * 1024 / fileSize) * 100 + " %");
                    }
                });
        IOUtils.copyBytes(in, out, 4096);
    }


    /**
     * 从HDFS上下载文件
     */
    @Test
    public void copyToLocalFile() throws Exception {
        Path src = new Path("");
        Path dst = new Path("");
        // 第一个参数：下载完成后是否删除原文件，默认是true
        // 最后一个参数：是否将RawLocalFileSystem用作本地文件系统
        fileSystem.copyToLocalFile(false, src, dst, true);
    }


    /**
     * 查看指定目录下所有文件的信息
     */
    @Test
    public void listFiles() throws Exception {
        FileStatus[] statuses = fileSystem.listStatus(new Path(""));
        for (FileStatus fileStatus : statuses) {
            // fileStatus的toString方法被重写过，直接打印可以看到所有信息
            System.out.println(fileStatus.toString());
        }
    }


    /**
     * 递归查看指定目录下所有文件的信息
     */
    @Test
    public void listFilesRecursive() throws Exception {
        RemoteIterator<LocatedFileStatus> files = fileSystem.listFiles(new Path(""), true);
        while (files.hasNext()) {
            System.out.println(files.next());
        }
    }


    /**
     * 查看文件的块信息
     */
    @Test
    public void getFileBlockLocations() throws Exception {
        FileStatus fileStatus = fileSystem.getFileStatus(new Path(""));
        BlockLocation[] blocks = fileSystem.getFileBlockLocations(fileStatus, 0, fileStatus.getLen());
        for (BlockLocation block : blocks) {
            System.out.println(block);
        }
    }

}
```

