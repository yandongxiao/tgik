# 079: YTT and Kapp

## ytt(yaml templating tool):

1. templating(go's text/template, jinja):
   1. yaml引擎不识别模板中的内容，开发人员使用起来困难
   2. 开发人员无法编写重用的libraries
   3. helm template 其实是两种语言的混合形式

2. overlay tool (kustomize, bosh ops files)
   1. 缺少语言级别的特性，例如，条件判断, 循环

3. specialized configuration language (jsonnet, dhall)
   1. 一种完全新的语言
   2. 周边工具缺乏
4. ytt:
   1. 完全兼容yaml的语法，# 被征用，作为 ytt 的指令
   2. #@ 后面可以跟一个 python 语法的初始化
   3. yaml + python + structure aware

## kapp:

1. 使用同步等待的方式（kapp必须知道CRD中表示Ready状态的字段，这个是困难的），等待部署完成

2. 命令行工具的交互性，可以方便的看到版本之间的差异性

3. prune 功能的实现：依赖每次部署都需要一个唯一的标识，podTemplateHash只是template部分, kapp通过configmap来实现

4. adopt 功能：如果该应用一开始不是用 kapp 管理，后面也可以被 kapp 接管。反方向是否可以？

5. 安全边界控制：比如，只允许部署到某个namespace下。

6. log 功能：基于 Deployment 的 log功能，Deployment 部署时不中断log
7. -c: 显示 changed 信息

