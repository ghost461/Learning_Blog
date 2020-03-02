# Git 操作相关
## 初始化目录->指定远程仓库->clone
git init
git remote add -f origin <url>
git clone <url>
## 正常流程
git status
git add <filename>
git commit -m "<information>"
git push
## 关于http proxy
git config --global http.proxy http://ip:port
git config --global https.proxy https://ip:port
git config --global --unset http.proxy
git config --global --unset https.proxy
## 只克隆部分文件
git config core.sparsecheckout true
echo “path/to/file” >> .git/info/sparse-checkout
git pull origin master
