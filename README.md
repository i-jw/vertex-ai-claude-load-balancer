[中文版](README_zh.md)
# vertex-ai-claude-load-balancer
load balance two account cross two region , also able to add more project in config file
### build docker image
```shell
# clone src from github
git clone https://github.com/i-jw/vertex-ai-claude-load-balancer.git
cd vertex-ai-claude-load-balancer

# change the project id in haproxy cfg

sed -i -e 's/PROJECT_ID_1/YOUR_PROJECT_ID/g' haproxy.cfg
sed -i -e 's/PROJECT_ID_2/YOUR_ANOTHER_PROJECT_ID/g' haproxy.cfg

# build image 
docker build -t vertex-ai-claude-load-balancer .




```
### testing
```shell
curl --verbose -L -i -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @request.json \
"http://container-ip-address:port/v1"

```
