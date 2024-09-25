---
title: Hexo-GitHub-Netlify-Cloudflare个人博客搭建记录
date: 2024-09-25 21:44:30
---

## Written at the Beginning

说是心血来潮，倒也蓄谋已久，心心念念的个人博客终于开了个头。

为什么想写博客呢，一是为了记录自己的学习过程（博主是个超级健忘的人），二是企图通过更新博客的方式push自己提高效率，好好学习（博主是个超级懒的人）。

经过几天的兜兜转转，踩了无数坑后，一个像样的博客总算是搭好了，先截图纪念一下init-page，期待后续功能的完善和补充 ^_^（自我push一下）。

![init-page](init-page.jpeg)



## 博客搭建

本站采用的是Hexo-GitHub-Netlify-Cloudflare的架构。

![博客框架](博客框架.png)

简单来说就是使用Hexo生成博客框架，使用GitHub托管网站静态资源，使用Netlify将托管在GitHub上的静态网站部署上线（在此之前需要申请一个域名），为了解决国内访问速度慢的问题，使用CloudFlare对域名进行CDN加速。

Hexo环境的搭建以及账号注册就不多赘述了，不过由于不支持国内身份验证所以只能发邮件给Netlify官方进行账号注册的心酸只有注册过的人才懂 TwT



## Notebook

### Hexo

#### Hexo目录结构

搭建好Hexo环境并安装好Hexo后，在终端进入自定义的空文件夹作为博客的根目录，执行命令：

```bash
hexo init
```

即可生成基本博客框架。

在根目录下查看目录结构：

```bash
tree -L 1
```

结果如下：

```bash
.
├── _config.landscape.yml
├── _config.yml
├── db.json	# 缓存文件，初始不存在，执行hexo clean即可删除
├── node_modules
├── package-lock.json
├── package.json
├── scaffolds
├── source
└── themes
```

在Finder中效果如下（MacOS）：



各部分的含义：

![Hexo目录结构](Hexo目录结构.png)

- `_config.yml`

  为全局配置文件，网站的很多信息都在这里配置，比如说网站url、名称、副标题、描述、作者、语言、主题等等。具体可以参考官方文档：https://hexo.io/zh-cn/docs/configuration.html。

- `scaffolds`

  骨架文件，是生成新页面或者新博客的模版。可以根据需求编辑，当`hexo`生成新博客的时候，会用这里面的模版进行初始化。

- `source`

  这个文件夹下面存放的是网站的`markdown`源文件，里面有一个`_post`文件夹，所有的`.md`博客文件都会存放在这个文件夹下。此外，还可能有其他文件夹（如`_data`），具体看config文件的配置信息进行调整。

- `themes`

  网站主题目录，`hexo`有非常丰富的主题支持，主题目录会存放在这个目录下面。更多的主题参见：https://hexo.io/themes/

- `package.json`

​	记录了项目的一些基本信息、脚本命令`scripts`（可重命名一些命令及组合）以及依赖项`dependencies`

- `node_modules`

  保存着项目依赖项文件，在package.json配置好后，在终端输入命令

  ```bash
  npm install 
  ```

  即可自动下载安装所需依赖项。

- `_config.landscape.yml`

  此文件为当前项目主题的配置文件，landscape是hexo默认主题的名字，如果有更换主题，此文件一般保存在对应的theme文件夹中并按照不同主题进行个性化配置。

#### Hexo更换主题

网站部署上线之后，又花了几天时间对网站主题进行精挑细选，跌跌撞撞地删了又装终于也是换好了。总结下来，主题安装有几种方式：直接下载压缩包、clone GitHub仓库、使用主题开发团队开发的工具进行一键安装等。具体还是要看对应主题的文档教程。

##### 一个小坑：有些主题的GitHub仓库中带有git相关的文件夹（由git init的时候自动创建），因此clone到本地根目录的`theme`文件夹中成了项目的submodule，然而我们并不需要submodule，需要手动删除git相关的文件夹。

将想要更换的主题（一个文件夹）下载到本地根目录的`theme`文件夹中后，会在该主题文件夹中看见新的`_config.yml`文件（也可能命名为`_config.theme_name.yml`），该配置文件主要配置的就是该主题的具体功能及细节了。

接下来，我们要先修改根目录`_config.yml`文件的`theme`字段为当前主题的名字（如不确定，可在该主题文件夹中的`package.json`文件的`name`字段获取，这样即可保证在根目录下进行`hexo server`时使用的是我们更换的主题。

剩下的就是根据主题教程文档对`_config.yml`文件进行修改了，到这里就真正体会到了Hexo框架的便捷之处。（待我好好琢磨一番......）

#### Hexo上传文章

```bash
hexo new post "test" # 会在 source/_posts/ 目录下生成文件 ‘test.md’，打开编辑
hexo generate        # 生成静态HTML文件到 /public 文件夹中
hexo server          # 本地运行server服务预览，打开 http://localhost:4000 即可预览博客
```



### 使用Netlify时需要修改的建站脚本

在根目录的`package.json`文件中添加一行脚本信息：

```bash
{
    // ......
    "scripts": {
        "build": "hexo generate",
        "clean": "hexo clean",
        "deploy": "hexo deploy",
        "server": "hexo server",
        "netlify": "npm run clean && npm run build" # 这一行为新加
    },
    // ......
}
```

并且在Netlify配置`build command`时填写`npm run netlify`。



### Git

Git基本工作原理：

![git-command](git-command.jpg)

作为程序员，Git的重要性自不必多说，于此文记录少许常用但是博主常忘于是重新上网查导致效率低下（博主超级健忘）的命令。

#### 在本地空文件夹中建仓并连接远程仓

```bash
cd <your_directory>
git init
git remote add origin <repo_address>	# 为本地仓添加远程仓repo，命名为origin
```

#### 更改branch名称

```bash
git branch -M <branch_name>
```

#### 本地提交三部曲

```bash
git add .
git commit -m "本次提交说明"
git push -u origin <branch_name>	# -u：设置上游分支，origin：远程仓库的默认名称
```

#### 切换分支并pull远程分支至本地

```bash
git checkout <branch_name>
git pull origin <branch_name>
```

#### 恢复到特定提交

```bash
git log
git checkout <commit_id>
```

该命令属于进阶操作，有待深挖。

#### 查看当前状态

```bash
git status
```

**未跟踪（Untracked）**： 新创建的文件最初是未跟踪的。它们存在于工作目录中，但没有被 Git 跟踪。

**已跟踪（Tracked）**： 通过 `git add` 命令将未跟踪的文件添加到暂存区后，文件变为已跟踪状态。

**已修改（Modified）**： 对已跟踪的文件进行更改后，这些更改会显示为已修改状态，但这些更改还未添加到暂存区。

**已暂存（Staged）**： 使用 `git add` 命令将修改过的文件添加到暂存区后，文件进入已暂存状态，等待提交。

**已提交（Committed）**： 使用 `git commit` 命令将暂存区的更改提交到本地仓库后，这些更改被记录下来，文件状态返回为已跟踪状态。



### CDN加速原理

简单来说，CDN采用的是空间换时间的思想，在全球各地建立多台CDN服务器对网站的资源进行缓存，当用户请求访问远端服务器时，直接访问用户最近的CDN缓存服务器（也可能不是最近的，具体算法由CDN负载均衡系统决定），这样就大大缩短了访问时间，同时也缓解了远端服务器的通信压力。参考下面两张流程图：

![CDN工作原理](CDN工作原理.png)

![CDN工作原理](CDN工作原理.webp)



### 域名申请

阿里云购买，实名认证申请。域名解析类型为`CNAME`。



### Netlify建站及Cloudflare加速

具体配置过程过于杂乱以至于不想回忆......（其实是因为博主超级懒）



## 参考链接：

[个人博客搭建教程 | 爱扑bug的熊 (cuijiacai.com)](https://blog.cuijiacai.com/blog-building/)

[注册Netlify账号并激活的方法 - 雾林博客 (baiwulin.com)](https://www.baiwulin.com/89.html)

[主题配置 | Hexo-theme-ShokaX (kaitaku.xyz)](https://docs.kaitaku.xyz/guide/theme.html)

