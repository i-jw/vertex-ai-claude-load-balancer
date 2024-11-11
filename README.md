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

# change the project id in haproxy cfg

sed -i -e 's/PROJECT_ID_1/YOUR_PROJECT_ID/g' haproxy.cfg
sed -i -e 's/PROJECT_ID_2/YOUR_ANOTHER_PROJECT_ID/g' haproxy.cfg

# build image and push to repository
gcloud builds submit --region=$LOCATION --tag $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0 .
```

## deploy cloud run service
```shell
gcloud run deploy vertex-ai-claude-load-balancer \
 --container haproxy \
 --image='$LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/haproxy:v0 \
 --port='80'
```

### testing
```shell
curl --verbose -L -i -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @request.json \
"http://container-ip-address:port/v1"

```
