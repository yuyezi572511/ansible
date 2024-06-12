#                            Ansible自动化运维项目

## 1、项目简介

1.1项目说明：使用Ansible编写一个Playbook，自动化配置多个服务器的系统参数（nginx 安装，ssh配置，用户管理，重要文件权限设置....）。配置一个合规性检查机制，确保所有服务器都符合预定义的安全标准，通过Ansible的报告功能生成合规性检查报告，并提供修复不合规项的Playbook。

## 2、环境准备

2.1系统：centos7.9

2.2基础拓扑：准备三台主机，一台作为控制节点安装ansible工具，另外两台作为被管理节点不需要安装ansible.

| 主机名          | IP地址          | 节点类型   | 系统版本  |
| --------------- | --------------- | ---------- | --------- |
| 192.168.100.142 | 192.168.100.143 | 控制节点   | centos7.9 |
| 192.168.100.135 | 192.168.100.135 | 被管理节点 | centos7.9 |
| 192.168.100.136 | 192.168.100.136 | 被管理节点 | centos7.9 |

2.3系统centos7.9可能需要更新yum源（如下载速度慢，安装出现报错提示）：

yum提示1:Another app is currently holding the yum lock； waiting for it to exit...

解决办法：删除yum进程

```shell
rm -f /var/run/yum.pid 
```

yum提示2：Loading mirror speeds from cached hostfile....:

解决办法：更新yum源(注意：三台主机都得设置)改进：使用shell脚本

```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

```shell
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

```shell
yum clean all#清除所有
```

```shell
yum makecache #重新建立源数据
```

2.4安装ansible:

```shell
1.yum install epel-release -y
2.yum install ansible  -y
```

2.5设置主机清单

```
1.cd /etc/ansible

2.vim /etc/ansible/hosts
192.168.100.142
192.168.100.135
192.168.100.136
```

2.6使用密钥免密登陆

```shell
ssh-keygen -t rsa  -b 4096
```

```
ssh-copy-id root@192.168.100.142
ssh-copy-id root@192.168.100.135
ssh-copy-id root@192.168.100.136
```

```shell
ansible web -m ping #检测
```

## 3、项目步骤

### 步骤一：使用Ansible编写一个Playbook，自动化配置多个服务器的系统参数（nginx 安装，ssh配置，用户管理，重要文件权限设置....）根据需求可自行添加。编写 server.yaml.

一.nginx  server安装和配置：

1.安装pcre库支持：

 yum:  name=pcre-devel,pcre,zlib-devel state=installed

2.下载nginx包：

wget http://nginx.org/download/nginx-1.12.0.tar.gz;

3.预编译和编译再正常安装：

./configure --prefix=/usr/local/nginx;make;make install

注意：

cd /tmp;rm -rf nginx-1.12.0.tar.gz;如果已有nginx包会报错。

4.用户管理中：使用items多个值创建多个用户，通过{{}}定义列表变量，with_items选项传入变量的值

5.ssh中设置允许root登陆，在实际生产中root不允许登陆但在测试环境设为yes,设置禁止密码登陆，使用密钥登陆：ssh-keyggen

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name:  Pcre-devel and Zlib LIB Install.
      yum:  name=pcre-devel,pcre,zlib-devel state=installed
    - name:  Nginx WEB  Server Install Process.
      shell: cd /tmp;rm -rf nginx-1.12.0.tar.gz;wget http://nginx.org/download/nginx-1.12.0.tar.gz;tar xzf nginx-1.12.0.tar.gz;cd nginx-1.12.0;./configure --prefix=/usr/local/nginx;make;make install
    - name: permit root login
      lineinfile:  
        path: /etc/ssh/sshd_config  
        regexp: '^PermitRootLogin'  
        line: 'PermitRootLogin yes'
    - name: Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
      notify: Restart SSH  
    - name: Ensure sudo group exists 
      group:  
       name: sudo  
       state: present 
    - name: Linux system Add User list to sudo group.  
      user:  
       name: "{{ item }}"  
       state: present  
       groups: sudo  
       append: yes   
      with_items:  
        - user1  
        - user2  
        - user3  
        - user4
    - name: Set permissions on /etc/ssh/sshd_config
      file:
        path: /etc/ssh/sshd_config
        owner: root
        group: root
        mode: '0644'
    - name: Set permissions on /root/.ssh
      file:
        path: /root/.ssh
        owner: root
        group: root
        mode: '0700'
    - name: Set perminssions on /root/.ssh/authorized_keys
      file:
        path: /root/.ssh/authorized_keys
        owner: root
        group: root
        mode: '0600'
  handlers:  
    - name: Restart SSH  
      service:  
        name: sshd  
        state: restarted
   
      
```

![image-20240611201724020](C:\Users\86178\AppData\Roaming\Typora\typora-user-images\image-20240611201724020.png)

![image-20240611201801716](C:\Users\86178\AppData\Roaming\Typora\typora-user-images\image-20240611201801716.png)



### 步骤二：配置一个合规性检查机制，确保所有服务器都符合预定义的安全标准：检测远程主机Nginx目录是否存在，不存在则安装Nginx WEB服务，安装完并启动Nginx。启动nignx进程 ：/usr/local/nginx/sbin/nginx  成功运行就说明符合安装标准，所以只要判断/usr/local/nginx/目录是否存在就行，如果不存在那就为真,通过调用notify就安装nginx ,确保ssh预配置成功则重启ssh,判断用户组是否存在,存在则创建用户。对每个关键步骤后都添加检查机制，确定是否成功配置。编写server_check.yaml

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name:  Nginx server Install 
      file: path=/usr/local/nginx/ state=directory
      notify:
         - nginx install Nihao
         - nginx start
         
    - name: Check if Nginx server is installed
      stat:
        path: /usr/local/nginx/
      register: nginx_installed
      
    - name: permit root login
      lineinfile:  
        path: /etc/ssh/sshd_config  
        regexp: '^PermitRootLogin'  
        line: 'PermitRootLogin yes'
        
    - name: Check if PermitRootLogin is set to 'yes'
      shell: grep '^PermitRootLogin yes' /etc/ssh/sshd_config
      register: permit_root_login


    - name: Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
      notify: Restart SSH  
      
    - name: Check if PasswordAuthentication is disabled
      shell: grep '^PasswordAuthentication no' /etc/ssh/sshd_config
      register: password_auth_disabled

    - name: Ensure sudo group exists 
      group:  
       name: sudo  
       state: present 
       
    - name: Check if sudo group exists
      shell: getent group sudo
      register: sudo_group
      
    - name: Linux system Add User list to sudo group.  
      user:  
       name: "{{ item }}"  
       state: present  
       groups: sudo  
       append: yes   
      with_items:  
        - user1  
        - user2  
        - user3  
        - user4
        
    - name: Check if users are added to sudo group
      shell: getent group sudo
      register: sudo_group_info
      failed_when: "'user1' not in sudo_group_info.stdout and 'user2' not in sudo_group_info.stdout and 'user3' not in sudo_group_info.stdout and 'user4' not in sudo_group_info.stdout"
      
    - name: Set permissions on /etc/ssh/sshd_config
      file:
        path: /etc/ssh/sshd_config
        owner: root
        group: root
        mode: '0644'  
    - name: Set permissions on /root/.ssh
      file:
        path: /root/.ssh
        owner: root
        group: root
        mode: '0700'
    - name: Set perminssions on /root/.ssh/authorized_keys
      file:
        path: /root/.ssh/authorized_keys
        owner: root
        group: root
        mode: '0600'

    - name: Check permissions on /etc/ssh/sshd_config
      stat:
       path: /etc/ssh/sshd_config
      register: sshd_config_permissions

    - name: Check permissions on /root/.ssh
      stat:
       path: /root/.ssh
      register: ssh_dir_permissions

    - name: Check permissions on /root/.ssh/authorized_keys
      stat:
       path: /root/.ssh/authorized_keys
      register: authorized_keys_permissions
  handlers:  
    - name: Restart SSH  
      service:  
        name: sshd  
        state: restarted
    - name: nginx install Nihao
      shell: cd /tmp;rm -rf nginx-1.12.0.tar.gz;wget http://nginx.org/download/nginx-1.12.0.tar.gz;tar xzf nginx-1.12.0.tar.gz;cd nginx-1.12.0;./configure --prefix=/usr/local/nginx;make;make install
    - name: nginx start
      shell: /usr/local/nginx/sbin/nginx
      
```

![image-20240611202656005](C:\Users\86178\AppData\Roaming\Typora\typora-user-images\image-20240611202656005.png)

![image-20240611202814404](C:\Users\86178\AppData\Roaming\Typora\typora-user-images\image-20240611202814404.png)

若需要访问nginx界面，需要关闭访问墙：

```shell
service  firewalld  stop
```

![image-20240611234516297](C:\Users\86178\AppData\Roaming\Typora\typora-user-images\image-20240611234516297.png)



### 步骤三：通过Ansible的报告功能生成合规性检查报告，并提供修复不合规项的Playbook。

```shell
ansible-playbook  playbook.yml -vvv 
```

运行后这些输出是 检查Ansible 执行的 playbook ，它详细描述了 Ansible 如何与远程主机进行交互，并提供i报错点。

修复不合规项的Playbook:运行ansible-playbook可能会出现报错，需要提供修复的playbook。如控制节点版本不一致，如wget commnd not found,无法使用make编译可能导致无法安装。也可能会出现 ./configure:error: C compiler cc is not find 这个报错，如果出现这个报错， 它的意思就是说无法找到 c 编译器，所以就需要通过 yum install gcc Ubuntu apt-get install build -essential( 这个是乌班图的安装方式 ) 还有一种可能 ssl 也可能会报错 可以通过安装 yum install openssl-devel  ，当.configure 预编译成功后，执行 make 命令进行编译。根据需要添加一些依赖包（注意：已有则会报错）编写server_re.yaml

```yaml
- hosts: web
  remote_user: root
  tasks:
    - name:  Pcre-devel and Zlib LIB Install.
      yum:  name=make,wget,cre-devel,pcre,zlib-devel state=installed
    - name:  Nginx WEB  Server Install Process.
      shell: cd /tmp;rm -rf nginx-1.12.0.tar.gz;wget http://nginx.org/download/nginx-1.12.0.tar.gz;tar xzf nginx-1.12.0.tar.gz;cd nginx-1.12.0;./configure --prefix=/usr/local/nginx;make;make install
   
```


