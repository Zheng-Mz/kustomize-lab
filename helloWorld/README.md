[base]: ../../docs/glossary.md#base
[config]: https://github.com/kinflate/example-hello
[gitops]: ../../docs/glossary.md#gitops
[hello]: https://github.com/monopole/hello
[kustomization]: ../../docs/glossary.md#kustomization
[original]: https://github.com/kinflate/example-hello
[overlay]: ../../docs/glossary.md#overlay
[overlays]: ../../docs/glossary.md#overlay
[patch]: ../../docs/glossary.md#patch
[variant]: ../../docs/glossary.md#variant
[variants]: ../../docs/glossary.md#variant

# Demo: hello world with variants

步骤：

 1. 拉取已经存在的的 [base] 配置。
 1. 进行定制。
 1. 基于定制后的 base 新建2个不同的 [overlays] (_staging_ 和 _production_)。
 1. 运行 kustomize 和 kubectl 来部署 staging 和 production 。

首先创建一个工作空间：

<!-- @makeWorkplace @test -->
```
DEMO_HOME=$(mktemp -d)
```

或者：
> ```
> DEMO_HOME=~/hello
> ```

## 创建 base

让我们开始运行 [hello] 服务。

如果要使用 [overlays] 创建 [variants] ，必须先创建一个共同的 [base] 。

为了使本文档保持简洁，base 的资源位于补充目录中，并不在此处，请按照下面的方法下载它们：

<!-- @downloadBase @test -->
```
BASE=$DEMO_HOME/base
mkdir -p $BASE

curl -s -o "$BASE/#1.yaml" "https://raw.githubusercontent.com\
/kubernetes-sigs/kustomize\
/master/examples/helloWorld\
/{configMap,deployment,kustomization,service}.yaml"
```

观察该目录：

<!-- @runTree @test -->
```
tree $DEMO_HOME
```

可以看到：

> ```
> /tmp/tmp.IyYQQlHaJP
> └── base
>     ├── configMap.yaml
>     ├── deployment.yaml
>     ├── kustomization.yaml
>     └── service.yaml
> ```

这些 resources 可以立即在 k8s 集群中部署：

> ```
> kubectl apply -f $DEMO_HOME/base
> ```

实例化 _hello_ 服务， `kubectl` 只能识别 resources 文件。


### The Base Kustomization

`base` 目录中包含一个 [kustomization] 文件：

<!-- @showKustomization @test -->
```
more $BASE/kustomization.yaml
```

（可选）在 base 上运行 `kustomize` 将自定义 resources 输出到 `stdout` ：

<!-- @buildBase @test -->
```
kustomize build $BASE
```

### Customize the base 定制 base

定制 _app label_ 并应用于所有的 resources ：

<!-- @addLabel @test -->
```
sed -i.bak 's/app: hello/app: my-hello/' \
    $BASE/kustomization.yaml
```

查看效果：
<!-- @checkLabel @test -->
```
kustomize build $BASE | grep -C 3 app:
```

## 创建 Overlays

创建包含 _staging_ 和 _production_ 的 [overlay]：

 * _Staging_ 包含生产环境中无法应用的带有风险的功能。
 * _Production_ 包含更多的副本数。
 * 来自这些集群 [variants] 的问候将与来自其他集群的不同。

<!-- @overlayDirectories @test -->
```
OVERLAYS=$DEMO_HOME/overlays
mkdir -p $OVERLAYS/staging
mkdir -p $OVERLAYS/production
```

#### Staging Kustomization

在 `staging` 目录中创建一个 kustomization 文件，用来定义一个新的 name 前缀和一些不同的 labels 。

<!-- @makeStagingKustomization @test -->
```
cat <<'EOF' >$OVERLAYS/staging/kustomization.yaml
namePrefix: staging-
commonLabels:
  variant: staging
  org: acmeCorporation
commonAnnotations:
  note: Hello, I am staging!
bases:
- ../../base
patchesStrategicMerge:
- map.yaml
EOF
```

#### Staging Patch

新增一个自定义的 configMap 将问候从 _Good Morning!_ 改为 _Have a pineapple!_ 。

同时，将 _risky_ 标记设置为 true 。

<!-- @stagingMap @test -->
```
cat <<EOF >$OVERLAYS/staging/map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: the-map
data:
  altGreeting: "Have a pineapple!"
  enableRisky: "true"
EOF
```

#### Production Kustomization

在 `production` 目录中创建一个 kustomization 文件，用来定义一个新的 name 前缀和 labels 。

<!-- @makeProductionKustomization @test -->
```
cat <<EOF >$OVERLAYS/production/kustomization.yaml
namePrefix: production-
commonLabels:
  variant: production
  org: acmeCorporation
commonAnnotations:
  note: Hello, I am production!
bases:
- ../../base
patchesStrategicMerge:
- deployment.yaml
EOF
```


#### Production Patch

因为生产环境需要处理更多的流量，新建一个 production patch 来增加副本数。

<!-- @productionDeployment @test -->
```
cat <<EOF >$OVERLAYS/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: the-deployment
spec:
  replicas: 10
EOF
```

## Compare overlays 比较 overlays


`DEMO_HOME` 现在包含：

 - _base_ 目录：对拉取到的源配置进行了简单定制

 - _overlays_ 目录：包含在集群中创建不同 _staging_ 和 _production_ [variants] 的 kustomizations 和 patches 。

查看目录结构和差异：

<!-- @listFiles @test -->
```
tree $DEMO_HOME
```

可以看到：

> ```
> /tmp/tmp.IyYQQlHaJP1
> ├── base
> │   ├── configMap.yaml
> │   ├── deployment.yaml
> │   ├── kustomization.yaml
> │   └── service.yaml
> └── overlays
>     ├── production
>     │   ├── deployment.yaml
>     │   └── kustomization.yaml
>     └── staging
>         ├── kustomization.yaml
>         └── map.yaml
> ```

直接比较 _staging_ 和 _production_ 输出的不同：

<!-- @compareOutput -->
```
diff \
  <(kustomize build $OVERLAYS/staging) \
  <(kustomize build $OVERLAYS/production) |\
  more
```

部分比较输出：

> ```diff
> <   altGreeting: Have a pineapple!
> <   enableRisky: "true"
> ---
> >   altGreeting: Good Morning!
> >   enableRisky: "false"
> 8c8
> <     note: Hello, I am staging!
> ---
> >     note: Hello, I am production!
> 11c11
> <     variant: staging
> ---
> >     variant: production
> 13c13
> (...truncated)
> ```


## 部署

输出不同 _overlys_ 的配置：

<!-- @buildStaging @test -->
```
kustomize build $OVERLAYS/staging
```

<!-- @buildProduction @test -->
```
kustomize build $OVERLAYS/production
```

将上述命令传递给 kubectl 进行部署：

> ```
> kustomize build $OVERLAYS/staging |\
>     kubectl apply -f -
> ```

> ```
> kustomize build $OVERLAYS/production |\
>    kubectl apply -f -
> ```

也可使用 `kubectl` (v1.14.0 以上版本)：

> ```
> kubectl apply -k $OVERLAYS/staging
> ```

> ```
> kubectl apply -k $OVERLAYS/production
> ```