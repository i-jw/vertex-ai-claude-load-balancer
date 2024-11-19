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

# 编译并推送镜像到仓库
gcloud builds submit --region=$LOCATION --tag $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0 .
```
## 创建project id map
```shell
cat << EOF > projects.map
num  3
0  PROJECT_ID_0
1  PROJECT_ID_1
2  PROJECT_ID_2
EOF

```
## 部署容器到cloud run
```shell

gcloud storage buckets create gs://BUCKET_NAME --location $LOCATION
gcloud storage buckets cp projects.map gs://BUCKET_NAME/claude-lb-folder/projects.map


gcloud run deploy vertex-ai-claude-load-balancer \
    --container haproxy \
    --region $LOCATION  \
    --port 80  \
    --image $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0  \
    --add-volume name=project-map,type=cloud-storage,bucket=BUCKET_NAME,mount-options="only-dir=claude-lb-folder" \
    --add-volume-mount volume=project-map,mount-path=/data
 
gcloud beta run services update haproxy-claude \
    --region $LOCATION  \
    --port 80  \
    --image $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0  \
    --add-volume name=project-map,type=cloud-storage,bucket=BUCKET_NAME,mount-options="only-dir=claude-lb-folder" \
    --add-volume-mount volume=project-map,mount-path=/data

```

## 测试
```shell
cat << EOF > request.json
{
  "anthropic_version": "vertex-2023-10-16",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/png",
            "data": "iVBORw0KGg..."
          }
        },
        {
          "type": "text",
          "text": "What is in this image?"
        }
      ]
    }
  ],
  "max_tokens": 256,
  "stream": true
}
EOF

PROJECT_ID=your-gcp-project-id
MODEL=claude-3-5-haiku@20241022
LOCATION=us-east5

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @request.json \
"https://xxxxxx.us-central1.run.app/v1/projects/$PROJECT_ID/locations/$LOCATION/publishers/anthropic/models/$MODEL:streamRawPredict"

```