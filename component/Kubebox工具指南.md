# 前言
kubebox是一款用来方便容器维护的工具，用js编写，以nodejs打包，pkg生成exec文件的。

https://github.com/astefanutti/kubebox

# 安装
```
curl -Lo kubebox https://github.com/astefanutti/kubebox/releases/download/v0.5.0/kubebox-linux && chmod +x kubebox
cp  kubebox /usr/bin
```
然后直接运行 kubebox即可。kubebox 默认使用/root/.kube/config文件。具体参考参考github

可以选择ns下的某个pod进行cpu，内存，网络查看，也可以看日志，打开cli命令（R 按键）
# 自己编译打包
安装nodejs
```
#下载 https://nodejs.org/en/download/ nodejs包
# 我们使用的包为：node-v10.16.2-linux-x64.tar.xz
xz -d node-v10.16.2-linux-x64.tar.xz
tar xvf node-v10.16.2-linux-x64.tar
```
然后把nodejs的包放入path搜索路径中,编辑/etc/profile，增加如下内容:
```
export PATH=$PATH:/info/node-v10.16.2-linux-x64/bin
```
后面我们进行代码的编译：
```
$ git clone https://github.com/astefanutti/kubebox.git
$ cd kubebox
$ npm install
$ node index.js
```

如果想要生成可执行文件，执行如下指令:
```
npm install -g pkg
pkg package.json
```
成功后，可以看到有三个文件生成kubebox-linux，kubebox-macos，kubebox-win.exe
```
# ll
total 133648
-rw-r--r--   1 root root      642 Aug  7 16:15 Dockerfile
drwxr-xr-x   4 root root      235 Aug  7 16:15 docs
-rw-r--r--   1 root root      265 Aug  7 16:28 hello.js
-rwxr-xr-x   1 root root     1283 Aug  7 16:15 index.js
-rwxr-xr-x   1 root root 47333687 Aug  7 19:38 kubebox-linux
-rwxr-xr-x   1 root root 47454023 Aug  7 19:38 kubebox-macos
-rw-r--r--   1 root root 41872013 Aug  7 19:38 kubebox-win.exe
-rw-r--r--   1 root root     1600 Aug  7 16:15 kubernetes.yaml
drwxr-xr-x   5 root root      166 Aug  7 16:15 lib
-rw-r--r--   1 root root     1075 Aug  7 16:15 LICENSE
drwxr-xr-x 354 root root    12288 Aug  7 17:29 node_modules
-rw-r--r--   1 root root     1768 Aug  7 16:15 openshift.yaml
-rw-r--r--   1 root root     2498 Aug  7 16:15 package.json
-rw-r--r--   1 root root   131970 Aug  7 17:29 package-lock.json
-rw-r--r--   1 root root     6848 Aug  7 16:15 README.adoc

```
