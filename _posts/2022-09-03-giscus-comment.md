---
layout: post
title: 如何给自己的静态网站添加giscus评论模块？
category: common
tags: [common]
description: 如何给自己的静态网站添加giscus评论模块？
keywords: 个人网站,免费评论模块
excerpt: 如何给自己的静态网站添加giscus评论模块？
---

你好，我是Weiki，欢迎来到猿java。

维护个人的技术网站已经成为了很多程序员的日常工作，如何搭建免费的技术博客，可以参考我往期的文章：[如何搭建自己免费的技术博客](https://www.yuanjava.cn/posts/blog/)，有了网站后，我们希望有个评论区可以和读者互动，因此我们就来分享一下如何
搭建一个免费的评论模块。


## 素材
我们采用的是 [giscus](https://giscus.app/)，它是一个开源，免费的工具，可以和github很好的集成。

## 步骤

### 创建评论仓库

因为giscus是和github集成，所以需要在[github](https://github.com/)上创建一个 public的仓库用于存放评论的内容。比如仓库名：blog-comments

## 评论仓库授权
在 github上，导航到存储库blog-comments的主页，点击 ⚙️Settings 按钮
![img.png](https://www.yuanjava.cn/assets/md/common/repo-set.png)

找到 Discussions功能勾选上，再点击 Set up Discussions按钮，默认配置后，点击 Start Discussions按钮，授权完成
![img.png](https://www.yuanjava.cn/assets/md/common/giscussions.png)

![img.png](https://www.yuanjava.cn/assets/md/common/start-discuss.png)

## 启用Discussions

前往[giscus](https://github.com/apps/giscus)对 blog-comments启用giscus

![img.png](https://www.yuanjava.cn/assets/md/common/finish-discuss.png)


## 获取存储仓库的API 密钥

可以通过 GitHub GraphQL API 访问 GitHub 详细信息，也可以在[此链接](https://docs.github.com/en/graphql/overview/explorer)访问，然后使用你的GitHub 帐户登录

请求参数如下：
```text
// owner 是你github的账户名， name是你存储评论的仓库名
query {
  repository(owner: "yuanjavar", name:"blog-comments"){
    id
    discussionCategories(first:10) {
      edges {
        node {
          id
          name
        }
      }
    }
  }
}
```

接口返回，
```text
{
  "data": {
    "repository": {
      "id": "R_kgXssHsd5g",   // 仓库id
      "discussionCategories": {
        "edges": [
          {
            "node": {
              "id": "DIC_DJSHUSs4M-2",
              "name": "Announcements"
            }
          },
          {
            "node": {
              "id": "DIC_kDDOWWMs4CRM-3", // category id
              "name": "General"           // category name
            }
          },
          {
            "node": {
              "id": "DIC_kdyejDW4CRM-5",
              "name": "Ideas"
            }
          },
          {
            "node": {
              "id": "DIC_kSKDW4CRM-7",
              "name": "Polls"
            }
          },
          {
            "node": {
              "id": "DIC_kwDDDW5s4CRM-4",
              "name": "Q&A"
            }
          },
          {
            "node": {
              "id": "DIC_kwDKEKED5RM-6",
              "name": "Show and tell"
            }
          }
        ]
      }
    }
  }
}
```

### 安装 @giscus/react 包
```text
npm i @giscus/react
```

### 导入并使用 Giscus 组件

```text
import { Giscus } from "@giscus/react";

export default function Comment() {
  return (
    <Giscus
      repo="yuanjavar/blog-commnets"
      repoId="R_kgDOGjYtbQ"
      category="General"
      categoryId="DIC_kwDOGjYtbc4CA_TS"
      mapping="pathname"
      reactionsEnabled="0"
      emitMetadata="0"
      theme="dark"
    />
  );
}
```

### 最后我们看下效果

![img.png](https://www.yuanjava.cn/assets/md/common/result.png)


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

