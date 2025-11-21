---
title: IO流、序列化、反序列化
date: 2022-10-21 09:42:43
categories:
    - Java
tags:
    - IO流
    - 序列化
---
# IOStream

1.字节流（操作字节）

字节流的抽象父类

InputStream

OutputStream

```java
ByteCopy
    private static void test4() throws IOException {
        FileInputStream fis = new FileInputStream("jietu.png");
        FileOutputStream fos = new FileOutputStream("copy.png");

        byte[] bytes = new byte[1024];//“容器”
        int len;
        while ((len=fis.read(bytes))!=-1){
            fos.write(bytes,0,len);
        }
        fos.close();
        fis.close();
}
```

BufferedInputStream类

BufferedOutputStream类

BufferedInputStream内置了一个缓冲区（数组）

从BufferedInputStream中读取一个字节时BufferedInputStream会一次性从文件中读取8192个，存在缓冲区中，每次返回给程序一个

程序再次读取时，就不用找文件了，直接从缓冲区中获取，知道缓冲区中所有的都被使用过，才重新从文件中读取8192个。

BufferedOutputStream也内置了一个缓冲区（数组）

程序向流中写出字节时，不会直接写到文件，先写到缓冲区中，知道缓冲区写满，BufferedOutputStream才会吧缓冲区中的数据一次

性写到文件里。

```
BufferCopy
public static void test5() throws IOException {
    FileInputStream fis = new FileInputStream("jietu.png");
    BufferedInputStream bis = new BufferedInputStream(fis);
    FileOutputStream fos = new FileOutputStream("copy.png");
    BufferedOutputStream bos = new BufferedOutputStream(fos);
    
    int b;
    while ((b = bis.read()) != -1) {
        bos.write(b);
    }

    bis.close();//close方法具备刷新的功能，在关闭之前们就会先刷新一次缓冲区，将缓冲区的字节全部都刷新到文件上，再关闭
    bos.close();
}
```

字节流读取中文时是可能有问题的，因为有时可能会读到半个中文造成乱码。

输入输出的标准代码

```
public static void test4(){
    FileInputStream fis;
    FileOutputStream fos;
    try {
        fis = new FileInputStream("jietu.png");
        fos = new FileOutputStream("copy.png");
        int b;
        while ((b = fis.read()) != -1) {
            fos.write(b);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }finally {
        try {
            if(bis != null)
            fis.close();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
}

```

2.字符流（操作字符）

字符流的抽象父类

Reader类

Writer类

DateInputStream类

DateOutputStream类

4.IO程序书写

使用前，导入IO包中的类

使用时，进行IO异常处理

使用后，释放资源



# Serializable（json、ProtoBuf）

(Java序列化就是指把Java对象转换为字节序列的过程

Java反序列化就是指把字节序列恢复为Java对象的过程。

序列化最重要的作用：在传递和保存对象时.保证对象的完整性和可传递性。对象转换为有序字节流,以便在网络上传输或者保存在本地文件中。

反序列化的最重要的作用：根据字节流中保存的对象状态及描述信息，通过反序列化重建对象。

总结：核心作用就是对象状态的保存和重建。（整个过程核心点就是字节流中所保存的对象状态及描述信息）



实体类序列化

对象序列化：ObjectOutputStream类

对象反序列化：ObjectInputStream类

```
/**
 * @description:
 * @author: Kblayt
 * @time: 2022/2/28 11:29
 */
public class Student implements Serializable {

    private static final long serialVersionUID = -1;

    private String Name;
    private String Age;
    
    public String getName() {
        return Name;
    }
    public void setName(String name) {
        Name = name;
    }
    public String getAge() {
        return Age;
    }
    public void setAge(String age) {
        Age = age;
    }
    public Student(String name, String age) {
        Name = name;
        Age = age;
    }
}

```

```
/**
 * @description:序列化和反序列化
 * @author: Kblayt
 * @time: 2022/2/28 11:32
 */
public class SerializableTest {
    public static void main(String[] args) throws IOException {

        //序列化
        FileOutputStream fos;
        ObjectOutputStream oos = null;
        Student s1 = null;
        try {
            fos = new FileOutputStream("object.out");
            oos = new ObjectOutputStream(fos);
            s1 = new Student("lhg","18");
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                oos.writeObject(s1);
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                oos.flush();
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                oos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        //反序列化
        FileInputStream fis = null;
        ObjectInputStream ois = null;
        try {
            fis = new FileInputStream("object.out");
            ois = new ObjectInputStream(fis);
            Student s2 = (Student) ois.readObject();
            System.out.println(s2.getName()+""+s2.getAge());
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }finally {
            ois.close();
        }
    }
}
```



