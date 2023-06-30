```bash
# 安装rpm包
rpm -ivh xxx.rpm

#卸载rpm包
rpm -e xxx

# 查看安装了哪些rpm包
rpm -qa 

# 查看rpm包中有哪些文件
rpm -qpl xxx.rpm

# 将rpm包中文件还原到当前目录
rpm2cpio xxx.rpm | cpio -idvm 

# 创建打包环境
rpmdev-setuptree
rpmbuild/
|___BUILD
|___BUILDROOT
|___RPMS
|___SOURCES
|___SPECS
|          |___  xxx.spec
|___SEPMS

# .spec文件说明


```