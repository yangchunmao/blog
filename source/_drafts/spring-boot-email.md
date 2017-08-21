---
title: 'spring-boot邮件email发送功能的支持'
tags: spring-boot
categories: spring-boot
---
>邮件（email）发送是业务需求中不可缺少的一环，`spring boot`通过`Java Email`和`Spring Framework's email`的支持快速、简单的实现邮件的发送功能。
如何通过`spring boot`快速实现email发送呢？

## 准备工作，相应的依赖与配置文件
- 在`pom.xml`中添加依赖
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>
- 配置文件`application.properties`, 可根据自己的实际情况修改
        # email
        spring.mail.default-encoding=utf-8
        spring.mail.host=smtp.163.com
        spring.mail.protocol=smtp
        spring.mail.test-connection=false
        spring.mail.username=邮箱用户名
        spring.mail.password=密码，smtp的密码
        spring.mail.properties.mail.smtp.auth=true
        spring.mail.properties.mail.smtp.starttls.enable=true
        spring.mail.properties.mail.smtp.starttls.required=true
        # 设置超时时间，避免因线程被无响应的邮件服务器阻塞
        spring.mail.properties.mail.smtp.connectiontimeout=5000
        spring.mail.properties.mail.smtp.timeout=3000
        spring.mail.properties.mail.smtp.writetimeout=5000    

## 通过`JavaMailSender`发送简单的邮件
你可通过`SimpleMailMessage`,也可通过`MimeMessagePreparator`添加富文本邮件内容
    
    @Autowired
    private JavaMailSender mailSender;
    @Test
    public void testMail(){
        SimpleMailMessage mailMessage = new SimpleMailMessage();
        mailMessage.setFrom("yangchunmao8805@163.com");
        mailMessage.setTo("766361175@qq.com");
        mailMessage.setSubject("主题：简单邮件");
        mailMessage.setText("测试邮件内容");
        mailSender.send(mailMessage);
    }
    @Test
    public void testMimeMsgMail(){
        mailSender.send((mimeMessage -> {
            mimeMessage.setFrom(new InternetAddress("yangchunmao8805@163.com"));
            mimeMessage.setRecipient(RecipientType.TO,
                    new InternetAddress("766361175@qq.com"));
            mimeMessage.setSubject("主题：富文本测试");
            mimeMessage.setText("测试邮件内容");
        }));
    }

## 通过`MimeMessageHelper`邮件添加附件和内联静态资源

        @Test
        public void testMailAttachment() throws MessagingException {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom("yangchunmao8805@163.com");
            helper.setTo("766361175@qq.com");
            helper.setSubject("主题：测试图片附件");
            helper.setText("查看图片附件");
            helper.addAttachment("CoolImage.jpg", new FileSystemResource(new File("E:\\code128CTEST1.jpg")));
            mailSender.send(message);
        }
        @Test
        public void testMailInlineResource() throws MessagingException {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom("yangchunmao8805@163.com");
            helper.setTo("766361175@qq.com");
            helper.setSubject("主题：测试图片附件内联");
            helper.setText("<html><body><img src='cid:identifier1234'></body></html>", true);
            helper.addInline("identifier1234", new FileSystemResource(new File("E:\\code128CTEST1.jpg")));
            mailSender.send(message);
        }

## 应用`Freemarker`模板框架，设置邮件模板

+ 添加相关`Freemarker`依赖
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
+ 编辑`email`模板文件，`email-template.ftl`
        <!DOCTYPE html>
        <html>
            <body>
                <h3>你好,${user.userName},欢迎来到测试邮件模板</h3>

                <div>
                    你的邮件地址是<a href="mailto:${user.emailAddr}">${user.emailAddr}</a>
                    <#--<img src="cid:tupian12333444" >-->
                </div>
            </body>
        </html>
+ 具体的代码示例
        @Autowired
        private FreeMarkerConfigurer configurable;
        @Test
        public void testVelocityEmail(){
            mailSender.send((mimeMessage -> {
                MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
                helper.setFrom("yangchunmao8805@163.com");
                helper.setTo("766361175@qq.com");
                helper.setSubject("主题：测试图片附件内联123");

                Map model = new HashMap();
                User user = new User();
                user.setUserName("杨春茂");
                user.setEmailAddr("yangchunmao8805@163.com");
                model.put("user", user);

                Template t = configurable.getConfiguration().getTemplate("email-template.ftl");
                String content = FreeMarkerTemplateUtils.processTemplateIntoString(t, model);
                helper.setText(content, true);
                helper.addInline("tupian12333444", new FileSystemResource(new File("E:/th.jpg")));
            }));
        }

## 邮件代码与业务解耦，可通过``Spring AOP aspect`切面化
    @Aspect
    @Component
    public class EmailAop {

        private static final Logger logger = LoggerFactory.getLogger(EmailAop.class);

        @Pointcut("execution(public * org.illuwater.service.*.*(..))")
        public void email(){}

        @Autowired
        private FreeMarkerConfigurer configurer;
        @Autowired
        private JavaMailSender mailSender;
        /**
         * 方法执行返回值后, 固定相应的方法，执行发送邮件请求
         * @param user
         * @throws Throwable
         */
        @AfterReturning( returning = "user", pointcut = "email()")
        public void doAfterReturning(User user) throws Throwable{
            mailSender.send((mimeMessage -> {
                MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
                helper.setFrom("yangchunmao8805@163.com");
                helper.setTo("766361175@qq.com");
                helper.setSubject("主题：测试图片附件内联123");
                Map model = new HashMap();
                model.put("user", user);
                Template t = configurer.getConfiguration().getTemplate("email-template.ftl");
                String content = FreeMarkerTemplateUtils.processTemplateIntoString(t, model);
                helper.setText(content, true);
                helper.addInline("tupian12333444", new FileSystemResource(new File("E:\\th.jpg")));
            }));
        }
    }