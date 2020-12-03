
## [pulumi](https://github.com/pulumi/pulumi) - 基础设施代码化

pulumi 和 terraform 一样，都是自动化管理基础设施的工具，但是它解决了 terraform 配置的一个痛点：配置语法太过简单，导致配置繁琐。

terraform 虽然应用广泛，但是它默认使用的 HCL 语言太简单，表现力不够强。
这导致在更复杂的场景下，我们无法更自动化地进行基础设施配置，而需要更复杂的步骤：

1. 借助 Python 等其他语言先生成出 HCL 配置
2. 通过 `terraform` 命令行进行 plan 与 apply
3. 通过 Python 代码解析 `terraform.tfstat`，获取 apply 结果，再进行进一步操作。

这显然是一个很麻烦的过程。**其中最主要的原因，是 terraform 只做到了「基础设施即配置」，而「配置」过于简单。**

这种情况下，就需要用到真正的「基础设施即代码」工具 - Pulumi 了。它的优势如下：

1. pulumi 是目前最流行的 真-IaaS 工具（另一个是刚出炉没多久的 terraform-cdk），对各语言的支持最为成熟。
2. 兼容 terraform 的所有 provider，只是需要自行使用 [pulumi-tf-provider-boilerplate](https://github.com/pulumi/pulumi-tf-provider-boilerplate) 重新打包，有些麻烦。
   1. 我翻文档发现，pulumi 的官方 provider 里面，估计 90% 以上都是 `based on the terraform xxx provider`...
3. 状态管理和 secrets 管理有如下几种选择：
   1. 使用 app.pulumi.com（默认）:免费版提供 stack 历史管理，可以看到所有的历史记录。另外还提供一个资源关系的可视化面板。总之很方便。
   2. 本地文件存储：`pulumi login file:///app/data`
   3. 云端对象存储，目前貌似只支持 aws-s3/gcp/azure 三种。
   4. [gitlab 13 支持 Terraform HTTP State 协议](https://github.com/pulumi/pulumi/issues/4727)，等这个 pr 合并，pulumi 也能以 gitlab 为 backend 了。
   5. 使用 pulumi 企业版（自建服务）：比 app.pulumi.com 提供更多的特性，但是显然是收费的。。

上述工具支持通过 Python/TypeScript 等语言来描述配置。好处有：

1. 使用代码编写 Kubernetes 配置，no-yaml
   1. yaml 也存在和 HCL 一样的问题，配置太死板，导致我们现在需要通过 Python 来动态生成 yaml...
2. 批量创建资源，动态生成资源参数。
   1. 比如批量创建一批名称类似的 ECS 服务器/VPC交换机。如果使用 terraform，你需要编写 module 来实现配置的复用，然后使用 hcl 的特殊语法来动态拼接出资源名称，因为语法限制，这种 HCL 能实现的功能也很有限。
   2. 而使用 pulumi，Python/TypeScript 这类通用的编程语言，能满足你的一切需求，而且作为一个开发人员/DevOps，你应该对它们相当熟悉。


## 使用建议

1. 建议查看对应的 terraform provider 文档：pulumi 的 provider 基本都是封装的 terraform 版本，而且文档是自动生成的，比较难看懂，examples 也少。
2. stack: pulumi 官方提供了两种 stack 用法：[「单体」和「微-stack」](https://www.pulumi.com/docs/intro/concepts/organizing-stacks-projects/)
   1. 单体: one stack hold them all，通过 stack 参数来控制步骤。stack 用来区分环境 dev/pro 等。
   2. 微-stack: 每一个 stack 是一个步骤，所有 stack 组成一个完整的项目。
   3. 实际使用中，我发现「微-stack」模式需要使用到 pulumi 的 inter-stack dependencies，报一堆的错，而且不够灵活。因此目前更推荐「单体」模式。

我们最近使用 pulumi 完全重写了以前用 terraform 编写的云上配置，简化了很多繁琐的配置，也降低了我们 Python 运维代码和 terraform 之间的交互难度。
另外我们还充分利用上了 Python 的类型检查和语法检查，很多错误 IDE 都能直接给出提示，强化了配置的一致性和可维护性。

体验上，terraform 只是配置编写方式，以及状态管理有些不同。实际上都是通过同样的 provider 管理云上资源。
目前我们使用 pulumi/terraform，实现了云上环境（资源组、VPC专有网络、k8s集群、数据库、账号权限系统、负载均衡等等）的一键搭建与销毁。
不过由于阿里云 provider 暂时还：

1. 不支持管理 ASM 服务网格、DTS 数据传输等资源
2. OSS 等产品的部分参数也暂时不支持配置
3. 部分资源创建时间过长，比如 ElasticSearch

这些问题，导致我们仍然有部分配置需要手动处理，另外一些耗时长的资源，需要单独去创建。
因此还不能实现完全的「一键」。


## 缺点

pulumi 和 terraform 都有一个缺点，就是封装层次太高了。

封装的层次很高，优点是方便了我们使用，可以使用很统一很简洁的声明式语法编写配置。
而缺点，则是出了 Bug 可能很难排查。不遇到 Bug 不要紧，一遇到 Bug，我们常常无能为力，只能等官方解决。。

