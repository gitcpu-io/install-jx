# install jenkins-x for github 十步曲

## 第一步，准备安装仓库
git clone https://github.com/gitcpu-io/install-jx.git

### 设置访问域名
cd install-jx

vi jx-requirements.yml
```yaml
  ingress:
    domain: "gitpops.net"
```
git add .

git commit -am "change domain"

git push

## 第二步，准备oauth personal access token
> 先登录你的个人github账号，然后创建personal access token

https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,admin:repo_hook,write:packages,read:packages,write:discussion,workflow

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

## 第三步，通过安装jx-git-operator来安装jenkins-x

> 务必要安装在命名空间jx-git-operator中

kubectl create ns jx-git-operator

> 下载仓库

git clone https://github.com/jenkins-x/jx-git-operator.git

cd jx-git-operator/charts

- url 就是install-jx.git仓库

- username 就是你的github账号

- password 就是你的personal access token

helm -n jx-git-operator install --set url=https://github.com/gitcpu-io/install-jx.git --set username=rubinus --set password=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 jx-git-operator jx-git-operator

> 检查是否安装成功

kubectl -n jx-git-operator get po

kubectl -n jx get po

kubectl -n jx-observability get po

> 检查helm安装，如果install-jx有变动，可以升级

helm -n jx-git-operator list

cd jx-git-operator/charts

helm -n jx-git-operator upgrade --set url=https://github.com/gitcpu-io/install-jx.git --set username=rubinus --set password=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  jx-git-operator jx-git-operator


## 第四步，安装tekton-dashboard

> 如果访问不到镜像，可以执行下面的脚本用来替换

cd install-jx/install

./load_images.sh

kubectl apply -f tekton-ing.yaml

kubectl apply -f tekton-dashboard-release.yaml

> 配置所有域名解析

kubectl get ing -A


## 第五步，如果Oauth App token过期，可以使用Personal access token来创建

> 安装成功，需要删除 jx-git-operator，使用自己的仓库和prow的配置

kubectl delete ns jx-git-operator

> (可选)准备新的personal access token，来生成lighthouse-oauth-token

kubectl -n jx delete secret lighthouse-oauth-token

kubectl -n jx create secret generic lighthouse-oauth-token --from-literal=oauth=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


### 重启
kubectl -n jx scale deploy lighthouse-keeper --replicas=0

kubectl -n jx scale deploy lighthouse-foghorn --replicas=0

kubectl -n jx scale deploy lighthouse-tekton-controller --replicas=0

kubectl -n jx scale deploy lighthouse-webhooks --replicas=0

### 重启
kubectl -n jx scale deploy lighthouse-keeper --replicas=1

kubectl -n jx scale deploy lighthouse-foghorn --replicas=1

kubectl -n jx scale deploy lighthouse-tekton-controller --replicas=1

kubectl -n jx scale deploy lighthouse-webhooks --replicas=1


## 第六步，配置tekton pipeline的资源，作为presubmits和postsubmits
cd install-jx/install

kubectl -n jx apply -f ./pipeline

> 在这一步，一定要去修改为正确的secret中的账号密码（dockerhub、github）

kubectl -n jx apply -f ./resources

kubectl -n jx apply -f ./task

> 安装nfs client，nfs-client.yaml中的配置要改成nfs server的相应信息(如果需要)

kubectl -n jx apply -f ./storage


## 第七步，配置Prow

> 以jx-demo为例，作为chatops的代码仓库

https://github.com/gitcpu-io/jx-demo.git

### config的ConfigMap
cd install-jx/install

kubectl -n jx delete cm config

kubectl -n jx create cm config --from-file=config.yaml

### plugins的ConfigMap
cd install-jx/install

kubectl -n jx delete cm plugins

kubectl -n jx create cm plugins --from-file=plugins.yaml

## 第八步 搞定显示color label

cd install-jx/install

> 创建label-config的ConfigMap

kubectl -n jx apply -f labels-cm.yaml

> 运行job

kubectl -n jx delete -f label_sync_job.yaml

kubectl -n jx apply -f label_sync_job.yaml

kubectl -n jx delete -f label_sync_cron_job.yaml

kubectl -n jx apply -f label_sync_cron_job.yaml

## 第九步，安装argocd
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

> 备用

cd install-jx/install

kubectl -n argocd apply -f argocd-install.yaml

> 创建argocd ingress

kubectl -n argocd apply -f argocd-ing.yaml

> 设置域名解析

> argocd的初始密码

kubectl -n argocd get secret argocd-initial-admin-secret -oyaml

echo xxxxxxx |base64 -d

> 登录后修改密码，另外创建argocd-oauth的secret给pipeline使用

kubectl -n argocd apply -f ./resources/secret.yaml

### 登录argocd后，创建app

> 通过jx-demo-infra这个配置仓库

https://github.com/gitcpu-io/jx-demo-infra.git


# 第十步，使用Github App（如果需要）

## 如果要显示成GitHub App的账号信息，就需要生成 github-app install-access-token

## 编辑lighthouse的values.yaml.gotmpl

cd install-jx

vi versionStream/charts/jxgh/lighthouse/values.yaml.gotmpl

添加以下部分，使用githubApp
```yaml
githubApp:
  enabled: true
  username: "{{.Values.jx.secrets.pipelineUser.username}}"
```

## 准备jwt来访问api.github.com

> 其原理就是通过github.com上下载的GitHub App的private_key.pem
> 从.pem中提取出私匙的文本信息
> 连同GitHub App id 及过期时间，一起通过jwt.encode加密后生成的jwt token

### 生成private_key
yum install -y ruby

vi private_key.rb
```
require 'openssl'
private_pem = File.read("private_key.pem")   ##这里是GitHub App创建后下载的private_key.pem
private_key = OpenSSL::PKey::RSA.new(private_pem)
puts private_key
```

ruby private_key.rb

输入结果就是private_key的值，copy一下

### 打开 https://jwt.io/#debugger-io
1.选择RS256，HEADER:ALGORITHM部分自动出现，不用动

2.PAYLOAD:DATA 需要注意，用iss表示GitHub App的id，exp值需要最大10分钟的过期时间
```json
{
  "exp": 1640316991,
  "iss": "158861",
  "iat": 1516239022
}
```

3.VERIFY SIGNATURE部分在 private key部分，放入之前生成的private-key

4.复制左边的jwt的值

### 用得到的jwt token设置变量
```shell
GITHUB_JWT_TOKEN=\
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NDIwMzcyOTYsImlzcyI6IjE1ODg2MSIsImlhdCI6MTUxNjIzOTAyMn0.rUEbPYhbbI0WPoLwb64GlvecKF_nx6D-6te8n7hYkvoJxJYpflrRJlUtE57qXLKHPfGBdhSgInJ5BIa1VAJfTEifEQnsg_nkhqDMbfq5qsa1cqQi3Uu-POEknlhfs_qU5hrpm8WJBRaOagMwUwabUoKP89zFzBLecyoTFuX3IOPjUtoSIBs8z2q5WmwIC4GQJ4vdA-27tyAA_b2ORPgcvxSTNDpRmgc1DzHCEHEZjy3p7VV6fu0tLrIJ6G_6MIzakRNTnknb0dTrmjwJ0mDjaKGRkibmv07ArXF-3eZDVHx308NIHOo3_XwP8KrRkZFhq9L9Se8zbcjoN5EW7Ns_Fg

echo $GITHUB_JWT_TOKEN
```

### 获取已安装到这个Github APP下所有的repo，[].id 就是repo的安装id
```shell
curl \
-H "Authorization: Bearer $GITHUB_JWT_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/app/installations

"id": 21510467
这个id是Github app的ID 需要找到你自己的
```

### 通过安装 id 来查询对应的仓库：install access token

```shell
curl \
-X POST \
-H "Authorization: Bearer $GITHUB_JWT_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/app/installations/21510467/access_tokens \
-d '{"repositories":["jx-demo"]}'

21510467这个id是Github app的ID 需要用你自己上一步查询到的
这里的jx-demo需要换成你的仓库 也就是代码仓库
```

比如生成的 install access token = ghs_GESyuBbmGYPSIc0BvAsIKR5KhpjJIO1TIhoE

### 最后通过install-access-token生成git hub app token
kubectl -n jx delete secret tide-githubapp-tokens

kubectl -n jx create secret generic tide-githubapp-tokens --from-literal=username=gitcpu-io-jx-bot --from-literal=gitcpu-io-jx-bot=https://github.com/gitcpu-io=ghs_GESyuBbmGYPSIc0BvAsIKR5KhpjJIO1TIhoE

> 如果keeper、foghorn、webhooks的deployment没有下面部分，可以添加

```yaml
        ####进程启动参数flag添加
        - name: GITHUB_APP_SECRET_DIR
          value: /secrets/githubapp/tokens

        ####容器参数添加volumeMounts
        volumeMounts:
          - mountPath: /secrets/githubapp/tokens
            name: githubapp-tokens
            readOnly: true

      ####Pod挂载volumes
      volumes:
      - name: githubapp-tokens
        secret:
          defaultMode: 420
          secretName: tide-githubapp-tokens
```

### 重启
kubectl -n jx scale deploy lighthouse-keeper --replicas=0

kubectl -n jx scale deploy lighthouse-foghorn --replicas=0

kubectl -n jx scale deploy lighthouse-tekton-controller --replicas=0

kubectl -n jx scale deploy lighthouse-webhooks --replicas=0

### 重启
kubectl -n jx scale deploy lighthouse-keeper --replicas=1

kubectl -n jx scale deploy lighthouse-foghorn --replicas=1

kubectl -n jx scale deploy lighthouse-tekton-controller --replicas=1

kubectl -n jx scale deploy lighthouse-webhooks --replicas=1


# Github

## 关于token
- Personal access token 主要用于非生产环境，git push
- Oauth APP Token 次之，包括Personal token，可以在用作cli上提交代码git push
- Github APP 的 install access token功能最大，包括oauth token，但不能用作git push
- 都可以访问Github API
- Person token 和 Oauth token不能操作 Github app的相应api
- 任何用户创建的issue和pr，可以被其它用户来修改，前提是token有相应的权限：合作者且Write权限


## 关于PR
- 任何一个PR都是一个issue，但不是每一个issue都是一个PR
- A提交的PR，可以由B其它人来修改PR，合作者且Write权限
- PR不要太大，提交小的commit
- 如果PR没有reviewer，可以通过/assign @reviewer
- PR的commit第一个字母大写，标题不要超过50个字符；内容每行不超过72个字符

## 关于文档错误
- https://docs.github.com/cn/rest/reference/pulls#list-review-comments-on-a-pull-request

> 错误 - > 查询PR的comment
```shell
curl \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/pulls/16/comments

```

> 正确 pulls -> issues
```shell
curl \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/issues/16/comments

```

- https://docs.github.com/cn/rest/reference/pulls#create-a-review-comment-for-a-pull-request

> 错误 - > 给PR添加comment
```shell
curl \
-X POST \
-H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/pulls/16/comments \
-d '{"body":"add comment by install access token for PR"}'

```

> 正确  pulls -> issues
```shell
curl \
-X POST \
-H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/issues/16/comments \
-d '{"body":"add comment by install access token for PR"}'

```

# Personal access token
- 用于cli下，git push，相当于密码
- 做为授权token，可访问github api

> 设置变量
```shell
PERSONAL_ACCESS_TOKEN=\
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

```

> 创建issue
```shell
curl \
  -X POST \
  -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues \
  -d '{"title":"add issue by personal access token","body":"这是通过personal-access-token创建的"}'

```

> 使用personal access token修改其它token创建的issue
```shell
curl \
  -X PATCH \
  -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues/comments/1000977157 \
  -d '{"body":"modify comment by personal access token for other token commit"}'

```

> 使用other person create personal access token修改其它用户token创建的issue
```shell
curl \
-X PATCH \
-H "Authorization: Bearer $OTHER_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/issues/comments/1000977157 \
-d '{"body":"modify comment by other person personal access token for other token commit"}'

```


# Oauth App
> OAuth token对GitHub API进行身份验证，并包括 personal access token功能

> oauth token过期时间8小时，刷新token是6个月

> 设置变量
```shell
client_id=\
bdd9dda1c0dfda725fb4

client_secret=\
04dce2b8c7d5d8e788d60cb117de201cf33370e8

```

加上网页返回的code一起生成 oauth token

> user-to-server OAuth access token

## oauth 2.0测试流程三步曲
- 第一步：用户确认授权

  https://github.com/login/oauth/authorize?scope=user,public_repo&client_id=bdd9dda1c0dfda725fb4&redirect_uri=http://baidu.com&response_type=code&state=1

  如果是mac用户，直接在terminal中输入：

  open https://github.com/login/oauth/authorize?scope=user,public_repo&client_id=$client_id&redirect_uri=http://baidu.com&response_type=code&state=1

- 第二步：Github跳转到之前设定好的回调地址，并带着code

  code只有一次有效，10分钟，重新执行第一步，会使code失效：

  https://www.baidu.com/?code=60307876548fdcd71c80&state=1

> 设置变量
```shell
code=\
609cc0153a5d72011545

```

- 第三步：后台服务通过code，调用Github api换取oauth token，这一步必须用到 client_secret

open https://github.com/login/oauth/access_token?client_id=$client_id&client_secret=$client_secret&code=$code

```shell
curl \
-H "Accept: application/json" \
https://github.com/login/oauth/access_token?client_id=$client_id&client_secret=$client_secret&code=$code

返回结果
{"access_token":"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","token_type":"bearer","scope":"public_repo,user"}

```

## 验证Oauth token

> 设置变量
```shell
GITHUB_OAUTH_TOKEN=\
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

curl \
  -H "Authorization: token $GITHUB_OAUTH_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user

curl \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/pulls

```

> REPO=github账号名/仓库名

> 使用oauth token创建issue
```shell
curl \
  -X POST \
  -H "Authorization: Bearer $GITHUB_OAUTH_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues \
  -d '{"title":"add issue by oauth token","body":"这是通过oauth-token创建的"}'

```

> 使用oauth token修改其它token创建的issue
```shell
curl \
  -X PATCH \
  -H "Authorization: Bearer $GITHUB_OAUTH_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues/comments/1000977157 \
  -d '{"body":"modify comment by oauth access token for other token commit - 101"}'

```

# GitHub App
### GitHub App id 一般就是6位数字

### GitHub App pem (private key) 需要保存到文件目录中

## GitHub App通过install与其它repo勾搭上

> install access token：是GitHub App访问已经install过的任何repo，调用github api请求对repo操作时，

> 需要用经过身份验证的token，这个token会在一小时后过期，需要定期提前更新

## install-access-token生成步骤：
1.从Github APPs下载保存到文件目录：private_key.pem

2.从private_key.pem中提取私匙部分（通过算法）比如下脚本

保存并运行：ruby private_key.rb 以下是内容
```
require 'openssl'
private_pem = File.read("private_key.pem")
private_key = OpenSSL::PKey::RSA.new(private_pem)
puts private_key
```
3.连同过期时间（最大10分钟）和 GitHub App id做为 jwt的playload
- Playload部分,exp=当前时间 + 10分钟，生成的10位unix时间戳
```json
{
"exp": "1640334864",
"iss": "158861",
"iat": 1516239022
}
```

4.用jwt.encode来加密，最后生成jwt token
```shell
JWT.encode(payload, private_key, "RS256")
```

5.打开 https://jwt.io/#debugger-io

6.调用github api生成install-access-token

https://docs.github.com/cn/rest/reference/apps#create-an-installation-access-token-for-an-app

## 用得到的jwt token设置变量
```shell
GITHUB_JWT_TOKEN=\
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOiIxNjQwNTg3MjkzIiwiaXNzIjoiMTU4ODYxIiwiaWF0IjoxNTE2MjM5MDIyfQ.PGeSR1pcfWAtIuvn1OsO2g4z-TUdVQ7nIOu52QPqtgLCbFXHn1Wb5D9gvuRdtOwS3dpbGzy9_dEXditUdtDG5GF57KSm7eXpTrE0Gz3r-akOOL12yJv4ie2o4G-XnPcQ2PylNAyCArA0YKfD6ag83Bnb82SYzMptB0jI3uT0NOrWQc_NdBc3uJEClTp1O-FqgHvccQ6efM5I4lyxqyDm72SAgNaU4Ng_ZIraSNjiLKuwNIsqslSBXJj1Jkn8tDode5PwVv52TUUuNSuTJp9xVcCLfBzHgJJfSa0x3j_e3U98aumdlvg3eVpVvyH6wKgH-YRYC1zwS_eEuhUn7dpEZA

echo $GITHUB_JWT_TOKEN
```

## 验证创建的Git app信息
```shell
curl \
-H "Authorization: Bearer $GITHUB_JWT_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/app

```

## 获取已安装到这个Github APP下所有的repo，[].id 就是repo的安装id
```shell
curl \
-H "Authorization: Bearer $GITHUB_JWT_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/app/installations

"id": 21510467

```

## 通过安装 id 来查询对应的：install access token

```shell
curl \
-X POST \
-H "Authorization: Bearer $GITHUB_JWT_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/app/installations/21510467/access_tokens \
-d '{"repositories":["jx-demo"]}'

```

比如生成的 install access token = ghs_JSjJO1wSguc0ChEjhA5mVpefY3D80S30W71J

## 这个token的过期时间是1个小时，需要及时更新

## 你可以使用这个过期是1小时的token来 调用Github Api 如下是验证操作：

> 设置变量
INSTALL_ACCESS_TOKEN=\
ghs_JSjJO1wSguc0ChEjhA5mVpefY3D80S30W71J

> 设置repo变量
REPO=\
gitcpu-io/jx-demo

> 下面的 PR 和 issue 的 ID 需要先创建出来

>> 创建issue
```shell
curl \
  -X POST \
  -H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues \
  -d '{"title":"add issue by install access token","body":"这是通过install-access-token创建的"}'

```

>> 修改PR
```shell
curl \
-X PATCH \
-H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/pulls/16 \
-d '{"title":"Modify PR by install access token","body":"这是通过install-access-token修改的"}'

```

>> 查询本次PR的文件变动，默认一页30条
```shell
curl \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/pulls/16/files \

```

>> 添加comment for issues
```shell
curl \
-X POST \
-H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/issues/1/comments \
-d '{"body":"add comment by install access token for issues"}'

```

>> 添加comment for PR 和 for issues 是相同的API
```shell
curl \
-X POST \
-H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
-H "Accept: application/vnd.github.v3+json" \
https://api.github.com/repos/$REPO/issues/16/comments \
-d '{"body":"add comment by install access token for PR"}'

```

>> 查询comment id，无需授权token (issue和 PR 用相同的api查询)
```shell
curl \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues/16/comments

取一个comment id = 1000976560
```

>> 修改comment for PR
```shell
curl \
  -X PATCH \
  -H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues/comments/1000977344 \
  -d '{"body":"modify comment by install access token for PR"}'

```

>> 修改comment for issue
```shell
curl \
  -X PATCH \
  -H "Authorization: Bearer $INSTALL_ACCESS_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/$REPO/issues/comments/1000977157 \
  -d '{"body":"modify comment by install access token for issue"}'

```



# OWNERS
每一个文件目录都可以有一个OWNERS文件，子目录继承父目录的reviewer/approver

## PR
每一个PR首先自动选择至少2个reviewer，再选择唯一的一个reviewer

每个reviewer在输入lgtm后，再 /assign @approver（这里是一个具体的approver的名字）

## reviewer
- 选择潜在的reviewer

从当前目录与父目录的OWNERS文件中选择并去重

- 给文件分配权重

根据更改的代码行数为每一个文件分配权重，改动越多，权重越大

- 给reviewer分配权重

相加所有更改文件的权重，分配给reviewer

- 根据权重随机选择 2 个reviewer

1.必须是OWNERS文件中的reviewers

2.创建PR的人不能做为reviewer

3.在评论中 输入 /lgtm 或 /lgtm cancel 来取消

4.prow会为PR打上lgtm标签

## approver
- approver可以批准当前目录与子目录的文件

- 从所有的approvers中选择一个子集

- 最好选择一个即是approver同时又是reviewer

- 有时从当前目录，有时从父目录中选择

### 扩充approvers []的算法
- 如果PR作者执行/approve，那么需要 /assign 给其它approver

- 如果不是PR的approver，执行了/approve，那么还需要 /assign

- 如果PR中有2个文件目录，只是其中一个文件目录的approve，那到还需要 /assign

- 如果PR的approver，却执行了/lgtm，那么自动添加lgtm标签，并还需要 /assign


1.必须是OWNERS文件中的approvers

2.可以是创建PR的人

3.prow会选择性的选取一个approver，等待他输入/approve到评论中

4.如果/approve之后，还可以/approve cancel

5.prow会为PR打上approved标签

## PR合并之前
1.必须有 lgtm 和 approved 标签

2.必须没有 do-not-merge 和 hold 标签

3.如果有presubmit,那么必须运行过
