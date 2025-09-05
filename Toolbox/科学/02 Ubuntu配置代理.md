# 【保姆教程】Ubuntu 配置 Clash For Linux 客户端 2025

> CSDN博客都发不了[捂脸]

**系统环境**：ubuntu 22.04 64位
由于需要用**docker拉取镜像**，第一次在`linux` 安装  `clash`，配置代理，踩了不少坑。
特此记录，希望对各位有所帮助。
## 1.下载clash
- 切换到 **root** 权限
- **创建** Clash 文件夹
- **下载安装包**（由于Github Clash 仓库已经删库跑路了，目前只能通过本站下载地址进行 `Wget` 在线下载，以下只支持 `X86_64` 架构的系统使用）
```bash
# 切换超级管理员
su 

# 创建文件夹 
cd && mkdir clashcd clash 

# 本站文件 
wget https://doc.6bc.net/kehuduan/clash/clash-linux-amd64-v1.14.0.gz
```

## 2.可执行命令clash
- 解压Clash 
- 加权限
- 重命名 Clash ，并放在`可执行文件存放目录`，方便以后直接用 `clash` 启动，不用写完整的路径
- 通过查看**版本**确认操作是否成功

```bash
# 解压文件 
gzip -d clash-linux-amd64-v1.14.0.gz 

# 给予权限 
chmod +x clash-linux-amd64-v1.14.0 

# 改名移动 
mv clash-linux-amd64-v1.14.0 /usr/local/bin/clash 

# 查看版本 
clash -v
```
可我打印的是**v1.18.0**，可能原作者写错了名称~
![clash版本](https://i-blog.csdnimg.cn/direct/6184d6e507454b548d3a199f31736209.png)
然后用 `clash` 就能启动了，但是我们还没配**订阅链接**，等会再启动。

## 3.配置config.yaml
原命令是
```bash
# 进入目录 
cd $HOME/.config/clash/ 

# 导入订阅 
wget -O config.yaml 订阅地址 
```
但我用 `wget -O` ，得到的是订阅链接的原始内容（通常是经过编码的代理节点信息），它**不是完整的 Clash 配置文件**。

- 获取**clash配置文件**
所以，我们要把window的clash配置文件复制过来，到`config.yaml`。
像这样的内容
![config.yaml](https://i-blog.csdnimg.cn/direct/c61d4436690146b899cb84d9038ef2ec.png)
- 手动下载 `MMDB` 文件：[https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb](https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb)

	放在 `.config/clash` 文件夹下，当前文件夹结构为
	```
	~/.config/clash# ll
	total 9324
	drwxr-xr-x 2 root root    4096 Sep  3 16:01 ./
	drwx------ 4 root root    4096 Sep  3 15:40 ../
	-rw-r--r-- 1 root root   25405 Sep  3 16:01 config.yaml
	-rw-r--r-- 1 root root 9509719 Sep  3 15:56 Country.mmdb
	```

最后在 `config.yaml` 末尾添加`MMDB` 文件的路径（替换成你的路径）

```json
geoip:
  path: /root/.config/clash/Country.mmdb
```

所以完整的 `config.yaml` 为

```bash
port: 7890
allow-lan: false
mode: Rule
log-level: info

proxies:
  # 订阅解析后的代理节点列表
proxy-groups:
  # 规则组配置
rules:
  # 规则配置
  
geoip:
  path: /root/.config/clash/Country.mmdb
```

## 4.启动服务
- 生成 systemd 配置文件
	```bash
	vim /etc/systemd/system/clash.service
	```
	粘贴如下内容
	```shell
	[Unit] 
	Description=Clash - A rule-based tunnel in Go 
	Documentation=https://github.com/Dreamacro/clash/wiki 
	
	[Service]
	OOMScoreAdjust=-1000 
	ExecStart=/usr/local/bin/clash -f /root/.config/clash/config.yaml 
	Restart=on-failure 
	RestartSec=5 
	
	[Install] 
	WantedBy=multi-user.target
	```

- 然后`启动 clash 服务`
	```bash
	# 配置开机自启 
	systemctl enable clash 
	
	# 启动 clash 服务 
	systemctl start clash 
	```

- 写入环境变量
	```bash
	# 配置环境变量 
	echo -e "export http_proxy=http://127.0.0.1:7890\nexport https_proxy=http://127.0.0.1:7890" >> ~/.bashrc
	source ~/.bashrc
	```
打开`~/.bashrc`文件手动写入两行
```bash
# 配置环境变量 
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```
得到
![在这里插入代码片](https://i-blog.csdnimg.cn/direct/345a5c1903c444ae8924fd0eca92745b.png)
## 5.启动clash
- 直接启动clash
	```bash
	clash # 
	```

- 检查端口监听
	```bash
	ss -tuln | grep 7890
	```
	可以看到端口在监听状态
	![监听状态](https://i-blog.csdnimg.cn/direct/292d6f1e0c9047b5a0e69be2a9b48a5d.png)

使用 `nohup`，这样 `Clash` 进程会在后台运行，输出日志写入 `clash.log`，终端关闭也不会停止。
```bash
nohup clash > clash.log 2>&1 &
```
## 注意事项
1. 修改 config.yaml 文件内容，需重启服务 `systemctl restart clash.service`
2. 查看 clash 日志
	```bash
	journalctl -u clash.service -f
	```
3. 配置`MMDB` 相关内容之前，运行到最后没报错，那么这个可以省略

## 6.配置docker（可选）
创建或修改 Docker 系统服务的代理配置文件；
就是把这`两行代理`在docker这边再写一遍。
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF
```
重新加载 systemd 配置并重启 Docker 服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---
**参考**：https://doc.6bc.net/article/35/

