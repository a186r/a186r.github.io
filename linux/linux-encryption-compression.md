## Linux下加密压缩与解压

有一些文件需要加密后放在网盘上，Linux下可以使用以下方式：

- #### tar

    加密压缩：

    `tar -zcf - filename |openssl des3 -salt -k password | dd of=filename.des3`

    解压：

    `dd if=filename.des3 |openssl des3 -d -k password | tar zxf -`

    注意：命令最后面的"-" 它将释放所有文件， `-k password` 可以没有，没有时在解压时会提示输入密码

- #### zip

    加密压缩：

    `zip -re filename.zip filename` 回车，输入2次密码

    `zip -rP passwork filename.zip filename passwork` 是要输入的密码

    解压：

    `unzip filename.zip` 按提示输入密码
    
    `unzip -P passwork filename.zip passwork` 是要解压的密码，这个不会有提示输入密码的操作