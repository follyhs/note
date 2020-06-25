## Golang 共享库（module-go)

## 新手入门

### Step1：加入 module-go 项目组

项目组地址：https://code.aliyun.com/groups/module-go

### Step2：修改 git 配置

```
git config --global url.git@code.aliyun.com:.insteadOf https://code.aliyun.com/
```

### Step3：项目中开始使用

```
# 针对 go 1.11 以下版本，需要先设置好 GOPATH export GOPATH=YOUR_PROJECT_DIR  cd YOUR_PROJECT_DIR # 下载你所需要的包，比如需要使用 irpc，go get 会自动下载依赖的包 go get -u code.aliyun.com/module-go/irpc  
# 如果发现 go get 报错：SSL connection error... ，可以执行如下命令，然后再次尝试 go get export GIT_SSL_VERSION=tlsv1 git config http.sslVersion tlsv1
```



## AE 日志分析常用命令

```
# 统计所有模块的超时数排名 grep -E '(FATAL|ERROR)' /mnt/engine/arbiter/log/arbiter-8003.1.INFO | awk -F'[ \t]' '{count[$7]++} END {for (obj in count) {print count[obj], obj}}' | sort -nr 

# 统计所有模块的平均耗时排名 grep 'obj=' /mnt/engine/arbiter/log/arbiter-8003.1.INFO | grep 'consume=' | awk -F'(consume=|\t)' '{consume[$6] += $NF;count[$6] +=1;} END {for (obj in consume) {print consume[obj] / count[obj], obj}}' | sort -nr 

# 统计所有模块超过某个耗时的请求数排名 grep 'obj=' /mnt/engine/arbiter/log/arbiter-8003.1.INFO | grep 'consume=' | awk -F'(consume=|\t)' '{if ($NF > 50) {count[$6] +=1;}} END {for (obj in count) {print count[obj], obj}}' | sort -nr 

# 统计某个服务的超时率，比如：天网画像 obj=profile，统计不同的模块需要更换 obj，天网统计引擎为 obj=cppstats-register-mysql awk '/event=GotRequest/{sum++} /obj=profile.*Timed Out/{obj++}  END {print obj/sum, sum, obj}' /mnt/engine/arbiter/log/arbiter-8003.1.INFO 

# 统计某个服务的平均耗时（不包含超时的请求） grep 'obj=profile' /mnt/engine/arbiter/log/arbiter-8003.1.INFO | grep 'consume=' | awk -F'consume=' '{consume += $NF;count +=1;} END {print consume / count, consume, count}' 

# 统计某个服务的耗时分布 grep 'obj=profile' /mnt/engine/arbiter/log/arbiter-8003.1.INFO | grep 'consume=' | awk -F'consume=' '{count[$NF]++;} END {for (cost in count) {print cost, count[cost]}}' | sort -nr 

# 查看请求耗时高的模块 tail -F /mnt/engine/arbiter/log/arbiter-8003.INFO | grep 'obj=' | grep 'consume=' | awk -F'consume=' '{if ($NF > 20) {print $0}}' 

# 某家公司 qps 统计 grep -h 'event=GotRequest' /mnt/engine/arbiter/log/arbiter-8003.[1-9].INFO | grep TN83YsuKoZNwEDY5Qe8w | awk -F'[ \t.]' '{s[$2]++} END {for (t in s) {print s[t], t}}' | sort -nr | head  

# AE 处理整个请求的平均耗时 grep -h 'event=Finished' /mnt/engine/arbiter/log/arbiter-8003.[1-5].INFO | awk -F'consume=' '{print $2}' | awk '{cost+=$1; count++;} END {print cost/count}'
```

## Go Module 使用及迁移说明（go mod）

go module 是 Go1.11 新增的特性，从官方层面支持对依赖库的管理。

Modules 官方定义如下：

> 模块是相关Go包的集合。modules是源代码交换和版本控制的单元。 go命令直接支持使用modules，包括记录和解析对其他模块的依赖性。modules替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。

## 为什么使用 Go Module

目前存在的问题：

-  项目代码库膨胀厉害（直接提交第三方库源码）
- 不知道依赖库的版本
- 依赖库更新方式不友好
- 如果更新依赖库时，可能存在不兼容情况

Go Module 的好处：

- 无需再定义项目的 GOPATH
- 无需再提交依赖库源码（依赖库会缓存到 $GOPATH/pkg/mod 中进行集中管理）
- import 依赖路径和项目路径进行解耦，不用再存放在一起（所有项目可以共用 依赖库缓存）
- 明确依赖库列表、依赖关系
- 支持多版本依赖（通过 tag 区分版本，如果没有符合规范的 tag，会根据最近一次的 commit id 标识）

## GO111MODULE

GO111MODULE 是一个环境变量，用于控制是否启用 go mod 功能。

`GO111MODULE` 有三个值：

| 值   | 说明                                                         |
| :--- | :----------------------------------------------------------- |
| off  | 不适用 module 功能                                           |
| on   | 使用 module，和 GOPATH 不能共存                              |
| auto | 根据当前目录来决定是否启用 module 功能，触发启用的情况分两种：当前目录在 GOPATH/src 之外且该目录包含 go.mod 文件当前目录是包含 go.mod 文件的目录的子目录 |


设置方式：

```
# 全局生效 go env -w GO111MODULE=on  # 当前会话生效 export GO111MODULE=on
```

## go mod 使用说明

### 常用命令

```
# 查看支持哪些选项 ✗ go help mod 
# 初始化 go mod 
# go mod init [host/namespace/module-name]，比如  
✗ go mod init code.aliyun.com/arch/be-derive 
# 拉取缺少的模块，移除不用的模块 
✗ go mod tidy 
# 列出当前模块及其所有依赖项 
✗ go list -m all 
# 查看为什么依赖某个包 
✗ go mod why code.aliyun.com/module-go/irpc be-derive/dist/dist/config imports 
# 打印模块依赖图 go mod graph
```

### 使用约定

| 命令                        | 约定                                                         |
| :-------------------------- | :----------------------------------------------------------- |
| `go mod init [module name]` | `使用 域名/组名/项目（包）名这种完全路径的形式`比如：irpc 包：code.aliyun.com/module-go/irpcbe-derive 项目：code.aliyun.com/arch/be-derive |

##  迁移到 go module

### Step1：设置环境变量

```
# 私有仓库拉取地址替换 
✗ git config --global url.git@code.aliyun.com:.insteadOf https://code.aliyun.com/ 
# 设置自动开启 go module 
✗ go env -w GO111MODULE=auto 
# go 第三方库代理，解决国外库拉取不下来的问题 
# 如果此变量设置之后，拉取第三方库仍有问题，可以将 direct 去掉，再次尝试 
✗ go env -w GOPROXY=https://goproxy.io,direct 
# 私有库不走代理，设置原因：code.aliyun.com 是我们的私有库，不能通过 goproxy 拉取 
✗ go env -w GOPRIVATE=code.aliyun.com
```

### Step2：去掉当前项目的 GOPATH 设置

- 不要设置当前环境变量 GOPATH 为当前项目，可以执行 unset GOPATH，取消设置
- 去掉当前项目的 build.sh 文件中的 GOPATH 设置
- GOBIN 仍然保留

GOBIN 保留的原因：

1. 本次迁移主要是为了解决依赖管理问题，这个过程中必须去掉 GOPATH（大势所趋），但是不用去掉 GOBIN；
2. 目前看 GOBIN 会在 Golang 官方长期支持，而且除了用 go build 没有特别好的替代方案，所以，我们仍然保留 GOBIN；（或许以后有机会可以去掉）
3. 为什么不使用 go build，我们期望可以约定一个生成 bin 可执行文件的目录，GOBIN + go install 可以很好的解决这个问题；

### Step3：利用 go mod 初始化当前的项目

```
# 进入项目的根目录，比如 cd project/be-derive 
# 执行 go mod init mod-name 进行初始化 
# 注意：mod-name 格式为 "域名/组名/项目（包）名"，即 import 的导入路径 
✗ go mod init code.aliyun.com/arch/be-derive
```

### Step4：替换本地包路径

import 引入的包分三种：

- GOROOT 中的标准库，如 "fmt"
- 当前项目的本地库，如 "shumei/config"
- 公共库、第三方库，如 "code.aliyun.com/module-go/irpc"

由于新版的依赖管理要求，除了 GOROOT 下标准库以外，引入路径都必须是域名（判断依据是：引入路径的第一段是否包含 . 符号）。

所以，我们需要将本地库引入路径替换为 host/pkgname 的形式，否则将会有如下报错：

```
malformed module path “shumei/config” missing dot in first pat
```

路径替换形式，以 be-derive 本地包 shumei/config 为例：

（我们不再保留 src 目录）

```
# 1.将所有 src 下的本地包目录移动到 src 同级（目前应该是 src 的本地包只有 shumei 开头） 
✗ mv src/shumei ./ 
# 2.将本地引入的 shumei/xxx 替换为 code.aliyun.com/arch/be-derive/shumei/xxx 
# 比如： import "shumei/config" 替换为 import "code.aliyun.com/arch/be-derive/shumei/config"
# 此处使用 sed 批量替换即可
```

### Step5：开始使用 go mod 构建项目

```
# 清楚无用库 
✗ go mod tidy 
# 编译当前项目 
✗ ./dist/bin/build.sh 
# 执行编译过程中会拉取依赖库，默认存放位置是 $GOPATH/pkg/mod/xxx 目录下 
# GOPATH 的默认值是 $HOME/go 
# 查看缓存库 
✗ ll ~/go/pkg/mod total 24K drwxrwxr-x  4 fanjingwen fanjingwen 4.0K Mar 31 23:55 cache drwxrwxr-x  3 fanjingwen fanjingwen 4.0K Mar 31 23:32 code.aliyun.com drwxrwxr-x 13 fanjingwen fanjingwen 4.0K Mar 31 23:55 github.com drwxrwxr-x  3 fanjingwen fanjingwen 4.0K Mar 31 23:32 golang.org drwxrwxr-x  3 fanjingwen fanjingwen 4.0K Mar 31 23:32 gopkg.in drwxrwxr-x  5 fanjingwen fanjingwen 4.0K Mar 31 23:32 go.uber.org
```

## 目录结构约定

经过上述的迁移，我们的项目会形成如下的目录结构：

```
✗ ls bin  dist  go.mod  go.sum  README.md  shumei  
✗ ls dist bin  config  dist  
✗ ls dist/dist bin  config  log
```



| 目录名 | 功能                                                         |
| :----- | :----------------------------------------------------------- |
| bin    | 指定 GOBIN 目录，生成编译后的执行文件                        |
| dist   | dist 第一层用于存放配置文件和 shell 脚本，包含如下子目录：dist/bin 存放项目的 shell 脚本dist/config 存放项目的配置文件，一般为 xxx.jsondist/distdist/dist 用于存放编译后生成的整个项目结构，生成后包含如下子目录./dist/dist/bin 存放项目的 shell 脚本和编译后的执行文件，从 ./dist/bin 和 $GOBIN 复制过来./dist/dist/config 存放项目的配置文件，从 ./dist/config 复制过来./dist/dist/log 存放项目的日志 |
| shumei | 存放本项目的内部包（本地包）                                 |



## 问题与解决

### 问题一：glog import 出错

直接将自己项目中的 glog 引入替换为：code.aliyun.com/module-go/glog 即可。

具体如下：

```
import "github.com/glog" 替换为： import "code.aliyun.com/module-go/glog"
```

### 问题二：code.aliyun.com 项目拉取错误

回看：设置环境变量操作，确定 GOPRIVATE 设置正确。

### 问题三：build.sh 时，提示：$GOPATH/go.mod exists but should not

build.sh 中设置了 GOPATH，把相应设置去掉即可。