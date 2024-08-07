##  Apps Json 
-------------------
Test2
1
```shell
export APPS_JSON='[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://username:password@github.com/username/<CUSTOM_APP>.git",
    "branch": "main"
  }
]'

export APPS_JSON_BASE64=$(echo ${APPS_JSON} | base64 -w 0)
```

You can also generate base64 string from json file:

```shell
export APPS_JSON_BASE64=$(base64 -w 0 apps2.json)
```

## Build Your Image

docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15-hotfix \
  --build-arg=PYTHON_VERSION=3.11.6 \
  --build-arg=NODE_VERSION=18.18.2 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=menaheero/heerocrm:V9 \
  --file=images/custom/Containerfile .

## Update Main Compose File

Modify the compose.yaml file inside the frappe_docker folder like this

x-customizable-image: &customizable_image
  image: heerocrm:V3
  pull_policy: never


# Gitops

Create a directory with the name of gitops

mkdir ~/gitops

## TRAEFIK ENV
Here Im assume im using the domain alltargeting.com but its not exist in real. you can change your domain where ever you see alltargeting.com the reason instead of giving example.com I’m giving this. but in one file example.com will be there we have to correct there that’s why I’m giving some random domain example keep in mind

echo 'TRAEFIK_DOMAIN=test.alltargeting.com' > ~/gitops/traefik.env
echo 'EMAIL=admin@alltargeting.com' >> ~/gitops/traefik.env
echo 'HASHED_PASSWORD='$(openssl passwd -apr1 changeit | sed 's/\$/\\\$/g') >> ~/gitops/traefik.env

## Up the traefik container
Up the traefik container

docker compose --project-name traefik \
  --env-file ~/gitops/traefik.env \
  -f overrides/compose.traefik.yaml \
  -f overrides/compose.traefik-ssl.yaml up -d

## MariaDB
Note: change the DB_PASSWORD with some more secure basically this command creates a file inside the gitops folder with the name of mariadb.env if not create create manually add the DB_PASSWORD=changeit

echo "DB_PASSWORD=superh2soc" > ~/gitops/mariadb.env

## Run MariaDB Container
Make sure running these all commands inside the frappe_docker folder
Deploy the mariadb container

docker compose --project-name mariadb --env-file ~/gitops/mariadb.env -f overrides/compose.mariadb-shared.yaml up -d

## Step 8
Please do this step very carefully change the required data and run the command
change your domain and DB Password here you can see the erp.example.com don’t change that one only change the domain where you see the alltargeting.com. please do it carefully

cp example.env ~/gitops/erpnext-one.env
sed -i 's/DB_PASSWORD=123/DB_PASSWORD=superh2soc' ~/gitops/erpnext-one.env
sed -i 's/DB_HOST=/DB_HOST=mariadb-database/g' ~/gitops/erpnext-one.env
sed -i 's/DB_PORT=/DB_PORT=3306/g' ~/gitops/erpnext-one.env
sed -i 's/SITES=`erp.example.com`/SITES=\`test.alltargeting.com\`/g' ~/gitops/erpnext-one.env
echo 'ROUTER=erpnext-one' >> ~/gitops/erpnext-one.env
echo "BENCH_NETWORK=erpnext-one" >> ~/gitops/erpnext-one.env


## Step Create erpnext-one.yaml


docker compose \
  --project-name erpnext-one \
  --env-file ~/gitops/erpnext-one.env \
  -f compose.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.multi-bench.yaml \
  -f overrides/compose.multi-bench-ssl.yaml \
  config > ~/gitops/erpnext-one.yaml


## Step 10
Deploy erpnext-one containers:

docker compose --project-name erpnext-one -f ~/gitops/erpnext-one.yaml up -d

## Step 11
Create site alltargeting.com 
Change your site instead of alltargeting.com

In this command the site password is changei and DB password is changeit so please change those and make it secure. where ever you see changeit in the command change with a secure one.


docker compose --project-name erpnext-one exec backend \

bench new-site alltargeting.com --no-mariadb-socket --mariadb-root-password superh2soc --install-app erpnext --admin-password superh2soc




To enable sceduler login inside the backend container using this command

docker exec -it <CONTAINER_NAME> bash
example

docker exec -it erpnext-one-backend-1 bash
Before enabling the scheduler if you have a backup database restore it. After Enable Schedular

Here the enabling command

bench --site alltargeting.com enable-scheduler
If site not working
Check the the site is working if not working down and up the container



docker compose --project-name erpnext-one -f ~/gitops/erpnext-one.yaml down


docker compose --project-name erpnext-one -f ~/gitops/erpnext-one.yaml up -d