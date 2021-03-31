本项目是一个云书签的项目，取名为：签签世界。

部署教程：[docker-compose部署](https://github.com/FleyX/bookmark/blob/master/DEPLOY.md)

自用地址（你们也可以用，长期提供服务）:[bm.tapme.top](https://bm.tapme.top)


# 缘由

1. 主要用的是 chrome，但是有时候需要用其他的浏览器：Firefox，ie 等。然后这些浏览器上没有书签，想进个网站还得打开 chrome 复制 url，太麻烦。

2. chrome 必须翻墙才能同步书签，体验不是那么好。

3. 如果书签全放在 chrome 上，相当于绑定死 chrome 浏览器了，很难迁移到别的优秀浏览器，比如 firfox 上。

所以有了这样这样一个项目，建立一个和平台无关的书签管理器，可在任意平台使用。

# 主要功能

1. 基础的书签增删改查功能。支持chrome、firefox等浏览器书签文件导入，导出。

   按需加载书签节点，即使上万条书签也不会卡

   支持鼠标拖拽排序、移动位置、层级（移动端不支持）

   支持书签文件双向导入导出（从浏览器导入到系统、从系统导出到浏览器）

2. 强大的书签检索功能，毫秒级的关键字检索。

   支持使用 书签名搜索、url搜索、书签名拼音搜索、书签名拼音首字母搜索。（注意不支持组合使用）
4. 移动端支持，web页面可在浏览器中正常使用

## TODO

- 主页功能 
- 拼音检索 Ok!
- 书签导出 OK！
- 侧边栏显示
