dracut 在引导过程中有多个挂载hook参考：
https://l8liliang.github.io/2019/11/28/dracut.html
https://mirrors.edge.kernel.org/pub/linux/utils/boot/dracut/dracut.html

```bash
# 该目录下添加模块
/lib/dracut/modules.d/

# 编写模块下脚本
# 模块挂载脚本
module-setup.sh
#模块功能脚本

# 制作initramfs
dracut --add "模块名" xxx.img --force

```