---
title: 使用gpg-agent的SSH功能
slug: use-the-ssh-feature-of-gpg-agent
categories:
  - 系统管理
---

GPG和SSH是非常常用的有非对称加密功能的软件，然而，有没有觉得ssh-agent不太好用，有没有觉得要同时管理两者的密钥很麻烦？实际上，GPG提供的gpg-agent提供了对SSH协议的支持，这个功能可以大大简化密钥的管理工作。

使用GPG管理SSH密钥主要有这些好处：

- 备份密钥时只要备份一个GPG密钥即可，不用再单独备份SSH密钥
- 可以使用GPG提供的各种密钥管理功能（如子密钥）
- gpg-agent可以在需要时自动启动（SSH功能的自动启动需要systemd一类的软件支持，不过Arch Linux的官方gpg包已经提供了相关的systemd配置），而ssh-agent需要手动启动
- gpg-agent会自动加载所有被允许用于SSH的密钥，而ssh-agent必须手动使用ssh-add添加密钥，而且每次启动都要手动添加
- gpg-agent在密钥被使用时才会询问密码，而且可以设置缓存密码的时间，而ssh-agent在添加密钥时询问密码，只能设置添加的密钥的有效时间，失效后又要手动添加

虽然也可以通过一定的系统配置来获得一部分以上说到的好处，但是有现成的软件提供了这样的功能，显然要方便很多。

那么进入正题。首先要启用gpg-agent的SSH功能，这可以通过它的`--enable-ssh-suppport`选项来实现。一般gpg-agent都是自动启动的，只要把`enable-ssh-support`加入`~/.gnupg/gpg-agent.conf`即可。然后设置环境变量`export SSH_AUTH_SOCK="${XDG_RUNTIME_DIR}/gnupg/S.gpg-agent.ssh"`，这样SSH客户端就会去连接gpg-agent提供的socket了，可以把脚本放入`/etc/profile`来让系统自动设置环境变量。这样就完成了ssh-agent到gpg-agent的替换。

接下来就需要用于SSH连接的密钥了，这里有两种方法来向gpg-agent提供SSH密钥。

## 使用现有的SSH密钥

如果你有现有的SSH密钥而且不想更换，那么可以让gpg-agent使用现有密钥。

完成之前的配置后，运行`ssh-add SSH密钥文件的路径`首先输入现有密钥的密码来解密密钥，然后gpg-agent就会把这个SSH密钥转换成一个特殊GPG密钥保存到`~/.gnupg/private-keys-v1.d`，并在`~/.gnupg/sshcontrol`中添加相关信息，gpg-agent会提示你输入一个新的密码来加密这个GPG密钥。这个GPG密钥的实际上和原来的SSH密钥是一样的，只是格式不同。

之后在进行SSH连接时gpg-agent就会自动使用转换过来的GPG密钥，并询问新设置的密码，就算系统重启也不用重新添加了。

但是这么做有一个问题：添加的密钥只能通过`ssh-add`管理，普通的gpg密钥管理命令（如`--list-keys`）是不会识别这个特殊的GPG密钥的，所以最好还是使用第二种方法。

## 使用GPG密钥

GPG密钥也可以直接用于SSH连接，但这个密钥必须在创建时启用了认证功能。因为一般都不会创建认证用子密钥，所以这里也给出相关步骤。（什么，不知道主密钥和子密钥是啥？建议先看看[这篇Debian的wiki](https://wiki.debian.org/Subkeys)）

假设已经有一个主密钥了，那么运行`gpg --expert --edit-key 密钥的ID`，输入密钥的密码后就可以进入密钥的编辑环境。接着输入`addkey`来添加一个子密钥，首先会询问所需的密钥类型，选择一个你想要的算法（但必须是`(set your own capabilities)`结尾的），然后就会询问密钥的功能，根据提示，禁用签名（Sign）和加密（Encrypt）功能，并启用认证（Authenticate）功能，接下来就会根据选择的密钥算法的不同询问各种参数，全部完成后输入`save`保存密钥。

启动密钥编辑环境后的大致流程时这样的：

```
gpg> addkey（开始添加子密钥）
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 8（此处选择7/8/11均可，这里以RSA密钥为例）

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt （默认启用签名和加密功能）

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s（禁用签名功能）

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt（现在仅启用加密功能）

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e（禁用加密功能）

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions:（现在已禁用所有功能）

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? a（启用认证功能）

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Authenticate（现在仅启用认证功能）

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q（结束功能设置）
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)（这里根据不同的密钥算法会询问不同的参数，因为之前选的是RSA密钥，所以询问了RSA密钥大小）
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)（设置密钥过期时间，没有特殊需求用默认的不过期即可）
Key does not expire at all
Is this correct? (y/N) y（输入y来确认操作）
Really create? (y/N) y（再输入一次y来确认操作）

gpg> save（保存密钥）
```

然后需要把这个密钥添加到`~/.gnupg/sshcontrol`中，首先运行`gpg --list-keys --with-keygrip`，找到之前添加的子密钥，并复制对应的keygrip。

例如我运行上述命令的输出是这样的（这只是个例子，别把我的keygrip复制去:weary:）：

```
$ gpg --list-keys --with-keygrip
/home/ddosolitary/.gnupg/pubring.kbx
------------------------------------
pub   ed25519 2017-09-02 [SC]
      688E1D093C3638F588890D4450268311C7AD3F62（这里是主密钥的ID）
      Keygrip = BE8993494A433BD1C9B5B3DBF24C49E6C0093A2B
uid           [ultimate] DDoSolitary <DDoSolitary@gmail.com>
sub   cv25519 2017-09-02 [E]
      Keygrip = A032B25B82D024E94F307CC512269164F3147581
sub   ed25519 2017-09-06 [A]（A就表示这个子密钥启用的认证功能）
      Keygrip = 4E50133A8DDFBA70FC6B8FC6F5A4F63E278E3512（所以我们需要的子密钥的keygrip就是这个）

```

然后把这个keygrip添加到`~/.gnupg/sshcontrol`，gpg-agent就会自动使用这个密钥了。

最后，运行`ssh-add -L`，就可以获得对应的SSH公钥，添加到GitHub，BitBucket或服务器的`~/.ssh/authorized_keys`了。

这样就可以直接使用GPG密钥进行SSH连接，不再需要单独管理SSH密钥了。
