### 操作docker部署的opengauss数据库和postgres数据库

---

```shell
# 操作opengauss数据库
docker exec -it containter_name bash
su - omm
gsql -U user_name -d database_name

# 操作postgres数据库
docker exec -it containter_name psql -U user_name -d database_name
```