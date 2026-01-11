解决qtcreator工程文件例程报错error: cannot initialize object parameter of type ‘Ui_ChatDialog’ with an expression of type ‘MainWindow’

在完成用虚拟机linux ubuntu进行交叉编译时候，qtcreator不正常运行

qt下载好并且环境配置完成，kits和qt都已配置完成在qt creator中，在终端手动编译qmake make都完全没问题，但是在qt creator中却报错。
即使是新建工程例程都报错。


error: cannot initialize object parameter of type ‘Ui_ChatDialog’ with an expression of type ‘MainWindow’

如图所示

![](https://gitee.com/R-Poju/Images/raw/master/Qt/Qt_001.png)

**问题分析**
可能因为qt creator4.11.0 based on qt 5.12 版本略微冲突导致



**解决方法**

在qtcreator中选择帮助（help）-关于插件（about plugins）

![](https://gitee.com/R-Poju/Images/raw/master/Qt/Qt_002.png)



将ClangCodeModel取消勾选即可

![](https://gitee.com/R-Poju/Images/raw/master/Qt/Qt_003.png)



取消勾选之后需要重启 QtCreator



重启后，报错就解决了

![](https://gitee.com/R-Poju/Images/raw/master/Qt/Qt_004.png)