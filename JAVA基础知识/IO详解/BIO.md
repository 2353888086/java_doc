> BIO

- Java BIO就是传统的java IO编程
- BIO 同步阻塞，服务器实现模式为一个连接为一个线程，即客户端有连接请求是服务器端就需要启动一个线程进行处理



> BIO 工作机制

![1621755385197](E:\Z-资料\总结文档\文档图片\1621755385197.png)



> 实操一

~~~java
public class Client {
    public static void main(String[] args) throws IOException {
        /*创建Socket对象与服务端创建一个socket连接*/
        Socket socket = new Socket("127.0.0.1", 9999);
        /*Socket对象获取一个字节输出流*/
        OutputStream out = socket.getOutputStream();
        /*把字节输出流包装成一个打印流*/
        PrintStream ps = new PrintStream(out);
        ps.println("客户端端向服务端输出信息");
        ps.flush();
    }
}

public class Service {
    public static void main(String[] args)  {

        try {
            /*定义一个ServerSocket对象进行服务端的端口注册*/
            ServerSocket ss = new ServerSocket(9999);
            /*监听客户端的socket连接请求*/
            Socket socket = ss.accept();
            /*从socket管道中获取一个输入流对象*/
            InputStream is = socket.getInputStream();
            /*把字节输入流包装成一个缓冲字符输入流*/
            BufferedReader br = new BufferedReader(new InputStreamReader(is));

            String read = null;
            if((read=br.readLine()) !=null){
                System.out.println("接收信息"+read);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~



- 以上通信中，服务端会一直等待客户端的消息，如果客户端没有进行消息的发送，服务端将一直进入阻塞状态
- 如果服务端是按照行获取消息的，则客户端必须按照行进行消息发送，否则服务端将进入等待消息的阻塞状态



> 多发多收机制

~~~java
public class Client {
    public static void main(String[] args) throws IOException {
        /*创建Socket对象与服务端创建一个socket连接*/
        Socket socket = new Socket("127.0.0.1", 9999);
        /*Socket对象获取一个字节输出流*/
        OutputStream out = socket.getOutputStream();
        /*把字节输出流包装成一个打印流*/
        PrintStream ps = new PrintStream(out);
        Scanner sc = new Scanner(System.in);
        while(true){
            System.out.print("输入：");
            String msg = sc.nextLine();
            ps.println(msg);
            ps.flush();
        }

    }
}

public class Service {
    public static void main(String[] args)  {

        try {
            /*定义一个ServerSocket对象进行服务端的端口注册*/
            ServerSocket ss = new ServerSocket(9999);
            /*监听客户端的socket连接请求*/
            Socket socket = ss.accept();
            /*从socket管道中获取一个输入流对象*/
            InputStream is = socket.getInputStream();
            /*把字节输入流包装成一个缓冲字符输入流*/
            BufferedReader br = new BufferedReader(new InputStreamReader(is));

            String read = null;
            while((read=br.readLine()) !=null){
                System.out.println("接收信息"+read);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~



> BIO 接收多个客户端(client 不变)

~~~java
public class Service {
    public static void main(String[] args)  {

        try {
            System.out.println("---start---");
            /*定义一个ServerSocket对象进行服务端的端口注册*/
            ServerSocket ss = new ServerSocket(9999);
            while(true){
                /*监听客户端的socket连接请求*/
                Socket socket = ss.accept();
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        /*从socket管道中获取一个输入流对象*/
                        InputStream is = null;
                        try {
                            is = socket.getInputStream();
                            /*把字节输入流包装成一个缓冲字符输入流*/
                            BufferedReader br = new BufferedReader(new InputStreamReader(is));

                            String read;
                            while((read = br.readLine())!=null){
                                System.out.println("接收信息"+read);
                            }
                        } catch (IOException e) {
                            e.printStackTrace();
                        }

                    }
                }).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~

> 小结

- 每个socket接收，都会创建一个线程，线程竞争，切换上下文影响性能
- 线程会占用栈空间和CPU性能
- 并不是所有的线程都会进行IO操作，无意义线程处理
- 多并发会导致线程溢出，线程创建失败，导致宕机



> 伪异步IO

