---
title: SpringBoot复制resource中的资源目录到指定目录
description: 当想在程序运行时动态的更改部分配置,可以将配置放置在resource目录中,然后程序启动时在进行配置文件的读取,想要动态刷新配置可以重复的进行读取,或者代码中监听文件的变动,此时就需要将resource目录中的文件,拷贝到jar包外部的动作
date: 2025-03-24 17:00:00 +0800
categories: [后端, 技术细节]
tags: [SpringBoot,框架操作,操作resource]
toc: true
comments: true
mermaid: true
math: true
pin: false
image:
  path: /assets/posts/2025-03-24-SpringBoot复制resource中的资源目录到指定目录/poster.webp
  alt: SpringBoot复制resource中的资源目录到指定目录
---

## 一、工具类（依赖hutool）
```java
import cn.hutool.core.io.FileUtil;
import cn.hutool.core.io.resource.ResourceUtil;
import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.util.StrUtil;
import lombok.extern.slf4j.Slf4j;
import sun.net.www.protocol.file.FileURLConnection;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.JarURLConnection;
import java.net.URL;
import java.net.URLConnection;
import java.util.Enumeration;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

/**
 * @description jar包文件拷贝工具
 **/
@Slf4j
public class ResourceFileCopyUtil {

    /**
     * 从jar中拷贝一份库文件副本到jar的运行同级目录下,用于后续加载
     *
     * @param resourceRelativePath resource目录下资源相对路径,示例 /resources/templates/ 目录则传templates
     * @param targetFolderPath     目标的外部系统的文件夹路径(绝对路径),给空则默认当前jar包所在路径
     * @throws IOException
     */
    public static String copyRecourseFromJarByFolder(String resourceRelativePath, String targetFolderPath) {
        File file = copyRecourseFromJarByFolder(resourceRelativePath, StrUtil.isEmpty(targetFolderPath) ? null : new File(targetFolderPath));
        return file.getAbsolutePath();
    }

    public static File copyRecourseFromJarByFolder(String resourceRelativePath, File targetFolderPath) {
        //从jar中拷贝一份库文件副本到jar的运行同级目录下,用于后续加载
        File distPath = targetFolderPath;
        if (ObjectUtil.isNull(targetFolderPath)) {
            String jarPath = ResourceFileCopyUtil.class.getProtectionDomain().getCodeSource().getLocation().getPath();
            log.info("copyRecourseFromJarByFolder: 当前jar包路径:{}", jarPath);
            if (jarPath.contains("!") && jarPath.contains(":")) {//jar启动时获取的路径有所不同
                jarPath = jarPath.substring(jarPath.indexOf(":") + 1, jarPath.indexOf("!"));
                jarPath = new File(jarPath).getParent();
                log.info("copyRecourseFromJarByFolder: 当前jar包规整后路径:{}", jarPath);
            }
            distPath = new File(jarPath);
        }
        File finalDistPath = new File(distPath, resourceRelativePath);
        log.info("copyRecourseFromJarByFolder: 目标路径:{}", finalDistPath);
        if (!finalDistPath.exists()) {
            //如果不存在,则拷贝到jar同级目录下
            log.info("copyRecourseFromJarByFolder:文件不存在,进行拷贝!");
            try {
                ResourceFileCopyUtil.loadRecourseFromJarByFolder(resourceRelativePath, distPath.getAbsolutePath());
            } catch (Exception e) {
                log.error("copyRecourseFromJarByFolder: 失败,请检查配置项是否正确:{}", e.getMessage());
                throw new RuntimeException(e);
            }
        } else {
            //如果已经存在,则不再拷贝
            log.info("copyRecourseFromJarByFolder:已经存在,无需拷贝!");
            //TODO  拓展校验文件的md5是否和现有相同,如果不同,则重新拷贝
        }
        return finalDistPath;
    }

    public static void loadRecourseFromJarByFolder(String folderPath, String targetFolderPath) throws IOException {
        URL url = ResourceUtil.getResource(folderPath);
        log.info("loadRecourseFromJarByFolder: url：{}", url);
        URLConnection urlConnection = url.openConnection();
        if (urlConnection instanceof FileURLConnection) {
            log.info("loadRecourseFromJarByFolder: IDEA启动模式：{}", url);
            copyFileResources(url, folderPath, targetFolderPath);
        } else if (urlConnection instanceof JarURLConnection) {
            log.info("loadRecourseFromJarByFolder: JAR 启动模式：{}", url);
            copyJarResources((JarURLConnection) urlConnection, folderPath, targetFolderPath);
        }
    }

    /**
     * 当前运行环境资源文件是在文件里面的
     *
     * @param url
     * @param folderPath
     * @throws IOException
     */
    private static void copyFileResources(URL url, String folderPath, String targetFolderPath) throws IOException {
        File root = new File(url.getPath());
        if (root.isDirectory()) {
            File[] files = root.listFiles();
            for (File file : files) {
                if (file.isDirectory()) {
                    loadRecourseFromJarByFolder(folderPath + "/" + file.getName(), targetFolderPath);
                } else {
                    loadRecourseFromJar(folderPath + "/" + file.getName(), targetFolderPath);
                }
            }
        }
    }

    /**
     * 当前运行环境资源文件是在jar里面的
     *
     * @param jarURLConnection
     * @throws IOException
     */
    private static void copyJarResources(JarURLConnection jarURLConnection, String folderPath, String targetFolderPath) throws IOException {
        JarFile jarFile = jarURLConnection.getJarFile();
        Enumeration<JarEntry> entrys = jarFile.entries();
        while (entrys.hasMoreElements()) {
            JarEntry entry = entrys.nextElement();
            if (entry.getName().startsWith(jarURLConnection.getEntryName()) && !entry.getName().endsWith("/")) {
                loadRecourseFromJar("/" + entry.getName(), targetFolderPath);
            }
        }
        jarFile.close();
    }

    public static void loadRecourseFromJar(String path, String recourseFolder) throws IOException {
        if (!path.startsWith("/")) {
            throw new IllegalArgumentException("The path has to be absolute (start with '/').");
        }

        if (path.endsWith("/")) {
            throw new IllegalArgumentException("The path has to be absolute (cat not end with '/').");
        }
        int index = path.lastIndexOf('/');

        String filename = path.substring(index + 1);
        String folderPath = recourseFolder + path.substring(0, index + 1);

        // If the file does not exist yet, it will be created. If the file
        // exists already, it will be ignored
        filename = folderPath + filename;
        File file = new File(filename);

        // Open and check input stream
        URL url = ResourceUtil.getResource(path);
        URLConnection urlConnection = url.openConnection();
        InputStream is = urlConnection.getInputStream();
        if (is == null) {
            throw new FileNotFoundException("File " + path + " was not found inside JAR.");
        }
        FileUtil.writeFromStream(is, file);
    }

}
```

## 二、使用
```java
//服务初始化后,拷贝resource目录下的templates目录到jar包的同级目录下，返回最终的文件根目录
@PostConstruct
public void init() {
    String distPath = ResourceFileCopyUtil.copyRecourseFromJarByFolder("templates", "");
}
//服务初始化后,拷贝resource目录下的config/1.cfg文件到jar包的同级目录下，返回最终文件路径
@PostConstruct
public void init() {
    String distPath = ResourceFileCopyUtil.copyRecourseFromJarByFolder("config/1.cfg", "");
}
```