# 使用步骤

## 1、申请 ChatGLM

https://open.bigmodel.cn/

进入控制台，点击界面右上角【API密钥】进行添加


## 2、创建日志仓库
https://github.com/xxxx/openai-code-review-log  -- 创建你自己的

## 3、申请 GitHub Token
地址：https://github.com/settings/tokens

记住！！！申请完一定要先复制！！！

![image](https://github.com/user-attachments/assets/5b2698d0-5110-44e2-a15e-357c7c6d86eb)

![image](https://github.com/user-attachments/assets/92550f09-43b5-4b63-bc44-441b9c1ebe0f)

## 4、申请公众号测试平台
申请地址 https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index
扫描登录就行，进入后创建消息模板就行，其他的忽略掉

### 4.1 模板消息接口

模板内容：

项目：{{repo_name.DATA}} 

分支：{{branch_name.DATA}} 

作者：{{commit_author.DATA}} 

说明：{{commit_message.DATA}}

![image](https://github.com/user-attachments/assets/cb1a93d0-6ea5-4b4a-a332-d0188a65b549)


## 5、GitHub Actions 配置
### 5、1 配置参数
进入你自己的的工程仓库（注意！！！是工程仓库，你的项目仓库，不是日志仓库），点击 Setting
![image](https://github.com/user-attachments/assets/4d340c43-7bf9-439d-b80e-43cf9d00fdb9)

分别添加以下配置项：

CHATGLM_APIHOST：https://open.bigmodel.cn/api/paas/v4/chat/completions

CHATGLM_APIKEYSECRET： -- 使用你自己申请的 ChatGLM 密钥

CODE_REVIEW_LOG_URI： -- 使用你自己的日志库地址，例如：https://github.com/xxxx/openai-code-review-log

CODE_TOKEN：  -- 使用你自己申请的 GitHub Token （步骤3中的）

WEIXIN_APPID：  -- 使用你自己的公众号测试平台的  【appID】

WEIXIN_SECRET：  -- 使用你自己的公众号测试平台的  【appsecret】

WEIXIN_TEMPLATE_ID：  -- 使用你自己的公众号测试平台的 【消息模板ID】

WEIXIN_TOUSER：  -- 使用你自己的公众号测试平台的 【用户列表中的微信号】

## 6、工作流
### 6、1 创建工作流
进入你自己的的工程仓库（注意！！！是工程仓库，你的项目仓库，不是日志仓库），点击 Actions

![image](https://github.com/user-attachments/assets/c30cb7ca-ae33-4478-a47e-820ab029aad2)

### 6、2 写入脚本并提交
``` xml
# 将 sdk release，使用远程下载 jar 的方式，不需要每次都通过 mvn 构建
# https://github.com/LinJiangXian-17/openai-code-review-log/releases/download/1.0/openai-code-review-sdk-1.0.jar
name: Run Java Git Diff By DownLoad Remote JAR

on:
  push:
    branches:
      - '*'  # 需要时打开，master-close 改成 master 或者 '*'
  pull_request:
    branches:
      - '*'  # 需要时打开，master-close 改成 master 或者 '*'

jobs:
  build-and-run:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
        with:
          fetch-depth: 2  # 检出最后两个提交，以便可以比较 HEAD~1 和 HEAD

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt'  # 你可以选择其他发行版，如 'adopt' 或 'zulu'
          java-version: '11'

      # 创建依赖库目录
      - name: Create libs directory
        run: mkdir -p ./libs

      # mvn 构建换成下载远程 jar，需要 release jar
      - name: Download openai-code-review-sdk JAR
        run: wget -O ./libs/openai-code-review-sdk-1.0.jar https://github.com/LinJiangXian-17/openai-code-review-log/releases/download/1.0/openai-code-review-sdk-1.0.jar

      # 获取工程名称
      - name: Get repository name
        id: repo-name
        run: echo "REPO_NAME=${GITHUB_REPOSITORY##*/}" >> $GITHUB_ENV

      # 获取分支名称
      - name: Get branch name
        id: branch-name
        run: echo "BRANCH_NAME=${GITHUB_REF#refs/heads/}" >> $GITHUB_ENV

      # 获取提交者名称
      - name: Get commit author
        id: commit-author
        run: echo "COMMIT_AUTHOR=$(git log -1 --pretty=format:'%an <%ae>')" >> $GITHUB_ENV

      # 获取提交信息
      - name: Get commit message
        id: commit-message
        run: echo "COMMIT_MESSAGE=$(git log -1 --pretty=format:'%s')" >> $GITHUB_ENV

      - name: Print repository, branch name, commit author, and commit message
        run: |
          echo "Repository name is ${{ env.REPO_NAME }}"
          echo "Branch name is ${{ env.BRANCH_NAME }}"
          echo "Commit author is ${{ env.COMMIT_AUTHOR }}"
          echo "Commit message is ${{ env.COMMIT_MESSAGE }}"

      - name: Run Code Review
        run: java -jar ./libs/openai-code-review-sdk-1.0.jar
        env:
          # Github 配置；GITHUB_REVIEW_LOG_URI「https://github.com/LinJiangXian-17/openai-code-review-log」、GITHUB_TOKEN「https://github.com/settings/tokens」
          GITHUB_REVIEW_LOG_URI: ${{ secrets.CODE_REVIEW_LOG_URI }}
          GITHUB_TOKEN: ${{ secrets.CODE_TOKEN }}
          COMMIT_PROJECT: ${{ env.REPO_NAME }}
          COMMIT_BRANCH: ${{ env.BRANCH_NAME }}
          COMMIT_AUTHOR: ${{ env.COMMIT_AUTHOR }}
          COMMIT_MESSAGE: ${{ env.COMMIT_MESSAGE }}
          # 微信配置 「https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index」
          WEIXIN_APPID: ${{ secrets.WEIXIN_APPID }}
          WEIXIN_SECRET: ${{ secrets.WEIXIN_SECRET }}
          WEIXIN_TOUSER: ${{ secrets.WEIXIN_TOUSER }}
          WEIXIN_TEMPLATE_ID: ${{ secrets.WEIXIN_TEMPLATE_ID }}
          # OpenAi - ChatGLM 配置「https://open.bigmodel.cn/api/paas/v4/chat/completions」、「https://open.bigmodel.cn/usercenter/apikeys」
          CHATGLM_APIHOST: ${{ secrets.CHATGLM_APIHOST }}
          CHATGLM_APIKEYSECRET: ${{ secrets.CHATGLM_APIKEYSECRET }}
```

## 7、大功告成
修改你的工程代码并提交测试一下，工作流执行结束后会有公众号消息通知以及在你的日志仓库中会生成评审日志文件
