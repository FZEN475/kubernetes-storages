# kubernetes-storages
## Description
* Объектное хранилище minio.
  * Объекты хранятся на NFS.
* PostgreSQL
  * Конфигурация primary+read.
  * Пароль postgres статический.
  * Ротация паролей клиентов vault.
  * Резервное копирование каждый час.
  * При каждом запуске pod, база восстанавливается из последней резервной копии init скриптом.
    * Пояснение: нет достаточно быстрого облака. Для достижения максимальной скорости, файлы хранятся в контейнере. (Субоптимально.)
* PGAdmin
  * Предварительно настроенное подключение.
  * Доступ к базе с данными postgres.
* Redis
  * Конфигурация redis+sentinel; три реплики.
  * Сертификаты отключены. (gitlab не подключается к sentinel по ssh при развёртывании helm).
  * Резервное копирование на PVC.

## Dependency
* [Образ](https://github.com/FZEN475/ansible-image)
* [Library](https://github.com/FZEN475/ansible-library)
* <details><summary> .env </summary>

  ```properties
  TERRAFORM_REPO="https://github.com/FZEN475/kubernetes-storages.git"
  #GIT_EXTRA_PARAM="-btemp_branch"
  SECURE_SERVER=""
  SECURE_PATH=""
  LIBRARY="https://github.com/FZEN475/ansible-library.git"
  ``` 
  </details>
* <details><summary> secrets </summary>

  ```yaml
  secrets:
    - id_ed25519
  ```
</details>

## Stages
### [init](https://github.com/FZEN475/kubernetes-storages/blob/main/playbooks/_0_init/_1_install.yaml)
* Создание CronJob для резервного копирования etcd и pgsql.
### [minio](https://github.com/FZEN475/kubernetes-storages/blob/main/playbooks/_1_minio/_1_install.yaml)
* Очистка записей minio в базе etcd.
* Установка minio.
### [pgsql, pgadmin](https://github.com/FZEN475/kubernetes-storages/blob/main/playbooks/_2_pgsql/_1_install.yaml)
* Создание ConfigMap [pg_hba.conf](https://github.com/FZEN475/kubernetes-storages/blob/main/config/_2_pgsql/pg_hba.conf)
* Установка pgsql.
  * При запуске создаются пустые базы и пользователи, после чего запускается восстановление и резервной копии.
* Создание ConfigMap [pgadmin_servers.json](https://github.com/FZEN475/kubernetes-storages/blob/main/config/_2_pgsql/pgadmin_servers.json)
* Установка pgadmin.
### [redis](https://github.com/FZEN475/kubernetes-storages/blob/main/playbooks/_3_redis/_1_install.yaml)
* Установка redis.

## Troubleshoots

<!DOCTYPE html>
<table>
  <thead>
    <tr>
      <th>Источник</th>
      <th>Проблема</th>
      <th>Решение</th>
    </tr>
  </thead>
  <tr>
      <td>minio</td>
      <td>После переустановки:<br/>не применяются новые пароли;<br/>Не создаются buckets.</td>
      <td>

Не очищена база etcd.
</td>
  </tr>
  <tr>
      <td>postgresql</td>
      <td>Не проходит init скрипт.</td>
      <td>

Проверить values.yaml<br/>`replication.synchronousCommit` доложен быть "off".
</td>
  </tr>
  <tr>
      <td>pgadmin</td>
      <td>При настройке автоматического доступа к серверам, запрашивается пароль.</td>
      <td>

При условии, что passfile верный и верно указано его расположение.<br/>
Файл с паролями должен быть с теми же правами владения, что и пользователь запустивший сервер pgsql.<br/>
При обычном монтировании секрета в kubernetes, устанавливается только группа через `securityContext.fsGroup`, но этого не достаточно.<br/>
И права должны быть 0600.
</td>
  </tr>
  <tr>
      <td>redis</td>
      <td>Не ошибка но заметка.</td>
      <td>

Файлы учётных данных redis и sentinel похожи, но есть отличия при ограничении команд.
</td>
  </tr>
</table>



