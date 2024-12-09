# Docker Compose - Elasticsearch and Kibana
***
> docker-compose.yml

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.authc.api_key.enabled=true
      - xpack.security.authc.realms.native.native1.order=0
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - esdata:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
    container_name: kibana
    environment:
      - SERVER_NAME=kibana
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=${ELASTICSEARCH_USERNAME}
      - ELASTICSEARCH_PASSWORD=${ELASTICSEARCH_PASSWORD}
      # - ELASTICSEARCH_SERVICEACCOUNTTOKEN=${ENV_WITH_THE_TOKEN_VALUE}
    ports:
      - "5601:5601"
    depends_on:
      elasticsearch:
        condition: service_started

volumes:
  esdata:
    driver: local
```
---
> .env

```.env
STACK_VERSION=version_actual
ELASTIC_PASSWORD=tu_password
ELASTICSEARCH_USERNAME=elastic #default
ELASTICSEARCH_PASSWORD=tu_password
ENV_WITH_THE_TOKEN_VALUE=clave_generada
```
---
## ¿Qué hace esta configuración?

### 1. Elasticsearch:
* Usa la imagen oficial `elasticsearch:8.16.1`.
* Configura la seguridad con `xpack.security.enabled=true`.
* Habilita el uso de contraseñas y claves de API `(xpack.security.authc.api_key.enabled=true)`.
* Define una contraseña para el usuario `elastic` mediante la variable de entorno `ELASTIC_PASSWORD`.

### 2. Kibana:
* Usa la imagen oficial `kibana:8.16.1`.
* Conecta Kibana a Elasticsearch mediante el usuario `elastic` y su contraseña.
* Define `ELASTICSEARCH_HOSTS` apuntando al servicio de Elasticsearch.

### 3. Volúmenes:
* Se crea un volumen llamado `esdata` para persistir los datos de Elasticsearch.
---
## Acceso a los Servicios

* #### Elasticsearch:
    *  URL: `http://localhost:9200`
    *  Usuario: `elastic`
    *  Contraseña: `tu_password`

* #### Kibana:
    * URL: `http://localhost:5601`
    * Usa el mismo usuario y contraseña configurados en Elasticsearch (`elastic` / `tu_password`).
---
## Ejecutar el `docker-compose.yml`
Para ejecutar el `docker-compose.yml` para poder ya ejecutar los servicios de `elasticsearch` y `kibana`, usamos el siguiente comando.
```bash
docker-compose up -d
```
---
## Errores Kibana inicialización

```Log
[FATAL][root] Reason: [config validation of [elasticsearch].username]: value of "elastic" is forbidden. This is a superuser account that cannot write to system indices
```

### Paso 1: Generar un Service Account Token

Ejecuta el siguiente comando dentro del contenedor de Elasticsearch para generar el token:

```bash
docker exec -it elasticsearch bin/elasticsearch-service-tokens create elastic/kibana kibana-service
```

Esto generará un token como este (lo que vez es un ejemplo de token):

```plaintext
SERVICE_TOKEN elastic/kibana/kibana-service = eyJ2ZXIiOiIxIiwidHlwIjoiSldUIiwiYWxnIjoiSFMyNTYifQ...
```

Copia y guarda este token, ya que lo necesitarás para configurar Kibana.

### Paso 2: Configurar Kibana con el Token

En el archivo `.env`, el key generado debe ser copiado en el apartado `ENV_WITH_THE_TOKEN_VALUE=eyJ2ZXIiOiIxIiwidHlwIjoiSldUIiwiYWxnIjoiSFMyNTYifQ...`, debes comentariar los datos `- ELASTICSEARCH_USERNAME=${ELASTICSEARCH_USERNAME}` y `- ELASTICSEARCH_PASSWORD=${ELASTICSEARCH_PASSWORD}`, y descomentarear el dato `- ELASTICSEARCH_SERVICEACCOUNTTOKEN=${ENV_WITH_THE_TOKEN_VALUE}`. Entonces el servicio de kibana quedaria de la siguiente forma en el  `docker-compose.yml`:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.authc.api_key.enabled=true
      - xpack.security.authc.realms.native.native1.order=0
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - esdata:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
    container_name: kibana
    environment:
      - SERVER_NAME=kibana
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      # - ELASTICSEARCH_USERNAME=${ELASTICSEARCH_USERNAME}
      # - ELASTICSEARCH_PASSWORD=${ELASTICSEARCH_PASSWORD}
      - ELASTICSEARCH_SERVICEACCOUNTTOKEN=${ENV_WITH_THE_TOKEN_VALUE}
    ports:
      - "5601:5601"
    depends_on:
      elasticsearch:
        condition: service_started

volumes:
  esdata:
    driver: local
```

### Paso 3: Comandos para reinstalar Kibana

1. Primero paramos el contenedor por seguridad antes de eliminarlo.

```bash
docker-compose stop kibana
```

2. Despues eliminamos el contenedor que ya paramos. Le damos `-f` para que no pregunte si deseo eliminar el contenedor.

```bash
docker-compose rm -f kibana
```

3. Volvemos reinstalar el contenedor especifico que es kibana.

```bash
docker-compose up -d kibana
```

Si no se hace de esta forma, despues se pierde el key generado previamente para que `kibana` pueda conectarse a `elasticsearch`.

---

## Verifica el Acceso

* __Elasticsearch:__ Continúa accediendo a `http://localhost:9200` con el usuario elastic y su contraseña para tareas administrativas.
* __Kibana:__ Ve a `http://localhost:5601`. Kibana debería conectarse exitosamente a Elasticsearch usando el token.
