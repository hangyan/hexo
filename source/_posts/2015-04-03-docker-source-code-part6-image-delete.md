---

title: docker 源码分析(6) -- 镜像删除

tags: [docker]
imagefeature:
comments: true
share: true
excerpt: "详解镜像删除机制"
---


本篇的主要内容是关于如何删除镜像的.听起来是挺简单的一件事,但是`docker`本身的删除
策略定义并不清除，而且在实际的使用过程中,似乎总是与我们预计的结果不符,比如磁盘空间并没有被释放.所以在此单列一篇,详细解析镜
像删除的过程.

<!--more-->

## 简介
[上篇](http://hangyan.github.io/docker/docker-source-code-part5-images/)已经介绍过
很多关于镜像相关的源码,本篇就不在重复,着重分析与删除相关的部分

## 子命令执行

首先仍然是`subcommand`的执行部分,镜像删除的子命令式`rmi`,与之对应的函数便是
`CmdRmi`.

`docker rmi`接受两个参数:

`github.com/docker/docker/api/client/commands.go#CmdRmi`:

```go
var (
    cmd     = cli.Subcmd("rmi", "IMAGE [IMAGE...]", "Remove one or more images")
    force   = cmd.Bool([]string{"f", "-force"}, false, "Force removal of the image")
    noprune = cmd.Bool([]string{"-no-prune"}, false, "Do not delete untagged parents")
)
```

`force`用来决定是否在有容器使用了此镜像时(非运行状态)强制删除,`noprune`指定不删
除没有`tag`的`parent layers`.

```go
for _, name := range cmd.Args() {
    body, _, err := readBody(cli.call("DELETE", "/images/"+name+"?"+v.Encode(), nil, false))
    if err != nil {
        fmt.Fprintf(cli.err, "%s\n", err)
        encounteredError = fmt.Errorf("Error: failed to remove one or more images")
    } 
```

`docker rmi`可以接受多个参数,可以从上面代码看到,实际执行时是对每一个`image`都执
行一个`DELETE`请求并处理返回结果.

服务端的`handler`为:

`github.com/docker/docker/api/server/server.go#createRouter`:
```go
"DELETE": {
    "/containers/{name:.*}": deleteContainers,
    "/images/{name:.*}":     deleteImages,
},
```


## 请求处理

`deleteImages`的主要工作仍是`job`环境的设置以及参数的传递:

```go
func deleteImages(eng *engine.Engine, version version.Version, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
    if err := parseForm(r); err != nil {
        return err
    }
    if vars == nil {
        return fmt.Errorf("Missing parameter")
    }
    var job = eng.Job("image_delete", vars["name"])
    streamJSON(job, w, false)
    job.Setenv("force", r.Form.Get("force"))
    job.Setenv("noprune", r.Form.Get("noprune"))

    return job.Run()
}
```

`image_delete`对应的`handler`暂时定义在`daemon`包中,后续可能也会移到`graph`包中:

![ ][1]


## 镜像删除

```go
func (daemon *Daemon) ImageDelete(job *engine.Job) engine.Status {
    if n := len(job.Args); n != 1 {
        return job.Errorf("Usage: %s IMAGE", job.Name)
    }
    imgs := engine.NewTable("", 0)
    if err := daemon.DeleteImage(job.Eng, job.Args[0], imgs, true, job.GetenvBool("force"), job.GetenvBool("noprune")); err != nil {
        return job.Error(err)
    }
    if len(imgs.Data) == 0 {
        return job.Errorf("Conflict, %s wasn't deleted", job.Args[0])
    }
    if _, err := imgs.WriteListTo(job.Stdout); err != nil {
        return job.Error(err)
    }
    return engine.StatusOK
}
```


可以看到，主要的删除是通过`daemon.DeleteImage`函数进行，而`Table`结构体则是用来
记录删除的结果:

```go
type Table struct {
    Data    []*Env
    sortKey string
    Chan    chan *Env
}

func NewTable(sortKey string, sizeHint int) *Table {
    return &Table{
        make([]*Env, 0, sizeHint),
        sortKey,
        make(chan *Env),
    }
}
```

### 镜像查找
要删除镜像，就要先获取到关于这个镜像的一些详细信息：名称，`tag`,父子关系等。而输
入既可能是镜像 ID，也可能是名字:

`github.com/docker/docker/daemon/image_delete.go#DeleteImage`:

```go
// 解析名字和 tag.repoName 有可能是镜像 ID
repoName, tag = parsers.ParseRepositoryTag(name)
if tag == "" {
    tag = graph.DEFAULTTAG
}

// 查找镜像，先从 TagStore 里面找，找不到就当成镜像 ID 从 Graph 中找.`Get(repoName)`似乎
// 显得有点多余，因为 LookupImage 里面已经包含了所有步骤。
img, err := daemon.Repositories().LookupImage(name)
if err != nil {
    if r, _ := daemon.Repositories().Get(repoName); r != nil {
        return fmt.Errorf("No such image: %s:%s", repoName, tag)
    }
    return fmt.Errorf("No such image: %s", name)
}

// 如果输入的是镜像 ID，repoName 和 tag 置为空.
if strings.Contains(img.ID, name) {
    repoName = ""
    tag = ""
}
```


### 父子关系查找
docker 的镜像存储是一个树形结构，每个镜像只有一个父节点（镜像），但可以有多个子节
点（镜像）。`Graph`并没有提供多少相应的数据结构来进行节点查找，所以只能是遍历查
询：

```go
byParents, err := daemon.Graph().ByParent()
if err != nil {
    return err
}
```
`ByParent`返回了所有镜像与其子镜像(可为多个)之间的映射关系.

`github.com/docker/docker/graph/graph.go`:

```go
func (graph *Graph) ByParent() (map[string][]*image.Image, error) {
    byParent := make(map[string][]*image.Image)
    err := graph.walkAll(func(img *image.Image) {
        parent, err := graph.Get(img.Parent)
        if err != nil {
            return
        }
        if children, exists := byParent[parent.ID]; exists {
            byParent[parent.ID] = append(children, img)
        } else {
            byParent[parent.ID] = []*image.Image{img}
        }
    })
    return byParent, err
}
```


### 镜像名称与 ID 关系
镜像 ID 与名称并不是一一映射的关系。比如说我们用`docker tag`命令给一个镜像赋予了一
个新的名字，旧的名字是不会被取代的.所以经常会发现不同的镜像名称对应的 ID 都是同一
个:

![ ][2]

当我们删除镜像时，不管是指定名字还是 ID，这样的映射关系也需要提前找出来:

```go
repos := daemon.Repositories().ByID()[img.ID]
```

`github.com/docker/docker/graph/tags.go`:
```go
func (store *TagStore) LookupImage(name string) (*image.Image, error) {
    // FIXME: standardize on returning nil when the image doesn't exist, and err for everything else
    // (so we can pass all errors here)
    repos, tag := parsers.ParseRepositoryTag(name)
    if tag == "" {
        tag = DEFAULTTAG
    }
    img, err := store.GetImage(repos, tag)
    store.Lock()
    defer store.Unlock()
    if err != nil {
        return nil, err
    } else if img == nil {
        if img, err = store.graph.Get(name); err != nil {
            return nil, err
        }
    }
    return img, nil
}
```

最终`repos`的值就是一个 名称:tag 列表.

如果`repos`是多个，那么删除的情况就比较复杂.综述如下:

首先有两个基本原则:

- 要删除的镜像必须没有子节点（没有被别的镜像使用） ,不然最多只会去掉`tag`信息,不
  会删除实际数据
- 删除是沿着镜像的树形结构向上删除，会删除所有没有被使用的所有父节点,直到找到一
  个正在被别的镜像使用的镜像为止.

有点类似于 Linux 上的各种包管理软件对依赖关系的处理.具体的情况如下:


1. `docker rmi <image id>`
   - 对应单个`name:tag` 最简单情况，直接删除,
   - 对应同一个`name`,多个`tag` 直接删除
   - 对应多个`name`，不加`force`参数会报错，加上之后会将第一个`name`对应的`tag`
     信息都删掉,但镜像不会删除

2. `docker rmi name:[tag]`
   - 与`image id`一一对应  直接删除
   - 与其他的`tag`(`name`相同)对应同一个`image id` 只会去掉这个`name:[tag]`信息,
     不删除数据
   - 与不同的`name:[tag]`共享一个`image id` 只会去掉这个`name:[tag]`信息，不删除
     实际数据


情况复杂，但是分析起来，其实都是很容易理解的。


`github.com/docker/docker/daemon/image_delete.go#DeleteImage`:
```go
if repoName == "" {
    for _, repoAndTag := range repos {
        parsedRepo, parsedTag := parsers.ParseRepositoryTag(repoAndTag)
        if repoName == "" || repoName == parsedRepo {
            repoName = parsedRepo
            if parsedTag != "" {
                tags = append(tags, parsedTag)
            }
        } else if repoName != parsedRepo && !force {
            // the id belongs to multiple repos, like base:latest and user:test,
            // in that case return conflict
            return fmt.Errorf("Conflict, cannot delete image %s because it is tagged in multiple repositories, use -f to force", name)
        }
    }
} else {
    tags = append(tags, tag)
}
```

从上面代码可以看到，当输入为`image id`时,如果`name:[tag]`有多个，只会将第一个的信息清除掉,
其他的都跳过了，未做处理。

### 删除

真正在删除镜像之前，还需要考虑容器。如果要删除的镜像还有容器在使用(不论是否正在
运行)它，则不能删除.从上面总结的种种情况来看,真正会删除实际数据的情况不多，所以
只用考虑这几种情况即可:

`github.com/docker/docker/daemon/image_delete.go#DeleteImage`:
```go
if len(repos) <= 1 {
    if err := daemon.canDeleteImage(img.ID, force); err != nil {
        return err
    }
}
```

```go
func (daemon *Daemon) canDeleteImage(imgID string, force bool) error {
    // 返回容器列表
    for _, container := range daemon.List() {
        parent, err := daemon.Repositories().LookupImage(container.ImageID)
        if err != nil {
            if daemon.Graph().IsNotExist(err, container.ImageID) {
                return nil
            }
            return err
        }
        
        if err := parent.WalkHistory(func(p *image.Image) error {
            if imgID == p.ID {
                if container.IsRunning() {
                    if force {
                        return fmt.Errorf("Conflict, cannot force delete %s because the running container %s is using it, stop it and retry", stringid.TruncateID(imgID), stringid.TruncateID(container.ID))
                    }
                    return fmt.Errorf("Conflict, cannot delete %s because the running container %s is using it, stop it and use -f to force", stringid.TruncateID(imgID), stringid.TruncateID(container.ID))
                } else if !force {
                    return fmt.Errorf("Conflict, cannot delete %s because the container %s is using it, use -f to force", stringid.TruncateID(imgID), stringid.TruncateID(container.ID))
                }
            }
            return nil
        }); err != nil {
            return err
        }
    }
    return nil
}
```

从上面也可以看到`force`参数的作用:当有容器在使用这个镜像但并未运行时，`force`可
以强制删除镜像。


需要注意的上，上面对`repos`长度的判断并未覆盖全部的情况：当一个镜像 ID 对应对个一
个`name`的多个`tag`时，此时用`docker rmi <iamge id>`就绕过了与容器相关的检查,
详见[issue 12135](https://github.com/docker/docker/issues/12135).


真正的删除分两步。第一步：`untag`,删除相关`tag`信息:

```go
for _, tag := range tags {
    tagDeleted, err := daemon.Repositories().Delete(repoName, tag)
    if err != nil {
        return err
    }
    if tagDeleted {
        out := &engine.Env{}
        out.Set("Untagged", repoName+":"+tag)
        imgs.Add(out)
        eng.Job("log", "untag", img.ID, "").Run()
    }
}
```

`daemon.Repositories().Delete`即是从`TagStore`中删除相关信息.



第二步： 删除实际镜像数据，这一步并不一定会执行。

```go
tags = daemon.Repositories().ByID()[img.ID]
if (len(tags) <= 1 && repoName == "") || len(tags) == 0 {
    if len(byParents[img.ID]) == 0 {
        if err := daemon.Repositories().DeleteAll(img.ID); err != nil {
            return err
        }
        if err := daemon.Graph().Delete(img.ID); err != nil {
            return err
        }
        out := &engine.Env{}
        out.Set("Deleted", img.ID)
        imgs.Add(out)
        eng.Job("log", "delete", img.ID, "").Run()
        if img.Parent != "" && !noprune {
            err := daemon.DeleteImage(eng, img.Parent, imgs, false, force, noprune)
            if first {
                return err
            }

        }

    }
}
```

这时候的`tags`已经是清楚过一个`repo`之后的了，有可能为空，也有可能有其他的值（这
时候就不能删除镜像数据）。`len(tags) <= 1 && repoName == ""`是应对原本就没有
`tag`的镜像，比如我们在用`docker build`时生成的很多中间镜像。

真正的数据删除是在`daemon.Graph().Delete()`:

`github.com/docker/docker/graph/graph.go`:
```go
func (graph *Graph) Delete(name string) error {
    id, err := graph.idIndex.Get(name)
    if err != nil {
        return err
    }
    tmp, err := graph.Mktemp("")
    graph.idIndex.Delete(id)
    if err == nil {
        err = os.Rename(graph.ImageRoot(id), tmp)
        // On err make tmp point to old dir and cleanup unused tmp dir
        if err != nil {
            os.RemoveAll(tmp)
            tmp = graph.ImageRoot(id)
        }
    } else {
        // On err make tmp point to old dir for cleanup
        tmp = graph.ImageRoot(id)
    }
    // Remove rootfs data from the driver
    graph.driver.Remove(id)
    // Remove the trashed image directory
    return os.RemoveAll(tmp)
}
```

`/var/lib/docker/graph`和`/var/lib/docker/aufs/`下的相关目录都会被清除.



## 总结
在明晰了镜像删除的原理之后，自己如果确实想要释放磁盘空间，可以人工对相应目录里的
数据进行删除。


[1]: http://hangyan.github.io/images/posts/docker/source-6/handler-name.png "handler-name"
[2]: http://hangyan.github.io/images/posts/docker/source-6/multi-names.png "multi-names"






