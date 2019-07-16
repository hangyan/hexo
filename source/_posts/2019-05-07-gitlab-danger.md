---
title: 在 Gitlab 中使用 Danger
toc: true
tags:
  - 工具
  - CI/CD
  - Gitlab
date: 2019-05-07 23:23:52
excerpt: 通过 Danger 扩充 Gitlab 的 CI/CD 功能
categories: 技术
---

Gitlab 社区版的 CI 功能非常好用，能够很方便的地做到代码的 lint/build/test/等等。不过社区版在多人协作上(比如 Merge Request)上阉割了不少功能，
比如将 MR assign 给多人等。通常来说，在代码合并这块，CI/CD 一般包括两部分: 代码本身以及 MR/PR 本身。Danger 这个工具正好可以补足 Gitlab 在后者的不足。

## 功能

Gitlab CI 的关注点在于提交的代码本身，而 Danger 的关注点在于 Merge Request 本身，当然也可以做到很多 Gitlab CI 能做到的事情，各种第三方插件也能极大地扩种 Danger 自身的能力。目前我觉得几个非常有用的功能是：

1. 检查 Commit Message 的格式。这个功能是很基本的，但是很多 CI 系统本身都不支持。
2. 检查与 jira 的关联。强制让每一个 MR 都关联一个 jira,方便项目管理
3. 检查 MR 是否打标签。在 MR 非常多的时候用于给 MR 归类，在 Github 上的大项目上我们经常见到
4. 检查 MR 的大小。改动太大的 MR 是不推荐的，因为 Review 起来难度太大，推荐分裂成比较小的 MR
5. 检查是否 rebase 过了目标分支,保持一个干净的提交记录。


## 安装

因为 MR 本身属于 Git 系统的一个功能，所以 Danger 的一个主要能力在于与各大平台的集成性上,目前主流的 Gitlab/Github/都支持。也很容易部署。下面简要介绍与 Gitlab 的集成方法

### 创建用户

1. 创建一个新的用户，给予目标 Group 的 Reporter 权限
2. 生成一个具有 API 访问权限的 TOKEN，并且在目标项目的 CI/CD 配置里加到环境变量里: `DANGER_GITLAB_API_TOKEN`


### 添加 Danger 到 Gitlab CI 文件中

一个示例如下：

```yaml
danger:
  only:
    - merge_requests
  stage: pre
  image: hangyan/danger
  script:
    - danger --fail-on-errors=true
```

将其放在一个单独的 stage 中，一般在 MR 最开始的时候运行。`hangyan/danger`镜像的 Dockerfile 如下:

```Dockerfile
FROM ruby:2.4.1
MAINTAINER hangyan <hangyan@hotmail.com>

RUN gem install danger-gitlab danger-jira danger-commit_lint

RUN danger --version

ENTRYPOINT ["danger"]
```

内容比较简单，安装了三个包

1. danger-gitlab: danger 本身
2. danger-jira: 检查与 jira 的关联
3. danger-commit_lint: 检查 Commit Message

### 添加 Dangerfile

Dangerfile 本身是一个 ruby 文件，作为配置文件来说有一定的学习成本，但一般的简单配置官网都可以找到示例。下面是一个 Example Config:


```ruby
# check milestone
has_milestone = gitlab.mr_json["milestone"] != nil
warn("This MR does not refer to an existing milestone", sticky: true) unless has_milestone

# check label
failure "Please add labels to this MR" if gitlab.mr_labels.empty?


# check jira
jira.check(
  key: ["DEV", "INCI"],
  url: "http://jira.alaudatech.com/browse",
  search_title: true,
  search_commits: true,
  fail_on_warning: true,
  report_missing: true,
  skippable: true
)


# Ensure a clean commits history
if git.commits.any? { |c| c.message =~ /^Merge branch/ }
  fail('Please rebase to get rid of the merge commits in this PR')
end


# Warn when there is a big PR
warn("Big PR, try to keep changes smaller if you can") if git.lines_of_code > 500


# commit message
commit_lint.check

```


每一部分都有响应的注释。


### 效果

![](/images/gitlab/danger.png)



## Links
1. [Dockerfile](https://github.com/hangyan/Danger)
2. [Danger](https://danger.systems/guides/getting_started.html)

