## Primeros pasos con Elasticsearch
Este proyecto muestras los pasos a seguir en el primer acercamiento con el motor de busquedas Elasticsearch y su integracion con las visualizaciones y tableros de Kibana

## Detalles del proceso

### Crear indice

En la consola DEVTOOLS he creado el indice con la siguiente instruccion
```  
PUT /log_consultas
```
Lo que ha generado el indice correspondiente
![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/CREAR%20INDICE.png?raw=true)

Una vez que generé el indice, procedí a cargar en él, un documento con la siguiente instruccion
```  

### Template
PUT log_consultas/_doc/1
{
"@timestamp":"2010-05-15T22:00:54",
"estado_consulta":"consumo",
"servicio":"consulta",
"administrador":"Juan Carlos",
"consultas_realizadas":52
}
```
![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/BUSCANDO%20EL%20MAPPING.png?raw=true)

Lo que ha generado el primer registro en el indice y con ello se ha generado un template automatico, el cual copié para editarlo. 
Para los campos he decidido que aquellos de tipo "text" sean cambiados a tipo "keyword" para optimizar los resultados de las busquedas al tener coincidencias precisas contra el formato text.
Tambien agregué configuraciones para el almacenamiento del indice. Considerando un cluster distribuido y con ingestas continuas
El resultado final se encuentra en el documento [log_consultas.json](https://github.com/mvasquezn/elasticTest/blob/main/log_consultas.json)

Una vez que complete el template. El paso siguiente fue eliminar el indice anterior para que todos los documentos tengan el mismo mapping. Para ello en la consola ejecuté el siguiente comando.
```  
DELETE log_consultas
```
![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/ELIMINANDO%20EL%20INDICE.png?raw=true)

Entonces fue momento de cargar el nuevo template con el siguiente comando.

```  
PUT _template/log_consultas
{
  "index_patterns": [
    "log_consultas*"
  ],
  "settings": {
    "index.refresh_interval": "1s",
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.mapping.total_fields.limit": 10000
  },
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "administrador": {
        "type": "keyword"
      },
      "consultas_realizadas": {
        "type": "long"
      },
      "estado_consulta": {
        "type": "keyword"
      },
      "servicio": {
        "type": "keyword"
      }
    }
  }
}
```
![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/CARGANDO%20EL%20TEMPLATE.png?raw=true)

Se generó el template correspondiente con el nombre log_consultas

### Carga de los datos.

Para empezar a poblar el indice con nuevos documentos, he tomado el ejemplo deñ archivo enviado para la prueba y han sido indexados con el siguiente comando:
```
POST _bulk

Aquí va todo el documento .JSON
```

### Creando el Index pattern.

Para poder visualizar la carga de los documentos en Discover, he creado un index pattern que toma como marca de tiempo el campo @timestamp.

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/CREANDO%20INDEX%20PATTERN.png?raw=true)

Con esto ya puedo ver datos por medio del Discover colocando como filtro el rango de fechas correspondiente a los datos cargados.

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/DOCUMENTOS%20CARGADOS.png?raw=true)

## Consultas

Para realizar las consultas documentadas en la prueba he decidido cambiar el uso de SEARCH API por el COUNT API ya que contiene el numero de registros filtrados de acuerdo a lo solicitado en la prueba.
Los resultados son los siguientes. 

- Obtener el número de registros con estado_consulta igual a error y consumo.
Para lo cual he ocupado una consulta tipo Boolean para extrer datos de dos diferentes tipos de estado de consulta

```
GET log_consultas/_count
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "estado_consulta": "error"
          }
        },
        {
          "term": {
            "estado_consulta": "consumo"
          }
        }
      ]
    }
  }
}
```

Los resultados son los siguientes:

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/PRIMER%20CONSULTA.png?raw=true)

-Obtener el número de registros realizados por el administrador Juan Lara. 

Repitiendo el uso de COUNT API y una consulta de tipo TERM
```
GET log_consultas/_count
{
  "query" : {
    "term" : { "administrador": "Juan Lara" }
  }
}

```
Los resultados son los siguientes:

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/SEGUNDA%20CONSULTA.png?raw=true)

-Obtener el número de registros con estado_consulta igual a informativo y servicio igual a borrado

Aquí he decidido utilizar un QUERY STRING para agrupar las condiciones del planteamiento

```
GET log_consultas/_count
{
  "query": {
    "query_string": {
      "query": "(estado_consulta:informativo) AND (servicio:borrado)"
    }
  }
}
```

Obteniendo el siguiente resultado.

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/TERCER%20CONSULTA.png?raw=true)

-Obtener la suma de los valores en consultas_realizadas con estado_consulta igual a error

Para esta consulta, utilicé API SEARCH con una agregación de tipo suma. Mediante el siguiente comando
```
POST log_consultas/_search?size=0
{
  "query": {
    "constant_score": {
      "filter": {
        "match": { "estado_consulta": "error" }
      }
    }
  },
  "aggs": {
    "consultas_realizadas_value": { "sum": { "field": "consultas_realizadas" } }
  }
}
```

Con el siguiente resultado.

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/CUARTA%20CONSULTA.png?raw=true)


## Visualizaciones. 
El detalle de las visualizaciones y el tablero se encuentran disponibles en el archivo [OBJECTS_TEST](https://github.com/mvasquezn/elasticTest/blob/main/OBJECTS_TES.ndjson)

Como ya había creado el index pattern en pasos anteriores, ahora podia crear distintos tipos de visualizaciones desde la librería.

- Vista de heat map, donde mostraras el número de servicios realizados por administrador.

Para lo cual he contado el numero de documentos, en el eje "Y" he divido los resultados por las diferencias encontradas en el campo administrador y por ultimo para el eje "X" el mismo proceso en el campo estado_consulta
El resultado es el siguiente


![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/HEATMAP.png?raw=true)

- Vista de Barras, donde se grafique el número de registros con estado_consulta igual a error a través del tiempo.

Aquí he aprovechado una de las ultimas caracteristicas de ELK y he creado la visualizacion con ayuda del Drop & Down de Lens y los campos estado_consulta y @timestamp.
Con el siguiente resultado

![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/CHART%20CONSULTAS%20BY%20TIMESTAMP.png?raw=true)


## Tablero
 Con las visualizaciones creadas y una metrica adicional que cuenta el numero de mensajes agrupados por estado_consulta he creado el siguiente tablero. 
 
 ![alt text](https://github.com/mvasquezn/elasticTest/blob/main/Imagenes/DASHBOARD_TEST.png?raw=true)
 
 
