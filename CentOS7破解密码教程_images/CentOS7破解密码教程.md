# CentOS 7破解密码教程
1. 启动时任意键暂停启动
   ![NewDocument](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程.png)
2. 按e键进入编辑模式
   ![NewDocument1](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程1.png)
3. 将光标移动linux16开始的行，添加内核参数rd.break
   ![NewDocument2](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程2.png)
4. 按ctrl-x启动
    ![NewDocument3](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程2.png)
5. mount –o remount,rw /sysroot
   ![NewDocument4](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程3.png)
6. chroot /sysroot
   ![NewDocument5](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程5.png)
7. passwd root
   ![NewDocument6](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程7.png)
8. touch /.autorelabel
   ![NewDocument7](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程7.png)
9. exit
   ![NewDocument8](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程8.png)
10. reboot
   ![NewDocument9](C:\Users\Jack\OneDrive\文档\CentOS7破解密码教程_images\CentOS7破解密码教程9.png)
