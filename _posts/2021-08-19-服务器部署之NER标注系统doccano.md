

## 服务器部署之NER标注平台doccano

#### 背景

从事NLP工作的小伙伴或多或少会接触数据标注，这里要提到老牌的brat[<sup>1</sup>](#refer-anchor)标注平台，只需一个简单的配置文件和一个存储标注数据的文件夹即可完成搭建。由于brat标注平台出现的比较早，网络上有大量的文章可以参考[<sup>2</sup>](#refer-anchor)。

最近在阅读NER商用化经验时，无意发现了一个开源的标注平台doccano[<sup>3</sup>](#refer-anchor)，UI很酷，功能也比较丰富，重点是持续更新中。最近正好有一个NER标注的任务，借此机会，我们开始进行~~非常规网络环境下~~的标注系统搭建。中间会夹杂一些pip冷知识和Linux服务器连网攻略。

#### 介绍

doccano是一个开源的文本标注工具，目前支持文本分类、序列标注以及seq2seq，支持自定义标签，用于情感分析，NER，机器翻译，文本摘要等任务，全程傻瓜式操作。可以试玩一下[demo](http://doccano.herokuapp.com/)。

安装方式包含pip docker和开发者模式，本次采用最简单的pip方式安装。

> ### pip installation
>
> To install doccano, simply run:
>
> ```
> pip install doccano
> ```
>
> After installation, simply run the following command:
>
> ```
> # Initialize database.
> doccano init
> # Create a super user.
> doccano createuser --username admin --password pass
> # Start the webserver.
> doccano webserver --port 8000
> ```
>
> And in another terminal, run the following command:
>
> ```
> # Start the task queue.
> doccano task
> ```
>
> Go to http://0.0.0.0:8000/.

#### 实践

pip安装，首先要把网络环境和pip虚拟环境配置好，网络正常或者已经熟悉的小伙伴可以略过此步骤。

##### SSH 端口转发[<sup>4</sup>](#refer-anchor)

SSH一共提供了 3 种端口转发，分别是`本地转发（-L参数）`、`远程转发（-R参数）`和`动态转发（-D参数）`。

**远程端口转发是将发送到远程端口的请求，转发到目标端口**。

这样，就可以通过访问远程端口，来访问目标端口的服务。使用**-R**属性，就可以指定需要转发的端口，语法是这样的:

```shell
-R 远程网卡地址:远程端口:目标地址:目标端口
ssh -NvR ${yourport}:109.105.x.x:8080 ${username}@${ip}
```

这里省略了远程网卡地址，但其实是把`${yourport}`的请求转发到代理服务器上。

##### Anaconda 虚拟环境搭建

本例子采用anaconda3，上海交大的镜像比较快，端口需要自己改一下。

```properties
default_channels:
  - https://anaconda.mirrors.sjtug.sjtu.edu.cn/pkgs/r
  - https://anaconda.mirrors.sjtug.sjtu.edu.cn/pkgs/main
custom_channels:
  conda-forge: https://anaconda.mirrors.sjtug.sjtu.edu.cn/cloud/
  pytorch: https://anaconda.mirrors.sjtug.sjtu.edu.cn/cloud/
channels:
  - defaults

show_channel_urls: true
ssl_verify: false

proxy_servers:
  http: http://127.0.0.1:${yourport}
  https: https://127.0.0.1:${yourport}
```

也可以指定虚拟环境目录和下载包的缓存目录，主要用于镜像盘不够大的情况，例如SGI。

```properties
envs_dirs:
  - /data/conda/envs
pkgs_dirs:
  - /data/conda/pkgs
```

下面开始创建anaconda虚拟环境，之所以指定版本是因为有些版本的openssl还得自己解决，第一次下载把`--offline`去掉。

```shell
conda create -n tf13 python=3.6.5 pip=20.2.4 --offline
# 刷新环境
source activate
source deactivate
conda activate tf13
# 在tf13环境下安装pip
conda install pip --offline
```

Tips：有些常用命令可以用 `alais`来解决，一起来解放双手呀。

```shell
alias idea='bash /home/${username}/env/idea-IC-173.4674.33/bin/idea.sh &'
alias acc='source activate'
alias refresh='source deactivate'
alias gpu='nvidia-smi'
```

##### pip 环境配置

修改pip配置文件，路径：`~/.pip/pip.conf` 这里使用阿里云镜像，端口需要自己改一下。

```properties
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
timeout = 300000

[install]
trusted-host = mirrors.aliyun.com
proxy = https://127.0.0.1:${yourport}
```

需要注意的是，有的时候`.bashrc`会改环境变量`proxy_http`和`proxy_https`，执行下面两行命令清空环境变量。

```shell
export http_proxy=
export https_proxy=
```

有的时候是在不想配环境了，可以直接指定参数方式来做。

```shell
pip install transformers --proxy 127.0.0.1:${yourport} -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com --default-timeout=300000
```

##### doccano安装

So，我们终于可以开始安装doccano了，先进入虚拟环境。

```shell
acc doccano # source activate doccano 偷懒呀
pip install doccano
```

然后报错了，果然不可能那么顺利。

> Complete output from command python setup.py egg_info:
> Download error on https://pypi.org/simple/setuptools_scm/: [Errno -3] Temporary failure in name resolution -- Some packages may not be found!
> Download error on https://pypi.org/simple/setuptools-scm/: [Errno -3] Temporary failure in name resolution -- Some packages may not be found!
> Couldn't find index page for 'setuptools_scm' (maybe misspelled?)
> Download error on https://pypi.org/simple/: [Errno -3] Temporary failure in name resolution -- Some packages may not be found!

错误信息怎么出现了 `https://pypi.org/simple/`，我明明指定了镜像，所以估计是pip安装时，在setup.py中有一些依赖需要安装，然而setup.py默认用`https://pypi.org/simple/`安装。所以，报错了。

直接打开网站https://github.com/doccano/doccano/blob/master/setup.py，把里面的依赖`required = [...]`都拷出来，保存到`requirements.txt`里面。格式是这样的：

```
apache-libcloud>=3.2.0
colour>=0.1.5
conllu>=4.2.2
```

然后执行命令

```shell
pip install -r requirements.txt
```

再次运行`pip install doccano`，又有错误。什么？错误信息太多找不到位置？你可以搜error附近的。

> ERROR: Could not find a version that satisfies the requirement setuptools_scm
> ERROR: No matching distribution found for setuptools_scm

安装`setuptools-scm`解决。

```shell
pip install setuptools-scm
```

再次安装`doccano`，成功了，部署过程应该没有什么坑。

#### 功能介绍

doccano标注平台UI风格不错，但我们更关注于实力怎么样。

主要需求点：

1. 数据导入导出
2. 标签添加
3. 角色权限管理
4. 能否实现单账户同时标注(懒着建号)，同一条数据多人标注(背靠背方式也是常用的一种标注方式，用于提高标注质量)

##### 导入与导出

支持多文件上传，导入文件格式可以是文本, json, CoNLL(常用于NER)，导出文件为json。唯一不足时多文件，不能自动区分。不过可以通过json格式增加额外的字段来实现这一功能。

##### 标签添加

支持自定义标签颜色，标签对应的值，堪称傻瓜式配置。

##### 角色权限管理

用户角色分为标注人员，审核人员和超级管理员。

##### 额外功能

单一账户实测可以同时标注，多账户可以在项目中设置是否共享标注结果。

除此之外，标注平台还包含一些统计功能和辅助功能。

增加用户可以在 http://127.0.01:8000/admin/ 中添加，或者命令行 `doccano createuser --username user --password pass`

##### 标注平台使用注意事项

1. 如果有预标注标签，下标是不计算空格的，否则标签很容易跑偏。
2. 尽量保证导入数据无BOM utf-8格式。
3. Label的key范围只有`0~9` `a~z`，如果生成文件超过这个范围，会无法导入哟。
4. 附64色标签生成代码，拿走不谢。

```python
def gen_labels():
    labels_list = []
    color_base = ['00', '55', 'AA', 'FF']
    color_list = []
    label_map = {}
    for a,b,c in product(color_base, color_base, color_base):
        color_list.append(f'#{a}{b}{c}')
    id = 1
    with open('./label_description.txt', 'r', encoding='utf-8') as file:
        for line in file.readlines():
            if line:
                block = line.strip('\n').split('\t')
                label_map[block[1]] = f'{block[1]}-{block[3]}'
                labels_list.append({
                    # "id": id,  // 不要使用id字段，id是和界面中的key绑定的，输入非法会无法导入。
                    "text": f'{block[1]}-{block[3]}',
                    "background_color": color_list[id-1],
                    "text_color": "#ffffff",
                    "parent_category": block[0],
                    "description": block[3],
                    "type": block[2],
                })
                id += 1
    print(labels_list)
    with open('E:\Project\Honor\product/touch/labels.json', 'w', encoding='utf-8') as file:
        json.dump(labels_list, file, ensure_ascii=False)
    return label_map
```

---

<div id="refer-anchor"></div>
#### 参考

[1] [Github brat项目](https://github.com/nlplab/brat)

[2] [brat搭建教程](https://blog.csdn.net/u014028063/article/details/8932930)

[3] [Github doccano项目](https://github.com/doccano/doccano)

[4] [SSH端口转发](https://blog.fundebug.com/2017/04/24/ssh-port-forwarding)