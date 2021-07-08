[TOC]

# Introduction to Backstage (12.3k star) 

## 什么是 Backstage？

1. Spotify 是一家公司，随着公司的成长，开发人员的效率开始降低。一个重要原因是，效率工具分散、也没有统一的标准。当开发人员遇到问题时（比如构建问题、发布问题等），他们无法通过效率工具，有效地解决问题。
2. 他们开发了一个"大前端"（包括：CICD、kubernetes event 等），作为开发人员的唯一入口。它也是一个 App Store，通过插件机制，集成现有的效率工具。最终，为开发人员提供一致的体验。
3. It create a new & better  standard for engineering organizations everywhere

## 通过 Backstage 创建服务

1. 你可以定义自己的 Home Page。
2. 针对前端服务，后端服务，数据服务等，Backstage 都有预先定义好的模板。通过向导的方式，你创建的服务会自动包含模板提供的特性。以 Apollo Service 模板创建的服务为例，它会有gRPC、Http、Deployment、loggin and metrics 等功能。向导会自动在 github 上，创建应用，设置 CICD 等。你只需要 Clone 代码，开发即可。
3. **在公司内部，可以对 Template 进行修改。**

## 通过 Backstage 管理服务

1. 每个 Team 有自己的 Team Page。
2. 通过 Backstage 的插件机制，你可以自定义 Metric 的含义，包括：cloud cost，security issues 等。
3. Ownership：如果你是团队中某个服务的负责人，你需要维护服务，oncall for your service。
4. Upgrades & Migration。负责服务的升级

![image-20210606123015169](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210606123015169.png)

## TODO

个人感觉，公司内部的各个系统，都在 Backstage 中涵盖了。但是，将它们集成到 Backstage 是困难的。不是技术上的问题。