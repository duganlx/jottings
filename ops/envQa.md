# 开发环境 Q&A

## 开发环境

Q：好用的 vscode 插件

- `Docker`, `Remote - Container`
- `Prettier - Code formatter`

Q：vscode 配置

```
Tab Size: 2
Word Wrap: on
```

Q：wins 开发软件

`wsl2`, `Windows Terminal`

Q：jhl 远程服务器访问

```bash
ssh -oHostKeyAlgorithms=+ssh-dss {USER}@192.168.3.200 -p 22

# .bashrc
alias sshfork='sshpass -p 141915.yun1 ssh -oHostKeyAlgorithms=+ssh-dss  lvxiang@192.168.3.200 -p 22'
```

## 前端

Q：在安装 react-grid-layout 依赖后，引用时显示 无法找到模块“react-grid-layout”的声明文件。 “/workspace/node_modules/react-grid-layout/index.js”隐式拥有 "any" 类型。

使用运行命令并重启 vscode 即可 `yarn add @types/react-grid-layout`。

## go

Q：jhlfund 私仓配置

```bash
# 配置go mod 私有仓库
go env -w GOPRIVATE=gitlab.jhlfund.com
# 配置不使用代理
go env -w GONOPROXY=gitlab.jhlfund.com
# 配置不验证包(无用)
go env -w GONOSUBDB=gitlab.jhlfund.com
# 配置不加密访问
go env -w GOINSECURE=gitlab.jhlfund.com
```

Q: go 依赖包保存位置

- `/root/.cache/go-build`
- `/root/gowork/pkg/mod ---> GOPATH/pkg/mod`

## python

Q：在.ipynb 文件中，出现 OSError: /usr/local/miniconda/envs/py310/lib/python3.10/site-packages/zmq/backend/cython/../../../../.././libstdc++.so.6: version `GLIBCXX_3.4.30' not found (required by /usr/local/miniconda/envs/py310/lib/python3.10/site-packages/pygsf/libs/libgsf2-log.so)