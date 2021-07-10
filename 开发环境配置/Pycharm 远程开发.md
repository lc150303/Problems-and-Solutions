Author: Scorpior  
Version: 1.0

# Pycharm 远程开发配置
GPU 服务器通常采用 Linux 系统，并且不会配置图形桌面，无法方便地利用 IDE 进行代码开发。Pycharm 专业版的远程开发功能可以很好地满足我们对代码自动补全等需求，还有一个额外的好处是可以通过配置远程解释器方便地切换不同服务器。如果不使用远程开发，我们可能得在每台服务器上都配置 IDE 才能方便地使用它们。

下文中我们把远程 Linux 主机称为**服务器**，我们直接操作的电脑称为**本地**。
## 一. 前置工作
首先

## 二. 新建项目
首先我们在本地 Pycharm 中新建项目 Pure Python，并在右侧第一行配置好本地路径（箭头1）。  
然后在右侧下方配置 python 解释器，选择 Previously configured interpreter （箭头2），然后选择右边3个点配置远程服务器（箭头3）。
![新建项目-配置路径](../imgs/py_dev_env_config_0.png)

进入 interpreter 配置页面如下图，左边栏选择 SSH Interpreter，填好右边服务器与用户名后点击右下 Next
![新建项目-登录服务器](../imgs/py_dev_env_config_1.png)

在弹出的认证界面点 Yes，然后在下一个页面填好登录密码，就来到真正选择解释器的界面。我们在这里选择要用的 python 解释器的路径，如果装了 miniconda 并新建了环境（比如名为 abc 的环境），那么路径应该为 `~/miniconda3/envs/abc/bin/python`。这里 `bin` 之前的路径按实际情况更改，最终要选 `python` 而不是 `python3`。
![新建项目-配置解释器](../imgs/py_dev_env_config_2.png)

选好后点 Finish，就回到第一张图，继续配置 Remote project location （箭头4）后即可点击右下角 Create。