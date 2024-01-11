SQLE部署



```bash
docker run -d -it \
--name sqle-server \
-p 10000:10000 \
-e MYSQL_HOST="10.10.10.10" \
-e MYSQL_PORT=3306 \
-e MYSQL_USER="username" \
-e MYSQL_PASSWORD="password" \
-e MYSQL_SCHEMA="sqle" \
actiontech/sqle-ce:latest
```

```shell
docker run --name cloudbeaver -tid -p 8080:8978 -v /var/cloudbeaver/workspace:/opt/cloudbeaver/workspace dbeaver/cloudbeaver:22.2.0
```



