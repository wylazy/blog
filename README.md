# hexo 博客源码

网站地址: [http://www.wangyulong.cn/](http://www.wangyulong.cn)

环境安装：

1. 配置主题为 maupassant
相关配置请参考 [maupassant](https://github.com/tufu9441/maupassant-hexo)

2. 为支持 MathJax 需要安装 pandoc
```
brew install pandoc
npm install node-pandoc --save
npm install hexo-renderer-mathjax2 --save
```
3. 为产生 RSS，需要安装以下插件
```
npm install hexo-generator-feed --save
```
用 hexo generate 就会自动生成 RSS ，放在 public/atom.xml

4. 为使用Graphviz，需要安装一下插件
```
brew install graphviz
npm install hexo-tag-graphviz --save
```
使用如下代码生成图片
```
{% graphviz "Caption of the graphviz" %}
digraph {
  A -> B;
}
{% endgraphviz %}
```
