[English](README.md)
# vertex-ai-claude-load-balancer
使用容器做代理，将Claude API请求在多个区域之间负载均衡，遇到429时自动切换区域重试，同时支持更多项目加入负载均衡的列表
## 创建镜像仓库
```shell
# 启用服务
gcloud services enable artifactregistry.googleapis.com

# 创建镜像仓库
PROJECT_ID=YOUR_PROJECT_ID
LOCATION=YOUR_LOCATION_OF_REPO
REPOSITORY_NAME=REPOSITORY_NAME

gcloud artifacts repositories create $REPOSITORY_NAME --repository-format=docker \
    --location=$LOCATION --description="Docker repository" \
    --project=$PROJECT_ID
```

## 编译镜像
```shell
# 从github克隆代码
git clone https://github.com/i-jw/vertex-ai-claude-load-balancer.git
cd vertex-ai-claude-load-balancer

# 修改配置文件，替换项目ID

sed -i -e 's/PROJECT_ID_1/YOUR_PROJECT_ID/g' haproxy.cfg
sed -i -e 's/PROJECT_ID_2/YOUR_ANOTHER_PROJECT_ID/g' haproxy.cfg

# 编译并推送镜像到仓库
gcloud builds submit --region=$LOCATION --tag $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0 .
```

## 部署容器到cloud run
```shell
gcloud run deploy vertex-ai-claude-load-balancer \
 --container haproxy \
 --image='$LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0 \
 --port='80'
```

## 测试
```shell
curl --verbose -L -i -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @request.json \
"http://container-ip-address:port/v1"

```