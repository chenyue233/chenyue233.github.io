# My First Hugo Blog


# HI 

实践出真知

###  `hugo` 基础命令

####  创建一个新的网站

```
hugo new site quickstart
```

#### **添加一个主题**

```
cd quickstart
git init
git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
```

####  **添加一个文章**

```
hugo new posts/my-first-post.md
```



#### 启动 `Hugo` 服务器

```
hugo server -D
```



#### **打包网站到 `/public` 文件夹**

```
hugo -d public
```


