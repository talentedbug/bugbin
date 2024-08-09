---
title: 低成本搭建在线 IDE
date: 2024-08-01 19:32:25
---

我省 JSOI 紧跟时代浪潮，在去年的 CSP-J/S2 中采用了最新的 NOI Linux 2.0，并且贴心地为我们初始化了 VS Code 的 C/C++ 插件，极大地提升了考生的编码体验。~~我大 CCF 牛逼！我爱 CCF！~~

不过我学校里的电脑显然没有跟上时代浪潮。老师可能是懒得更新，仍然使用传统的 Windows 7 加上 Dev-C++ 的环境，不仅体验糟糕，而且现在也不再符合比赛环境，有潜在的兼容性问题。正好手边有一部闲置的香橙派，想到了在它上面跑一个在线 IDE，然后在学校远程使用，岂不美哉？

## 1. 为什么

好吧不开玩笑了，下面认真地列一下需求以及对应的解决方案。

1. VS Code 早已结束了对 Windows 7 的支持，并且每次安装非常麻烦，需要一贯、舒适、便利且（最重要的是）符合比赛环境的工具链；
2. 有些写好的轮子、文档在电脑上，总是忘记拷到 U 盘上，但是云盘难以在没有手机的情况下登录，需要方便地获取；
3. 从购买硬件到运行成本都要尽量降低，最起码比各类云主机或者在线 IDE 要低。

对于这几个需求，我的解决方案是：

1. 单板机 Orange Pi Zero 3，以及 10W 或者 15W 充电器、充电线、网线、散热、外壳、SD 卡、读卡器等等，这些都可以在淘宝上搜索购买，大概 200 元搞定；
2. code-server，将 Code OSS 的 Electron 界面移植到浏览器上，作为服务器使用；
3. ChmlFrp，内网穿透，实现外网访问。

下面我将对搭建的步骤和一些细节进行描述。

## 2. 硬件

我选择了 [Orange Pi Zero 3](http://www.orangepi.cn/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-Zero-3.html)，主要是因为它价格便宜，性能勉强能用，并且功耗比较低。我选择了 4 GiB  内存版本，因为我后期还计划搭建其他服务，如果大家只是想要一个在线 IDE，2 GiB 的版本也可以。

硬件上面的坑点还是比较多的。首先需要单独购买电源线和充电器，充电找 10W 或者 15W 都可以，记得量一量长度够不够。最好再买一根网线，有线网络稳定并且初始 ssh 连接方便。

散热建议买一组铝片，效果比较明显，但是风扇没有什么用处，还容易积灰。再买一个外壳，亚克力的，对提升颜值和方便放置都有用。

如果你很不幸的选中了某家不能完全装进去的，记得留下网口、电源那一面，没有壳反而比较方便。

还要再买一张 SD 卡，建议 16 GiB 以上、32 GiB 最佳，不要贪便宜，一定选择正规厂商，才能保证长期运行的稳定性以及读写速度。读卡器那是必须的。

这部单板机最大的问题是没有电源键（？！），只要 ssh 挂了，唯一的解决方法就是硬关机（指拔电源线）。

## 3. 搭建服务器

关于硬件部分就不必多说了~~，不就是插上网线和电源线嘛~~。

软件部分，将 SD 卡用读卡器接在电脑上，如果是 Windows，会提示你格式化，这里不需要。我们前往 [Armbian](https://www.armbian.com/orange-pi-zero-3/) 下载，注意选择 “Server and IOT images with Armbian Linux v6.6”部分的镜像（名称是链接），这个版本没有图形界面，可以节省内存和 CPU。

下面再下载 [Balena Etcher](https://etcher.balena.io/)，如果是 Windows 还可以考虑 [Rufus](https://rufus.ie)，`dd` 我测试了好几次，貌似完全没法用，所以我们还是用以上两个吧。这里我们直接默认设置刷写，完成后会发现分出了一个大约 1 GiB 的小分区，其他的都是空闲，这里千万不要手贱创建分区，因为开机之后会自动进行分区扩容。

单板机的 SD 卡槽（奇怪地）在背面，SD 卡有字的一面朝外插入，这时候再先插入网线，再插入电源线，红灯常亮表示系统正在 uboot，稍等 2 到 3 分钟，黄灯常亮，红灯心跳闪烁，这个时候表示 Armbian 已经引导成功，正常启动。如果插电之后发现一个灯也不亮，~~先查查 SD 卡插没插~~拔掉电源重试一次。

下面最大的挑战是找到 IP。某些路由器，比如小米、华为等，会使用 Wi-Fi 设备的特殊 IP 段（192.168.31.x 之类），因为我们是有线连接，理论上只有少数几个设备会使用有线 IP 段 192.168.1.x，可以从 192.168.1.2 开始试试。如果找不到，登录路由器后台找找；路由器后台登不了，用 nmap 扫一扫常见 IP 段。以上可以轻松百度找到教程。

总之现在你找到 IP 地址了。Armbian 默认开机启用 sshd 服务和 root 登录，我们直接使用 `ssh root@192.168.1.x` 连接。如果你之前搞废了一次，这里会提醒你主机指纹不一致，我们只需要在 ~/.ssh/known_hosts 里删除对应条目，重新记录就可以了。

连接密码默认是 `1234`。首次连接需要你设置用户名、时区之类的信息，这里有几个坑：

1. 这个过程没有返回的选项！看清楚再选；
2. 输入密码有星号提示，但是 ssh 连接和后续 sudo 没有；
3. 一不小心把提示词退格掉了？继续输入就好。

~~我才不会告诉你这些我都干过呢。~~

因为我们使用 ssh 连接，终端显示的问题是不存在的，所以既可以选择 en-US 也可以选择 zh-CN。

完成了初始设置，我们就会得到一个 root 终端。因为 code-server 运行不使用 root 账户，我们要退出并且使用我们刚刚创建的用户登录。假设你刚刚创建了名为 `bug` 的账户，主机 IP 是 `192.168.1.5`，那么使用 `ssh bug@192.168.1.5` 再次登录。

接下来，对于一些命令我将不会做像上面这么详细的解释了，因为接下来就是一个普通 Linux 环境，需要你有一定的经验。

## 4. 系统准备

首先换源。由于 Armbian 使用 Debian 主仓库中 multiarch 的软件包，和少量为 arm64 打包的 Armbian 仓库，因此我们要换两次源。

> 以下命令以“$”开头表示你应当在普通账号下输入，以 # 开头表示应当在 root 账号下输入，没有则为输出或者在文本编辑器中输入。

```bash
$ cd /etc/apt
$ sudo mv sources.list sources.list.bak
$ touch sources.list
$ nano sources.list

# nano 界面中输入

deb https://mirrors.bfsu.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb https://mirrors.bfsu.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb https://mirrors.bfsu.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
deb https://mirrors.bfsu.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware

# Ctrl-W，y，回车，退出

$ cd sources.list.d
$ sudo nano armbian.list

# nano 界面中，将网址部分替换为以下

https://mirrors.bfsu.edu.cn/armbian

# 由于我写下这行文字时，为 Orange Pi Zero 3 编译的 Armbian 仍为测试版本，直接使用镜像网站的替换命令是无效的，必须这样替换
```

最后刷新缓存并更新。

```bash
$ sudo apt update && sudo apt upgrade
$ sudo reboot
```

稍候 2 到 3 分钟，再次使用 ssh 连接，登录。

安装工具链。

```bash
$ sudo apt install gcc g++ gdb
```

## 5. 安装 code-server

好的那么经过准备，我们可以开始安装 code-server 了！

[code-server](https://github.com/coder/code-server) 并不是微软的官方项目。VS Code 有一个隧道模式，也可以实现访问网页版本的 VS Code，但是因为它貌似只支持用官方的 vscode.dev 隧道且自定义能力奇差，前者在国内访问速度很慢，我们使用它并不方便。

Coder 开发了这个完整的网页版 Code，使用 Code OSS 即 VS Code 的完全开源版本作为基础，因为它是以 Electron 为开发框架的，而 Electron 本质上就是个 Chromium 内核，所以这个移植并不复杂。目前 code-server 处于非常迅速和积极的开发中，所以我们将锚定一个版本，毕竟谁都不想像 Arch Linux 一样天天更新对吧……

根据官方指南，安装是非常容易的：

```bash
curl -fsSL https://code-server.dev/install.sh | sh
```

这个命令不要用 sudo 或者 root 账户执行。code-server 并不支持 root 运行，并且用 root 执行暴露到网络的服务是非常危险的。我们这里会使用它的原生版本而不是 Docker 版本，因为以香橙派的性能，运行 Docker 的虚拟环境有点吃紧。

安装完成，程序会给出一些提示，我们首先需要修改一个最大的槽点。

```bash
nano ~/.config/code-server/config.yaml
```

这个文件现在应该是这样的：

```
bind-addr: 127.0.0.1:8080
auth: password
password: [SOME_PASSWORD]
cert: false
```

首先将 `bind-addr` 修改为 `0.0.0.0:8080`。因为在远程机器上，127.0.0.1 并没有绑定到香橙派的 IP 上，这是远程服务器上 hosts 中的特定绑定。而 0.0.0.0 是全局的，在任何机器上都指向服务器本身。

然后，由于这个文件在我们的家目录里，任何访问你的账号的人都可以直接看到密码，非常危险，我们可以使用 Argon2 进行散列化。我们使用一个[在线小工具](https://argon2.online/)。在 Plain Text Input 中填入你的密码，点击 Salt 右侧齿轮随机生成密钥，下面几个数字可以调大一些，亦可保持不动，最后点击 GENERATE HASH，复制 Output in Encoded Form 一栏内的内容。

在文件内，将 `password: ...` 一行替换为 `hashed-password: [THE_STRING_YOU_GOT]`。

然后，我们可以在局域网内访问 `[OPI_IP]:8080` 来查看。如果出现了 code-server 的密码界面，并且提示 Check the config file at /home/talentedbug/.config/code-server/config.yaml for the password. 则设置成功。

这里注意如果你的机器上设置了防火墙，以 firewalld 为例，需要开启端口：

```bash
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
```

到这里为止，我们 code-server 的搭建就结束了，接下来我们搭建内网穿透。

## 6. ChmlFrp 内网穿透

首先访问 [ChmlFrp](https://chmlfrp.cn) 并且直接点击进入管理面板，并且注册账号、实名验证。这些步骤非常简单，不细说了。

不过登录的时候有个小坑：ChmlFrp 对密码复杂度有要求，因此（顺理成章地）在登录界面上，如果输入的密码不符合复杂度要求，根本不会有任何提示，点击“登录”没有反应，不要以为坏了，再想想密码到底是啥。

注册完成后，我们在“隧道管理”——“隧道列表”里选择“添加隧道”，筛选“可建站”和“中国”节点，随意选择一个，随便填一个隧道名，然后在“内网端口”处填入 8080，外网端口选择一个你容易记住的端口号，其他保持默认或者留空即可。

待页面上提示成功之后，我们再前往软件下载，选择 “Linux”——arm64 的链接并复制，在 ssh 终端上：

```bash
wget [THE_LINK]
tar -xvf [FILE_NAME]
mv [DIR_NAME] ChmlFrp
sudo mv ChmlFrp /opt
```

最后一条命令是为了将客户端移动到一个全局的位置。如果你不需要或者没有权限，也可以就放在这儿，接下来将所有 `/opt/ChmlFrp` 改为 `/home/[USER_NAME]/ChmlFrp` 就行。

然后我们在网页上再找到“配置文件”，选择对应的节点并复制，粘贴到软件目录内的 `frpc.ini` 文件里，原有的内容需要清空。

最后，我们再添加一个 Systemd 单元文件，方便开机启动以及管理。

```bash
sudo touch /etc/systemd/system/chmlfrp.service
sudo nano /etc/systemd/system/chmlfrp.service

# In nano
[Unit]
Description=ChmlFrp for code-server
After=network.target

[Service]
Type=simple
ExecStart=/opt/ChmlFrp/frpc -c /opt/ChmlFrp/frpc.ini
WorkingDirectory=/opt/ChmlFrp

[Install]
WantedBy=multi-user.target
# Save and exit

sudo systemctl daemon-reload
sudo systemctl enable --now chmlfrp.service
```

这时候我们已经启动了内网穿透，通过 `sudo systemctl status chmlfrp.service` 我们可以看到输出：

```
映射启动成功，感谢您使用 ChmlFrp！
```

证明我们的内网穿透运行正常。

当然大部分教程给出的方案是用 `screen` 或者 `tmux` 将进程单独放在一个开启的虚拟终端中。我一直觉得这个方案非常不优雅，更倾向于使用 Systemd 单元文件，纳入系统的管理中。

刷新一下 ChmlFrp 的网页，我们应该已经看到隧道旁边的绿灯已经亮了，访问连接地址，我们就能看到刚刚在内网中看到的密码界面，但是此时我们已经实现了无论你在哪里，在多么古旧的机房里，都能访问到一个完整的 VS Code 环境。恭喜！

## 7. 小的调整

首先说说怎么升级。官方给出的方法是直接用安装命令覆盖，配置文件可以保留，我觉得没问题，但是在安装之前我们还是应该检查一下哪些文件会被覆盖，以防我们的修改不巧碰到了更新的文件。

其次，作为光荣的 OIer 我们需要 C/C++ 插件，而这个插件很不巧属于微软，因此在用于构建 code-server 的 Code OSS 中是没有的。我们一方面可以从 [VS Code Marketplace](https://marketplace.visualstudio.com/VSCode) 下载并安装，也可以在 `/usr/lib/code-server/lib/vscode/product.json` 里添加软件源，后者更加方便。

```json
"extensionsGallery": {
  "serviceUrl": "https://marketplace.visualstudio.com/_apis/public/gallery",
  "cacheUrl": "https://vscode.blob.core.windows.net/gallery/index",
  "itemUrl": "https://marketplace.visualstudio.com/items",
  "controlUrl": "",
  "recommendationsUrl": ""
}
```

这一段添加到上述文件中最后一个大括号内，并且在上一行需要添加逗号，否则 code-server 无法启动。

## 8. 碎碎念

这篇无比详细的文章终于完成了，而此时我已经用 code-server 大概 2 个星期了。用下来的体验是很好的，除了由于编译时默认加入调试参数，导致非常慢之外，还是很好的。并且实际上它还可以作为远程 ssh web，因为 VS Code 自带内置终端。

回头我去看了下 ChmlFrp 这个内网穿透，发现它基本属于公益性质，虽然有付费节点，但是即使是免费节点也是不限总流量，瞬时流量也完全够用，速度也是相当快。ChmlFrp 特别提到了它对于学生用户的友好，但是根据我参加其他这类公益服务的印象，很少有人会付钱去支持。所以如果大家用得高兴，有能力的话给 ChmlFrp 捐款或者购买 VIP 吧。
