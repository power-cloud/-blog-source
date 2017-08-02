# blog-source

## 简述
博客基于非常流行的[Hexo](https://hexo.io/zh-cn/)搭建，具体说明参见[官方文档](https://hexo.io/zh-cn/docs/)

文章需要大家使用markdown编写，一些特殊的解析，遵循hexo的说明。

## 如何使用
使用过程非常简单，归纳起来就三步：
1. 把blog-source的仓库clone到本地
2. 在./source/_posts文件夹内添加文章的.md文件
3. 把新文章push到github仓库

然后会在travis的持续集成环境中解析文章内容，生成新的博客内容并更新到github pages仓库，大家就可以通过[http://power-cloud.github.io/](http://power-cloud.github.io/)来访问了。

PS:整个过程只需要git支持，完全不需要安装依赖的包，方便随时随地写内容。
