# How to hugo

<span style="color:red">Replace `zhan9san` with your own github ID if you want
create your own github pages
</span>

## One-time task

### Create Your Project

It will create a new directory, named `zhan9san.github.io` in current directory.

```bash
hugo new site zhan9san.github.io
cd zhan9san.github.io
```

### Install the Theme

```bash
git init
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

### Basic Configuration

The following is a basic configuration, `config.toml` for the LoveIt theme:

```toml
baseURL = "https://zhan9san.github.io/"
# [en, zh-cn, fr, ...] determines default content language
defaultContentLanguage = "en"
# language code
languageCode = "en"
title = "My New Hugo Site"

# Change the default theme to be use when building the site with Hugo
theme = "LoveIt"

[params]
  # LoveIt theme version
  version = "0.2.X"

[menu]
  [[menu.main]]
    identifier = "posts"
    # you can add extra information before the name (HTML format is supported), such as icons
    pre = ""
    # you can add extra information after the name (HTML format is supported), such as icons
    post = ""
    name = "Posts"
    url = "/posts/"
    # title will be shown when you hover on this menu link
    title = ""
    weight = 1
  [[menu.main]]
    identifier = "tags"
    pre = ""
    post = ""
    name = "Tags"
    url = "/tags/"
    title = ""
    weight = 2
  [[menu.main]]
    identifier = "categories"
    pre = ""
    post = ""
    name = "Categories"
    url = "/categories/"
    title = ""
    weight = 3

# Markup related configuration in Hugo
[markup]
  # Syntax Highlighting (https://gohugo.io/content-management/syntax-highlighting)
  [markup.highlight]
    # false is a necessary configuration (https://github.com/dillonzq/LoveIt/issues/158)
    noClasses = false

```

### Push to github pages

1. Create a github repo, named `zhan9san.github.io`
2. Change the Github pages source. `Settings` -> `Pages` -> `Source` -> `Select folder` to `/docs`
3. Push codes

    ```bash
    git remote add origin git@github.com:zhan9san/zhan9san.github.io.git
    git add .
    git commit -m 'init'
    git push -u origin master
    ```

## Multiple-time task

### Create a Post

```bash
hugo new posts/first_post.md
```

### Launching the Website Locally

```bash
hugo serve
```

### Build the Website

```bash
hugo -d docs
```

### Push posts

```bash
git add .
git commit -m 'some comments'
git push
```

## Reference

[邵励志](https://zhuanlan.zhihu.com/p/105021100)
