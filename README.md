# Blog

[![Build Status](https://travis-ci.org/xianmobilelab/blog.svg?branch=master)](https://travis-ci.org/xianmobilelab/blog)

## Setup

``` shell
npm install hexo@3.3.9 -g
clone project
npm install
```

## How to create a post?

```
git checkout -b post/the-post-title
cd blog
hexo new the-post-title
# edit your post and commit with git
# create a pull request and waiting for review and merge
# xianmobile.com will auto deploy when CI pipline is done.
```

## How to use image resource in post?

1. When you create a new post, you also can see a same name directory in `source/_posts`
1. Please put the image into this directory. such as `source/_posts/my-image.png`
1. put this one into your post `{% asset_img my-image.png "" %}`

## Keep consistent

### About `#`

When you use `##` as a title, then the subtitle should be `###` not `####`, `#####` and `######`, please do not skip the leave. just like this:

``` markdown
## Subhead

### Caption

#### Title

##### Subtitle

###### Item

body

## Subhead 2
...
```

### About reference your blog

Please put your name and link on the top of post. just like this:

``` markdown
> 作者: 张三
> 原文: [https://example.com/blog-path](https://example.com/blog-path)
```

