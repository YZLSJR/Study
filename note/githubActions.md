---
tag: 部署
---

## 1. GitHub Actions

> 持续集成/持续部署（CI/CD）流程

:thinking:现在我的认知是：在多人开发的时候，用docker给每个人赋予相同的开发环境。代码上传到`GitHub`上。到项目部署阶段，再用自动化部署工具。

* 尝试用了用`GitHub Actions`自动化部署挺方便的。

  * 可集成`build`，`deploy`，`test`等等功能
    - [ ] test
    - [ ] docker

* 关于配置等敏感信息

  * 在本地开发时，你可以直接在代码中或通过本地配置文件（如`.env`文件）设置必要的配置信息。确保这些文件不会被推送到远程仓库（使用`.gitignore`）。

  * 在GitHub环境中，你可以使用GitHub Secrets或其他类似的机制来安全地存储这些信息。

  * 在代码中假如if判断语句，判断是否在本地

    - [ ] :heavy_exclamation_mark:误区： 到云服务器，还可以读取到GitHub secret吗——不可以

### :thought_balloon:`deploy.yml`的编写

* GitHub Actions 中的运行器（runner）理解为一个预配置的虚拟机环境（“空白的”），所以对于每个任务job中的每一个步骤都要加载

  ```yaml
  # 检出代码
  - uses: actions/checkout@v3
  ```

* GitHub Actions 中的上传和下载构件

  * **为什么需要上传和下载构件**？

    在 GitHub Actions 中，不同的作业（`jobs`）运行在隔离的环境中。如果您在一个作业中构建了一个应用程序，并希望在另一个作业中使用这个构建的应用程序，那么您需要使用构件（`artifacts`）来在作业之间传递数据。您首先在构建作业中上传构件，然后在需要使用这些文件的作业中下载构件。这是实现作业间数据共享的一种方式。

* 部署中该命令会覆盖旧的指定文件

  * ```yaml
    ARGS: "-rltgoDzvO --delete" # 部署参数，表示递归、压缩、保留权限、保留所有者、删除目标目录中源目录没有的文件
    ```

* 源代码

  * ```yacas
    # This workflow will build a golang project
    # For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-go
    
    # 命名
    name: Go
    
    # 触发条件
    on:
      push:
        branches: [ "main" ]
      pull_request:
        branches: [ "main" ]
    
    # 任务
    jobs:
      # 构建&测试
      build:
        # 运行环境
        runs-on: ubuntu-latest
        # 步骤
        steps:
          # 设置环境变量
        - name: Set environment variable
          run: echo "APP_ENV=production" >> $GITHUB_ENV
    
          # 检出代码
        - uses: actions/checkout@v3
    
          # 设置 Go 环境
        - name: Set up Go
          uses: actions/setup-go@v4
          
          # 指定 Go 版本
          with:
            go-version: '1.20'
    
          # 构建
        - name: Build
          run: go build -v ./...
    
          # 测试
        - name: Test
          run: go test -v ./...
      
      # 部署
      deploy:
        # 依赖于构建
        needs: build
        # 运行环境
        runs-on: ubuntu-latest
        # 步骤
        steps:
          # 检出代码
          - uses: actions/checkout@v3
          # 设置环境变量
          - name: Set environment variable from secret
            run: echo DSN=${{ secrets.DSN }} >> $GITHUB_ENV
          # 设置 Go 环境
          - name: Set up Go
            uses: actions/setup-go@v4
            with:
              go-version: '1.20'
          # 构建
          - name: Build
            run: go build -o voiceHanter
          # 部署
          - name: Deploy
            uses: easingthemes/ssh-deploy@v2.1.5
            env:
              SSH_PRIVATE_KEY: ${{ secrets.SERVER_SSH_KEY }} # 服务器私钥
              ARGS: "-rltgoDzvO --delete" # 部署参数，表示递归、压缩、保留权限、保留所有者、删除目标目录中源目录没有的文件
              SOURCE: "voiceHanter" # 需要部署的目录或文件
              REMOTE_HOST: ${{ secrets.SERVER_HOST }} # 服务器地址
              REMOTE_USER: ${{ secrets.SERVER_USER }} # 服务器用户名
              TARGET: ${{ secrets.SERVER_TARGET }} # 服务器目标目录
    
    ```

### :thought_balloon:**部署时如何处理** `secrets`

* **GitHub Actions 的 `secrets` 使用逻辑**
  - GitHub Actions 的 `secrets` 是为了在 GitHub 的 CI/CD 环境中安全地使用敏感信息而设计的。在 GitHub Actions 工作流中，您可以使用这些 `secrets` 来完成多种任务，比如构建 Docker 镜像、测试、部署等，而无需将敏感信息硬编码在代码中或公开暴露。

> 虽然 `secrets` 不能直接从 GitHub Actions 传递到您的生产服务器，但有几种方法可以安全地在部署过程中使用这些 `secrets`

#### 1.11 **环境变量注入**：

> **每次部署都需要设置**：您需要在每次部署时通过 GitHub Actions 脚本设置这些环境变量。这确保了敏感数据的安全，因为它们不会在服务器上以文件形式持久存储。

- 在部署脚本中，您可以将 `secrets` 作为环境变量注入到应用程序中。这意味着在部署过程中（如通过 SSH 运行远程命令），`secrets` 会临时作为环境变量设置，供应用程序读取。

  - :ear: **配置**`deploy_script.sh` **脚本**

    - `deploy_script.sh` 是一个在服务器上执行的脚本，它定义了部署过程中应该执行的步骤。这个脚本的作	用包括：

      - **接收环境变量**：脚本可以使用通过 SSH 命令传递的环境变量，如 `SECRET_VAR`。

      - **定义部署逻辑**：脚本中包含了将应用程序从代码库部署到生产环境所需的所有命令。这可能包括复制文件、运行数据库迁移、启动或重启服务等。

        :book:比如：
  
        - `sudo ./voiceHanter`
          - 超级用户（root）权限下，运行当前目录下的可执行文件` voiceHanter`
              * 一直在后台运行
                * `nohup sudo ./voiceHanter`
  
  - **示例场景**

    假设您的应用程序连接一个数据库来运行。在 GitHub Actions 工作流中，您通过 SSH 将数据库dns作为环境变量传递给 `deploy_script.sh` 脚本。脚本使用这个dns来配置应用程序或执行数据库相关的操作。一旦 SSH 会话结束，这个密码不会留在服务器上，确保了敏感信息的安全。

* 具体实现

  * 在 GitHub Actions 中通过环境变量注入 `secrets` 到应用程序中，通常涉及以下步骤：

    ##### **步骤 1: 在 GitHub 仓库中设置 Secrets**

    - 在 GitHub 仓库中，进入 Settings > Secrets。
    - 添加您需要的敏感数据作为 Secrets，比如 `DATABASE_PASSWORD` 或 `API_KEY`。

    ##### **步骤 2: 编写 GitHub Actions 工作流配置**

    * 编辑或创建一个 GitHub Actions 工作流 YAML 文件，比如 `.github/workflows/deploy.yml`。

    * **示例配置**：

      ```yaml
      yamlCopy codename: Deploy Application
      
      on:
        push:
          branches:
            - main
      
      jobs:
        deploy:
          runs-on: ubuntu-latest
          steps:
          - name: Checkout code
            uses: actions/checkout@v2
      
          - name: Deploy to Server
            run: ssh user@your_server_ip "export SECRET_VAR=${{ secrets.SECRET_VAR }}; ./deploy_script.sh"
            env:
              SECRET_VAR: ${{ secrets.SECRET_VAR }}
      ```

  
    * 解释:
  
      - 在 `deploy` 作业的 `run` 命令中，使用 SSH 命令连接到您的服务器，并在远程命令中设置环境变量。这里，`SECRET_VAR` 是通过环境变量传递给远程服务器的。
  
      - `${{ secrets.SECRET_VAR }}` 是从 GitHub Secrets 获取的值。
  
  
    **步骤 3: 在服务器上准备部署脚本**
  
    * 确保服务器上有一个脚本（如 `deploy_script.sh`），它可以使用这些环境变量。
      * 示例 `deploy_script.sh`:
  
    ```yaml
    bashCopy code#!/bin/bash
    
    # 使用环境变量
    echo "使用的密钥: $SECRET_VAR"
    
    # 在这里添加其他部署逻辑
    ```
  
    * 确保此脚本在服务器上具有执行权限 (`chmod +x deploy_script.sh`)。
  
    **安全性注意事项**
  
    - 确保 SSH 命令不会在日志中暴露敏感信息。
    - 使用 SSH 密钥进行认证，而非密码。

---

#### 1.12 **配置文件生成**：

- 在 GitHub Actions 工作流中，使用 `secrets` 生成配置文件（如 `.env`），然后将该文件传输到服务器。这样，应用程序可以在运行时读取这些配置。【已测试成功】
  * 会将`.env`留着服务器上。不如上面那个方法安全性高。

#### 1.13 **云服务密钥管理**：

- 使用云服务提供的密钥管理服务（如 AWS Secrets Manager、Azure Key Vault、Google Cloud Secret Manager）来存储 `secrets`。应用程序可以在运行时通过 API 调用这些服务来获取所需的 `secrets`。

#### 1.14**服务容器化**：

- 如果应用程序是以 Docker 容器形式部署的，可以在构建容器镜像时使用 `secrets`，然后将镜像推送到私有容器注册中心。部署时，直接从注册中心拉取镜像到生产环境。