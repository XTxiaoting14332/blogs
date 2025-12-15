---
title: 绕过25端口搭建自己的邮件服务器
date: 2025-12-15 19:00:47
categories: 
- 技术
tags:
- 邮件服务器
- Linux
---

最近整了台小鸡，正好又有自己的域名，于是心血来潮想搭一个邮件服务器。

# 前言
前几天也搜索过相关教程，发现要搭建邮件服务器需要先开启25端口，很不巧的是，目前大部分服务商都不允许放行25端口，所以只能另寻他法。
经过与AI的多轮探讨，最终决定采用**邮件中转**的方式来绕过25端口的限制。

# 选择中转服务

这里我个人推荐[Brevo](https://app.brevo.com/ "点我前往")，上手较快，使用起来简单，而且免费额度较为宽松，每天能发送300封邮件，对于个人自建的服务器来说完全够用。

注册好以后进入[SMTP&API](https://app.brevo.com/settings/keys/smtp "点我前往")，点击页面的 `Generate a new SMTP key` 按钮创建一个 API Key，并保管好备用。

<br>![](/img/posts/brevo1.png)<br>


# 安装 Postfix

以 Debian 为例
```bash
sudo apt install postfix
```

这时，会弹出如下的窗口，直接选择 `Internet with smarthost` 然后回车。
![](/img/posts/postfix1.jpg)

下一步到这里，把输入栏中的文本删完，并填入你邮件服务器的域名，比如 `admin@example.com` 就填 `example.com` 然后回车。
![](/img/posts/postfix2.webp)

下一步是填 SMTP 服务器，直接删掉原有的文本，填上 Brevo 的 `[smtp-relay.brevo.com]:587` 然后回车。
![](/img/posts/postfix3.webp)


至此， Postfix 安装完成。

# 配置 Postfix

首先我们需要创建一个登录凭证

```bash
sudo vi /etc/postfix/sasl_passwd
```

并往里面写入如下内容

```
[smtp-relay.brevo.com]:587 Brevo的登录用户:Brevo的API
```

其中登录用户是在[SMTP&API](https://app.brevo.com/settings/keys/smtp "点我前往")中的`Login`字段，会有一个类似于 `114514abc19@smtp-brevo.com` 的邮箱地址。

API就是刚刚获取到的 API Key.

然后使用以下命令将凭证转为数据库格式，并限制权限。
```bash
sudo postmap /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd.db
```

接下来创建邮件服务器需要的证书，这里我们直接使用 `Let's Encrypt` 通过 `Certbot` 获取免费证书。
```bash
sudo apt install certbot
# 先关闭 Postfix 服务
sudo systemctl stop postfix
sudo certbot certonly --standalone -d 你的服务器域名
```
接着会出现几个选项让你选择，全选择Y即可，接着就会签发一个3个月有效期的证书，并给出路径。
`/etc/letsencrypt/live/你的服务器域名/fullchain.pem`
`/etc/letsencrypt/live/你的服务器域名/privkey.pem`

接下来编辑`/etc/postfix/main.cf`,将`smtpd_tls_cert_file`项替换为`/etc/letsencrypt/live/你的服务器域名/fullchain.pem`
将`smtpd_tls_key_file`项替换为`/etc/letsencrypt/live/你的服务器域名/privkey.pem`

并在文件末尾添加以下内容
```
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
smtp_use_tls = yes
smtp_tls_security_level = encrypt
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth 
```

编辑`/etc/postfix/master.cf`并按照图示取消对应的注释
![](/img/posts/postfix4.webp)

启动 Postfix 服务
```bash
sudo systemctl start postfix
```

至此，Postfix服务配置完成，使用以下命令检查是否正常运行。
```bash
sudo ss -tuln | grep 587
```

# 安装 Dovecot
```bash
sudo apt install dovecot-core dovecot-imapd dovecot-pop3d
```

编辑`/etc/dovecot/conf.d/10-master.conf`，找到`postfix`相关字段，并取消对应的注释。
![](/img/posts/dovecot.webp)

重启 Dovecot 和 Postfix 服务
```bash
sudo systemctl restart dovecot
sudo systemctl restart postfix
```

# 测试发送
安装mailutils
```bash
sudo apt install mailutils 
```

发送邮件
```bash
echo "This is a test email." | mail -s "Test" 你的个人邮箱（qq邮箱、网易邮箱之类的）
```

如果没有问题的话，你的邮箱会收到一封主题为`Test`的邮件。

# 配置电子邮件路由和 IMAP 服务
因为25端口无法使用，所以我们只能使用第三方的imap服务做中转，同时需要配置一下电子邮件路由，将收到的邮件转发到你的个人邮箱。

## 配置电子邮件路由
为了方便我们直接使用Cloudflare提供的电子邮件路由服务，不配置路由的话其他人无法将邮件发到你的邮箱。
![](/img/posts/cfmail.webp)
直接重定向到QQ邮箱即可。

## 配置 IMAP 服务
这里我们直接用qq邮箱进行配置，进入QQ邮箱的[个人账号设置](https://wx.mail.qq.com/account "点我前往")，找到`安全设置`,然后开启`POP3/IMAP/SMTP/Exchange/CardDAV 服务`，接着创建授权码，保留备用。

# 使用 Thunderbird 客户端进行连接
[Thunderbird](https://www.thunderbird.net/zh-CN/ "点我前往") 是一款开源免费的邮箱客户端，很适合用来连接我们的邮箱服务器。

## 添加账号
填入你的电子邮件地址，比如admin@example.com,这时候会开始查找配置，不出意外的话会提示`未找到配置`，这时候我们点击`手动配置`，开始正式配置。

## 配置收件服务器
如下图所示，协议IMAP保持不变，服务器改成`imap.qq.com`，端口`993`，安全性`SSL/TLS`，身份验证用普通密码，用户名是你配置IMAP服务时用的QQ邮箱地址，密码是上面拿到的授权码。
![](/img/posts/thunderbird1.png)

## 配置发件服务器
如下图所示，服务器地址填你的服务器域名，安全性`STARTTLS`，端口`587`，验证方式普通密码，用户名是你服务器的登录用户名，密码是服务器登录用的密码，建议创建一个普通用户进行登录。
![](/img/posts/thunderbird2.png)

## PC 端操作
PC端下操作和移动端有些差异，在配置以后需要点击高级设置，然后点击`确定`。
![](/img/posts/tbpc.png)

这会跳转到一个新页面，让你输入IMAP服务器的授权码，然后勾选记住密码。
![](/img/posts/tbpc1.png)

输入以后点击`确定`就配置好了

SMTP服务器则是在发件的时候触发密码输入，这时候需要输入Linux服务器的用户密码。输入以后保存密码，即可正常发送。
![](/img/posts/tbpc2.png)

# 设置过滤
至此，我们的邮件服务器已经搭建完成，但是接入了QQ邮箱的IMAP以后，连同QQ邮箱的收件箱也一起接收过来了，这时候我们可以设置过滤条件。QQ邮箱或者Thunderbird客户端上均可设置，这里直接以QQ邮箱为例。

首先，我们在QQ邮箱的主界面侧边栏新建一个文件夹，名字随便起。接着进入设置，找到`收信规则`，添加规则，把指定收信人的信件移动到刚刚创建的文件夹里。
![](/img/posts/qqmail.png)

配置好以后，收到的信就可以直接在客户端的文件夹中过滤出来，如果没找到的话重启客户端看看。
![](/img/posts/tbpc3.png)

# Enjoy it!
至此，邮件服务器已经搭建完成，快去和朋友发邮件吧！