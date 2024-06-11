cuda版本：11.4

Python版本：3.7

由于网络问题，官网提供的指定pytorch和nvidai频道方法会卡住，proxychains->clash代理也没起作用。需要更换国内源（此前配置的源似乎也不起效）。最后命令：

`conda install pytorch torchvision torchaudio cudatoolkit=11.3 -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/linux-64/`

然而发现torch中cuda不可用…

查询后发现安装了cpu版本的pytorch

![image-20240527000110734](C:\Users\xiaoti\AppData\Roaming\Typora\typora-user-images\image-20240527000110734.png)

于是重新配置，指定如下命令：

`pip3 install torch==1.11.0+cu113 torchvision==0.12.0+cu113 torchaudio==0.11.0+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html`

由于网速填满，下载的whl文件有1.6G，于是proxychains走了代理