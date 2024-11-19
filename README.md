[中文版](README_zh.md)
# vertex-ai-claude-load-balancer
load balance two account cross two region , also able to add more project in config file
## create repo
```shell
# enable artifactregistry service
gcloud services enable artifactregistry.googleapis.com

# crate repository
PROJECT_ID=YOUR_PROJECT_ID
LOCATION=YOUR_LOCATION_OF_REPO
REPOSITORY_NAME=REPOSITORY_NAME

gcloud artifacts repositories create $REPOSITORY_NAME --repository-format=docker \
    --location=$LOCATION --description="Docker repository" \
    --project=$PROJECT_ID
```
### build docker image
```shell
# clone src from github
git clone https://github.com/i-jw/vertex-ai-claude-load-balancer.git
cd vertex-ai-claude-load-balancer

# build image and push to repository
gcloud builds submit --region=$LOCATION --tag $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0 .
```
##  create project id map
```shell
cat << EOF > projects.map
num  3
0  PROJECT_ID_0
1  PROJECT_ID_1
2  PROJECT_ID_2
EOF

```
## deploy cloud run service
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

### testing
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
"http://container-ip-address:port/v1/projects/$PROJECT_ID/locations/$LOCATION/publishers/anthropic/models/$MODEL:streamRawPredict"


```
