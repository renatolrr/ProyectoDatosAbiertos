# ProyectoDatosAbiertos

Proyecto sobre datos abiertos de la UGR

## Instalación CKAN (versión 2.5, Ubuntu 14.04 64 bits)

Pasos de instalación que luego habrá que automatizar:

### Instalar dependencias

```bash
sudo apt-get update && apt-get install -y apache2 git-core libapache2-mod-wsgi libpq-dev libpq5 nginx openjdk-7-jdk postgresql python-dev python-pastescript python-pip redis-server solr-jetty
```

### Instalar CKAN

```bash
wget http://packaging.ckan.org/python-ckan_2.5-trusty_amd64.deb
sudo dpkg -i python-ckan_2.5-trusty_amd64.deb
sudo pip install -r /usr/lib/ckan/default/src/ckan/requirements.txt
```

### Instalar base de datos PostgreSQL

```bash
sudo -u postgres createuser -S -D -R -P ckan_default
```

Introducir **PASSWORD**.

```bash
sudo -u postgres createdb -O ckan_default ckan_default -E utf-8
```

### Instalar Solr

Editar **/etc/default/jetty**:

- _NO_START=1_ -> **NO_START=0**
- _#JETTY_HOST=$(uname -n)_ -> **JETTY_HOST=127.0.0.1**
- _#JETTY_PORT=8080_ -> **JETTY_PORT=8983**

```bash
sudo service jetty start
sudo mv /etc/solr/conf/schema.xml /etc/solr/conf/schema.xml.bak
sudo ln -s /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /etc/solr/conf/schema.xml
sudo service jetty restart
```

Editar **/etc/ckan/default/production.ini**:

- _sqlalchemy.url = postgresql://ckan_default:pass@localhost/ckan_default_ -> **sqlalchemy.url = postgresql://ckan_default:{PASSWORD}@localhost/ckan_default**
- _#solr_url = <http://127.0.0.1:8983/solr>_ -> **solr_url=<http://127.0.0.1:8983/solr>**
- _ckan.site_url =_ -> **ckan.site_url = {HOST}**

Acceder a **<http://localhost:8983/solr/>**.

### Crear tablas de la base de datos

```bash
cd /usr/lib/ckan/default/src/ckan
paster db init -c /etc/ckan/default/production.ini
```

### Configurar DataStore

Editar **/etc/ckan/default/production.ini**:

- _ckan.plugins = stats text_view image_view recline_view_ -> **ckan.plugins = stats text_view image_view recline_view datastore**

```bash
sudo -u postgres createuser -S -D -R -P -l datastore_default
sudo -u postgres createdb -O ckan_default datastore_default -E utf-8
```

Editar **/etc/ckan/default/production.ini**:

- *#ckan.datastore.write_url = postgresql://ckan_default:pass@localhost/datastore_default -> **ckan.datastore.write_url = postgresql://ckan_default:{PASSWORD}@localhost/datastore_default**
- *#ckan.datastore.read_url = postgresql://datastore_default:pass@localhost/datastore_default -> **ckan.datastore.read_url = postgresql://datastore_default:{PASSWORD}@localhost/datastore_default**

```bash
sudo ckan datastore set-permissions | sudo -u postgres psql --set ON_ERROR_STOP=1
```

Comprobar acceso a DataStore (debería devolver un JSON):

```bash
curl http://localhost/api/3/action/datastore_search?resource_id=_table_metadata
```

Comprobar permiso de escritura:

```bash
curl -X POST http://localhost/api/3/action/datastore_create -H "Authorization: {USER_API}" -d '{"resource": {"package_id": {PACKAGE_ID}}, "fields": [ {"id": "a"}, {"id": "b"} ], "records": [ { "a": 1, "b": "xyz"}, {"a": 2, "b": "zzz"} ]}'
```

Comprobar recurso creado:

```bash
http://localhost/api/3/action/datastore_search?resource_id={5a3304ac-f742-4dfa-aa56-3829f795f326}
```

Eliminar recurso de prueba creado:

```bash
curl -X POST http://localhost/api/3/action/datastore_delete -H "Authorization: 6e30990b-8af0-4920-8fee-74958a6bcfaa" -d '{"resource_id": "5a3304ac-f742-4dfa-aa56-3829f795f326"}'
```

### Configurar FileStore

```bash
sudo mkdir -p /var/lib/ckan/default
```

Editar **/etc/ckan/default/production.ini**:

- *#ckan.storage_path = /var/lib/ckan* -> **ckan.storage_path = /var/lib/ckan/default**

```bash
sudo chown www-data /var/lib/ckan/default
sudo chmod u+rwx /var/lib/ckan/default
sudo service apache2 reload
```

Probar:
```bash
curl -H 'Authorization: {USER_API}' 'http://localhost/api/3/action/resource_create' --form upload=@{FILE} --form package_id={PACKAGE_ID}