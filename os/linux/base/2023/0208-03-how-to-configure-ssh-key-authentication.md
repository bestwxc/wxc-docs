# 怎么配置ssh key 公钥登录
1. 从A使用ssh连接目标机器B
2. 在A上生成（或放置已有的key）ssh的密钥，密钥生成位置 ~/.ssh/
3. 将A机器~/.ssh/id_rsa.pub中的内容放到B机器上指定用户的~/.ssh/authorized_keys中（追加），并确保权限必须为600
4. 在A机器上使用 ssh 用户@目标机器登录
