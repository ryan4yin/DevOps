# Jenkins Pipeline

参考：https://jenkins.io/doc/pipeline/examples


## 加快构建速度

在 「系统管理」-「系统配置」-「Pipeline Speed/Durability Settings」，将它设为 「Performance Optimized」


## 脚本片段

获取所有节点名称：

```
// This method collects a list of Node names from the current Jenkins instance
@NonCPS
def nodeNames() {
  return jenkins.model.Jenkins.instance.nodes.collect { node -> node.name }
}
```

自动注入 ssh 密钥用于 git 操作：

```
// For SSH private key authentication, try the sshagent step from the SSH Agent plugin.
sshagent (credentials: ['git-ssh-credentials-ID']) {
    sh("git tag -a some_tag -m 'Jenkins'")
    sh('git push <REPO> --tags')
}
```


拉取 git 仓库：
```
// 代码拉到当前文件夹中
git branch: 'dev', credentialsId: 'git-ssh-credentials-ID', url: 'http://gitlab.local/test_repo'

// 代码拉到文件夹 test_dir 中，会新建文件夹
sshagent (credentials: ['git-ssh-credentials-ID']) {
    sh("git clone --branch dev 'http://gitlab.local/text_repo' test_dir")
}
```
