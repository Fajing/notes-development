“curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused”

最近再次遇到该问题，详细了解了下，发现是 github 的一些域名的 DNS 解析被污染，导致DNS 解析过程无法通过域名取得正确的IP地址。可以通过修改/etc/hosts文件可解决该问题。

具体而言：

打开 https://www.ipaddress.com/ 输入访问不了的域名，获得对应的IP。

使用vim /etc/hosts命令打开不能访问的机器的hosts文件，添加如下内容：

199.232.68.133 raw.githubusercontent.com
199.232.68.133 user-images.githubusercontent.com
199.232.68.133 avatars2.githubusercontent.com
199.232.68.133 avatars1.githubusercontent.com
1
2
3
4
注：上面内容中199.232.68.133是raw.githubusercontent.com所在的服务器IP（通过 https://www.ipaddress.com/ 获知）。


可以使用国内源啦  再也不痛苦啦

/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"

