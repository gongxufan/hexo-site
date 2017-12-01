---
layout: post
title: "使用commons-net进行ftp下载的注意事项"
date: 2016-07-11 11:40
tags: java 
category: 随笔
description: 使用commons-net递归下载指定目录下的所有文件。

---
简单实现ftp递归下载，直接上代码。
```java
package cn.com.egova.easyshare.common.utils;  
  
import org.apache.commons.net.ftp.FTP;  
import org.apache.commons.net.ftp.FTPClient;  
import org.apache.commons.net.ftp.FTPFile;  
import org.apache.commons.net.ftp.FTPReply;  
import org.apache.log4j.Logger;  
  
import java.io.File;  
import java.io.FileOutputStream;  
import java.io.IOException;  
import java.net.SocketException;  
  
/** 
 * 使用apache的commons-net下载ftp文件或者文件夹到本地 
 * 一定要注意中文路径或者哟中文的文件，在执行ftp命令的时候要转换FTP协议的规定编码 
 * Created by gxf on 17-6-12. 
 */  
public class FtpUtil {  
    private String username;  
    private String password;  
    private String ftpHostName;  
    private int port = 21;  
  
    private FTPClient ftpClient = new FTPClient();  
  
    private final Logger logger = Logger.getLogger(FtpUtil.class);  
  
    private static FtpUtil instance;  
  
    /** 
     * 本地字符编码 
     */  
    private static String LOCAL_CHARSET = "GBK";  
  
    // FTP协议里面，规定文件名编码为iso-8859-1  
    private static String SERVER_CHARSET = "ISO-8859-1";  
  
    public static FtpUtil getInstance(String username, String password, String ftpHostName, int port) {  
        if (instance == null)  
            instance = new FtpUtil(username, password, ftpHostName, port);  
        return instance;  
    }  
  
    public FtpUtil(String username, String password,  
                   String ftpHostName, int port) {  
        this.username = username;  
        this.password = password;  
        this.ftpHostName = ftpHostName;  
        this.port = port;  
    }  
  
  
    /** 
     * 建立连接,设置各种连接属性 
     */  
    public void connect() {  
        try {  
            logger.debug("开始连接");  
            // 连接  
            ftpClient.connect(ftpHostName, port);  
            int reply = ftpClient.getReplyCode();  
            if (!FTPReply.isPositiveCompletion(reply)) {  
                ftpClient.disconnect();  
            }  
            // 登录  
            ftpClient.login(username, password);  
            ftpClient.setBufferSize(1024 * 1024);  
            ftpClient.setFileType(FTP.BINARY_FILE_TYPE);  
            ftpClient.enterLocalPassiveMode();  
            //本地文件名编码  
            ftpClient.setControlEncoding(LOCAL_CHARSET);  
            logger.debug("登录成功！");  
        } catch (SocketException e) {  
            logger.error("", e);  
        } catch (IOException e) {  
            logger.error("", e);  
        }  
    }  
  
    /** 
     * 关闭输入输出流 
     */  
    public void close() {  
        try {  
            ftpClient.logout();  
            logger.info("退出登录");  
            ftpClient.disconnect();  
            logger.info("关闭连接");  
        } catch (IOException e) {  
            logger.error("", e);  
        }  
    }  
  
  
    /** 
     * @param ftpFileName ftp服务器上的文件或者路径 
     * @param localDir    本地保存的文件或者路径 
     * @throws Exception 
     */  
    public void downFileOrDir(String ftpFileName, String localDir, boolean isDir) throws Exception {  
        //本地输出流记得手动关闭，系统调用不会关闭本地流  
        FileOutputStream fos = null;  
        try {  
            boolean downloadSuccess = false;  
            File temp = new File(localDir);  
            if (!temp.exists() && isDir) {  
                temp.mkdirs();  
            }  
            // 判断是否是目录  
            if (isDir) {  
                //切换到当前目录  
                if (!ftpClient.changeWorkingDirectory(  
                        new String(ftpFileName.getBytes(LOCAL_CHARSET), SERVER_CHARSET))) {  
                    logger.debug("目录：" + ftpFileName + " 切换失败," + ftpClient.getReplyString());  
                    return;  
                } else {  
                    logger.debug("目录：" + ftpFileName + " 切换成功," + ftpClient.getReplyString());  
                }  
                FTPFile[] ftpFile = ftpClient.listFiles();  
                for (int i = 0; i < ftpFile.length; i++) {  
                    String fileName = ftpFile[i].getName();  
                    //两个隐藏目录忽略  
                    if (".".equals(fileName) || "..".equals(fileName))  
                        continue;  
                    if (ftpFileName.endsWith("/"))  
                        ftpFileName = ftpFileName.substring(0, ftpFileName.length() - 1);  
                    if (localDir.endsWith(File.separator))  
                        localDir = localDir.substring(0, localDir.length() - 1);  
                    //递归  
                    downFileOrDir(ftpFileName + "/" + fileName, localDir  
                            + File.separator + fileName, ftpFile[i].isDirectory() ? true : false);  
                }  
            } else {  
                File localfile = new File(localDir);  
                String remoteFileName = new String(ftpFileName.getBytes(LOCAL_CHARSET), SERVER_CHARSET);  
                if (!localfile.exists()) {  
                    fos = new FileOutputStream(localfile);  
                    downloadSuccess = ftpClient.retrieveFile(remoteFileName, fos);  
                } else {  
                    logger.debug("开始删除本地文件：" + localfile);  
                    localfile.delete();  
                    logger.debug("文件已经删除");  
                    fos = new FileOutputStream(localfile);  
                    downloadSuccess = ftpClient.retrieveFile(remoteFileName, fos);  
                }  
                if (!downloadSuccess)  
                    logger.debug(localfile.getCanonicalPath() + " 下载失败,返回代码：" + ftpClient.getReplyString());  
                else  
                    logger.debug(localfile.getCanonicalPath() + " 下载成功,返回代码：" + ftpClient.getReplyString());  
            }  
        } catch (Exception e) {  
            logger.error("下载失败！", e);  
            throw e;  
        } finally {  
            if (fos != null) {  
                fos.close();  
            }  
        }  
    }  
  
  
    public String getUsername() {  
        return username;  
    }  
  
    public void setUsername(String username) {  
        this.username = username;  
    }  
  
    public String getPassword() {  
        return password;  
    }  
  
    public void setPassword(String password) {  
        this.password = password;  
    }  
  
    public String getFtpHostName() {  
        return ftpHostName;  
    }  
  
    public void setFtpHostName(String ftpHostName) {  
        this.ftpHostName = ftpHostName;  
    }  
  
  
    public int getPort() {  
        return port;  
    }  
  
    public void setPort(int port) {  
        this.port = port;  
    }  
  
    public static void main(String[] args) throws Exception {  
        FtpUtil ftpUtil = FtpUtil.getInstance("administrator", "eland_2014", "192.168.177.12", 21);  
  
        ftpUtil.connect();  
  
        String filePath = "/coredatabase/575A651B5B424200B160F92854DE986A/12220165325039161771642/";  
        String localPath = filePath.replace("/",File.separator);  
        ftpUtil.downFileOrDir(filePath, "D:\\test" + localPath, true);  
  
        filePath = "/coredatabase/7917FCF268BE4CE09DD1C0154F9A3B0B/12920165325039689107501/";  
        localPath = filePath.replace("/",File.separator);  
        ftpUtil.downFileOrDir(filePath, "D:\\test" + localPath, true);  
  
  
        filePath = "/coredatabase/D1DEDCA33DDF47E6BA0FD697245C5D14/12320155325006100699716/";  
        localPath = filePath.replace("/",File.separator);  
        ftpUtil.downFileOrDir(filePath, "D:\\test" + localPath, true);  
  
        filePath = "/coredatabase/3AA5084212284DCEB358F57F33BBE71D/13020165325036258525268/";  
        localPath = filePath.replace("/",File.separator);  
        ftpUtil.downFileOrDir(filePath, "D:\\test" + localPath, true);  
  
        ftpUtil.close();  
    }  
}  
```
代码会递归下载传入的目录下的文件，并在本地生成ftp服务器同样的的目录结构,中文路径和文件需要对中文进行编码转换，代码中服务器中文编码是GBK的。