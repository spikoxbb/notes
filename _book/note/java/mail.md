[TOC]

# 邮件收发过程

![](/home/spiko/notes/img/1120165-20171011215731184-1615174561.png)

1. 用户A的电子邮箱为：xx@qq.com，通过邮件客户端软件写好一封邮件，交到QQ的邮件服务器，这一步使用的协议是SMTP,对应图示的①；
2. QQ邮箱会根据用户A发送的邮件进行解析，也就是根据收件地址判断是否是自己管辖的账户，如果收件地址也是QQ邮箱，那么会直接存放到自己的存储空间。这里我们假设收件地址不是QQ邮箱，而是163邮箱，那么QQ邮箱就会将邮件转发到163邮箱服务器，转发使用的协议也是SMTP，对应图示的②；
3. 163邮箱服务器接收到QQ邮箱转发过来的邮件，也会判断收件地址是否是自己，发现是自己的账户，那么就会将QQ邮箱转发过来的邮件存放到自己的内部存储空间，对应图示的③；
4. 用户A将邮件发送了之后，就会通知用户B去指定的邮箱收取邮件。用户B会通过邮件客户端软件先向163邮箱服务器请求，要求收取自己的邮件，对应图示的④；
5. 163邮箱服务器收到用户B的请求后，会从自己的存储空间中取出B未收取的邮件，对应图示⑤；
6. 163邮箱服务器取出用户B未收取的邮件后，将邮件发给用户B，对应图示的⑥；
7. 最后三步用户B收取邮件的过程，使用的协议是POP3;

## 邮件服务器

- SMTP邮件服务器：替用户发送邮件和接收外面发送给本地用户的邮件，对应上图的第一、二步。它相当于现实生活中邮局的邮件接收部门（可接收普通用户要投出的邮件和其他邮局投递进来的邮件）。
- POP3/IMAP邮件服务器：帮助用户读取SMTP邮件服务器接收进来的邮件，对应上图的第六步。它相当于专门为前来取包裹的用户提供服务的部门。

## 邮件传输协议

- SMTP协议：全称为 Simple Mail Transfer Protocol，简单邮件传输协议。它定义了邮件客户端软件和SMTP邮件服务器之间，以及两台SMTP邮件服务器之间的通信规则。
- 全称为 Post Office Protocol，邮局协议。它定义了邮件客户端软件和POP3邮件服务器的通信规则。
- 全称为 Internet Message Access Protocol,Internet消息访问协议，它是对POP3协议的一种扩展，也是定义了邮件客户端软件和IMAP邮件服务器的通信规则。

# smtp和pop3协议

## SMTP协议发送邮件

SMTP 协议中一共定义了18条命令，但是发送一封电子邮件的过程通常只需要6条命令:

![](/home/spiko/notes/img/1120165-20171012205818918-1799542502.png)

1. 和SMTP服务器建立连接，telnet smtp.163.com 25。这条命令是和163邮箱建立连接

2. ehlo 发件人用户名，就是告诉SMTP服务器发送者的用户名。

3. 选择登录认证方式，一般在第二步执行完后，会提示有几种认证方式，一般选择的是login。即输入命令：auth login

4. 分别输入经过Base64加密后的用户名和密码。

5. 指明邮件的发送人和收件人

   　　　　mail from:<xxx@163.com>

   　　　　rcpt to:<xxx@qq.com>

6. 输入data命令，然后编写要发送的邮件内容，邮件的编写格式规则如下：

   　　　　第一步：输入data

   　　　　第二步：输入邮件内容　

   ```
   `from:<xxx``@163``.com>　　　　----邮件头发件人地址``to:<xxx``@qq``.com> 　　　　  ----邮件头收件人地址``subject:hello world　　　 ----邮件头主题``　　　　　　　　　　　　　　 -----空行``This is the first email sent by hand using the SMTP protocol　　　　　　　　　----邮件的具体内容`
   ```

7. 输入“.”表示邮件内容输入完毕
8. 输入quit命令断开与邮件服务器的连接

## 使用POP3协议手工接收邮件　

![](/home/spiko/notes/img/1120165-20171013233529668-98299246.png)

# 邮件的组织结构

## RFC822 邮件格式

RFC822 文档中定义的文件格式包括两个部分：邮件头和邮件体.

主要的邮件头字段的解释：

![](/home/spiko/notes/img/1120165-20171013234326574-1153565600.png)

FC822文档存在两个问题：

1. 定义了邮件内容的主体结构和各种邮件头字段的详细细节，但是，它没有定义邮件体的格式，RFC822文档定义的邮件体部分通常都只能用于表述一段普通的文本，而无法表达出图片、声音等二进制数据。
2. SMTP服务器在接收邮件内容时，当接收到只有一个“.”字符的单独行时，就会认为邮件内容已经结束，如果一封邮件正文中正好有内容仅为一个“.”字符的单独行，SMTP服务器就会丢弃掉该行后面的内容，从而导致信息丢失。

由于图片和声音等内容是非ASCII码的二进制数据，而RFC822邮件格式只适合用来表达纯文本的邮件内容，所以，要使用RFC822邮件格式发送这些非ASCII码的二进制数据时，必须先采用某种编码方式将它们“编码”成可打印的ASCII字符后再作为RFC822邮件格式的内容。邮件阅读程序在读取到这种经过编码处理的邮件后，再按照相应的解码方式解码出原始的二进制数据，这样就可以借助RFC822邮件格式来传递多媒体数据了。这种做法需要解决一下两个技术问题：

  1. 邮件阅读程序如何知道邮件中嵌入的原始二进制数据所采用的编码方式；

  2. 邮件阅读程序如何知道每个嵌入的图像或其他资源在整个邮件内容中的起止位置。

     

为了解决上面两个问题，人们后来专门为此定义了MIME（Multipurpose Internet Mail Extension，多用途Internet邮件扩展）协议。

## MIME协议

MIME协议用于定义复杂邮件体的格式，它可以表达多段平行的文本内容和非文本的邮件内容，例如，在邮件体中内嵌的图像数据和邮件附件等。另外，MIME协议的数据格式也可以避免邮件内容在传输过程中发生信息丢失。

MIME邮件在RFC822文档中定义的邮件头字段的基础上，扩充了一些自己专用的邮件头字段，例如，使用MIME-Version头字段指定MIME协议的版本，使用Content-Type头字段指定邮件体的MIME类型，使用Content-Transfer-Encoding头字段指定编码方法，如下所示： 　　　

```json
MIME-Version:1.0 Content-Type:multipart/mixed;boundary="----=_NextPart_000_0050_01C"
```

“multipart/mixed”部分说明邮件体中包含有多段数据，每段数据之间使用boundary属性中指定的字符文本作为分隔标识符。另外，MIME邮件也扩展了RFC822文档中已经定义了的邮件头字段的内涵，例如，定义了subject头字段中的值内容的格式，以便通过编码的方式让邮件主题中也可以使用非ASCII码的字符。subject头字段中的值嵌套在一对“=?”和“?=”标记符之间，标记符之间的内容由三部分组成：邮件主题的原始内容的字符集、当前采用的编码方式、编码后的结果.

# javaMail

## API

- Message 类:javax.mail.Message 类是创建和解析邮件的核心 API，这是一个抽象类，通常使用它的子类javax.mail.internet.MimeMessage 类。它的实例对象表示一份电子邮件。
- Transport 类：javax.mail.Transport 类是发送邮件的核心API 类，它的实例对象代表实现了某个邮件发送协议的邮件发送对象，例如 SMTP 协议。
- javax.mail.Store 类是接收邮件的核心 API 类，它的实例对象代表实现了某个邮件接收协议的邮件接收对象，例如 POP3 协议。
- Session 类：javax.mail.Session 类用于定义整个应用程序所需的环境信息，以及收集客户端与邮件服务器建立网络连接的会话信息，例如邮件服务器的主机名、端口号、采用的邮件发送和接收协议等。Session 对象根据这些信息构建用于邮件收发的 Transport 和 Store 对象。

## 使用 JavaMail 发送邮件

```java
public class Test {
    //发件人地址
    public static String senderAddress = "xx@qq.com";
    //收件人地址
    public static String recipientAddress = "xx@qq.com";
    //发件人账户名
    public static String senderAccount = "xx@qq.com";
    //发件人账户密码
    public static String senderPassword = "xx";

    public static void main(String[] args) throws Exception {
        //1、连接邮件服务器的参数配置
        Properties props = new Properties();
        //设置用户的认证方式
        props.setProperty("mail.smtp.auth", "true");
        //设置传输协议
        props.setProperty("mail.transport.protocol", "smtp");
        //设置发件人的SMTP服务器地址
        props.setProperty("mail.smtp.host", "smtp.qq.com");
        //2、创建定义整个应用程序所需的环境信息的 Session 对象
        Session session = Session.getInstance(props);
        //设置调试信息在控制台打印出来
        session.setDebug(true);
        //3、创建邮件的实例对象
        Message msg = getMimeMessage(session);
        //4、根据session对象获取邮件传输对象Transport
        Transport transport = session.getTransport();
        //设置发件人的账户名和密码
        transport.connect(senderAccount, senderPassword);
        //发送邮件，并发送到所有收件人地址，message.getAllRecipients() 获取到的是在创建邮件对象时添加的所有收件人, 抄送人, 密送人
        transport.sendMessage(msg,msg.getAllRecipients());

        //如果只想发送给指定的人，可以如下写法
        //transport.sendMessage(msg, new Address[]{new InternetAddress("xxx@qq.com")});

        //5、关闭邮件连接
        transport.close();
    }

    public static MimeMessage getMimeMessage(Session session) throws Exception{
        //创建一封邮件的实例对象
        MimeMessage msg = new MimeMessage(session);
        //设置发件人地址
        msg.setFrom(new InternetAddress(senderAddress));
        /**
         * 设置收件人地址（可以增加多个收件人、抄送、密送），即下面这一行代码书写多行
         * MimeMessage.RecipientType.TO:发送
         * MimeMessage.RecipientType.CC：抄送
         * MimeMessage.RecipientType.BCC：密送
         */
        msg.setRecipient(MimeMessage.RecipientType.TO,new InternetAddress(recipientAddress));
        //设置邮件主题
        msg.setSubject("jAVA MAIL","UTF-8");
        //设置邮件正文
        msg.setContent("java mail", "text/html;charset=UTF-8");
        //设置邮件的发送时间,默认立即发送
        msg.setSentDate(new Date());

        return msg;
    }
}
```

## 使用 JavaMail 接收邮件

```java
 public void receive() throws Exception {
        //1、连接邮件服务器的参数配置
        Properties props = new Properties();
        //设置传输协议
        props.setProperty("mail.store.protocol", "pop3");
        //设置收件人的POP3服务器
        props.setProperty("mail.pop3.host", "pop.qq.com");
        //2、创建定义整个应用程序所需的环境信息的 Session 对象
        Session session = Session.getInstance(props);
        //设置调试信息在控制台打印出来
        //session.setDebug(true);

        Store store = session.getStore("pop3");
        //连接收件人POP3服务器
        store.connect(senderAccount, senderPassword);
        //获得用户的邮件账户，注意通过pop3协议获取某个邮件夹的名称只能为inbox
        Folder folder = store.getFolder("inbox");
        //设置对邮件账户的访问权限
        folder.open(Folder.READ_WRITE);

        //得到邮件账户的所有邮件信息
        Message [] messages = folder.getMessages();
        for(int i = 0 ; i < messages.length ; i++){
            //获得邮件主题
            String subject = messages[i].getSubject();
            //获得邮件发件人
            Address[] from = messages[i].getFrom();
            //获取邮件内容（包含邮件内容的html代码）
            Object content = messages[i].getContent();
        }

        System.out.println("count===================="+messages.length);
        //关闭邮件夹对象
        folder.close();
        //关闭连接对象
        store.close();
    }
```

## 使用 JavaMail 发送带图片、附件的邮件

```java
 public MimeMessage getMimeMessage(Session session) throws Exception{
        //创建一封邮件的实例对象
        MimeMessage msg = new MimeMessage(session);
        //设置发件人地址
        msg.setFrom(new InternetAddress(senderAddress));
        /**
         * 设置收件人地址（可以增加多个收件人、抄送、密送），即下面这一行代码书写多行
         * MimeMessage.RecipientType.TO:发送
         * MimeMessage.RecipientType.CC：抄送
         * MimeMessage.RecipientType.BCC：密送
         */
        msg.setRecipient(MimeMessage.RecipientType.TO,new InternetAddress(recipientAddress));
        //设置邮件主题
        msg.setSubject("jAVA MAIL","UTF-8");
        //设置邮件正文
        //msg.setContent("java mail", "text/html;charset=UTF-8");

        // 5. 创建图片"节点"
        MimeBodyPart image = new MimeBodyPart();
        // 读取本地文件
        DataHandler dh = new DataHandler(new FileDataSource("src/main/test/wje.jpg"));
        // 将图片数据添加到"节点"
        image.setDataHandler(dh);
        // 为"节点"设置一个唯一编号（在文本"节点"将引用该ID）
        image.setContentID("pic");

        // 6. 创建文本"节点"
        MimeBodyPart text = new MimeBodyPart();
        // 这里添加图片的方式是将整个图片包含到邮件内容中, 实际上也可以以 http 链接的形式添加网络图片
        text.setContent("这是一张图片<br/><img src='cid:pic'/>", "text/html;charset=UTF-8");

        // 7. （文本+图片）设置 文本 和 图片"节点"的关系（将 文本 和 图片"节点"合成一个混合"节点"）
        MimeMultipart mm_text_image = new MimeMultipart();
        mm_text_image.addBodyPart(text);
        mm_text_image.addBodyPart(image);
        mm_text_image.setSubType("related");    // 关联关系

        // 8. 将 文本+图片 的混合"节点"封装成一个普通"节点"
        // 最终添加到邮件的 Content 是由多个 BodyPart 组成的 Multipart, 所以我们需要的是 BodyPart,
        // 上面的 mailTestPic 并非 BodyPart, 所有要把 mm_text_image 封装成一个 BodyPart
        MimeBodyPart text_image = new MimeBodyPart();
        text_image.setContent(mm_text_image);

        // 9. 创建附件"节点"
        MimeBodyPart attachment = new MimeBodyPart();
        // 读取本地文件
        DataHandler dh2 = new DataHandler(new FileDataSource("src/main/test/wje.jpg"));
        // 将附件数据添加到"节点"
        attachment.setDataHandler(dh2);
        // 设置附件的文件名（需要编码）
        attachment.setFileName(MimeUtility.encodeText(dh2.getName()));

        // 10. 设置（文本+图片）和 附件 的关系（合成一个大的混合"节点" / Multipart ）
        MimeMultipart mm = new MimeMultipart();
        mm.addBodyPart(text_image);
        mm.addBodyPart(attachment);     // 如果有多个附件，可以创建多个多次添加
        mm.setSubType("mixed");         // 混合关系

        // 11. 设置整个邮件的关系（将最终的混合"节点"作为邮件的内容添加到邮件对象）
        msg.setContent(mm);
        //设置邮件的发送时间,默认立即发送
        msg.setSentDate(new Date());

        return msg;
    }
```

