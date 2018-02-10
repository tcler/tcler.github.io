---
layout: post
title: "getFileChecksum in java"
---

### 第一个java程序~ ？？
嗯,, 是朋友问有没有认识会java的，需求是获取文件的 Checksum；第一反应是直接调用 md5sum，然后想一直听说java生态厉害 肯定有现成的 google 一下10分钟搞定

```
[yjh@test FileDigest]$ cat FileDigest.java
import java.util.*;
import java.io.*;
import java.security.*;

public class FileDigest {
        private static String getFileChecksum(File file, String type) throws IOException
        {
                MessageDigest digest = null;

                try {
                        digest = MessageDigest.getInstance(type);
                } catch(NoSuchAlgorithmException e) {
                        //System.err.println("Error: " + e.getMessage());
                        return "";
                }

                //Get file input stream for reading the file content
                FileInputStream fis = new FileInputStream(file);

                //Create byte array to read data in chunks
                byte[] byteArray = new byte[4096];
                int bytesCount = 0;

                //Read file data and update in message digest
                while ((bytesCount = fis.read(byteArray)) != -1) {
                        digest.update(byteArray, 0, bytesCount);
                };

                //close the stream; We don't need it now.
                fis.close();

                //Get the hash's bytes
                byte[] bytes = digest.digest();

                //This bytes[] has bytes in decimal format;
                //Convert it to hexadecimal format
                StringBuilder sb = new StringBuilder();
                for(int i=0; i < bytes.length; i++) {
                        sb.append(Integer.toString((bytes[i] & 0xff) + 0x100, 16).substring(1));
                }

                //return complete hash
                return sb.toString();
        }

        public static void main(String args[]) {
                for (int i=0; i<args.length; i++) {
                        try {
                                File file = new File(args[i]);
                                String checksum = getFileChecksum(file, "MD5");
                                System.out.println(checksum + "\t" + args[i]);
                        } catch(IOException e) {
                                System.err.println("Error: " + e.getMessage());
                        }
                }
        }
}
[yjh@test FileDigest]$ LANG=C javac FileDigest.java
[yjh@test FileDigest]$ java FileDigest  *
ad1ad1959b9f8cecf7c6a64407405f53        FileDigest.class
a8e19dde7126820ce10657bcdd0bc68b        FileDigest.java
```


嗯 感觉 java 也挺方便的~
