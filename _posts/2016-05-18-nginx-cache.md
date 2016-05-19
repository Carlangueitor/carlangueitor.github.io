---
layout: post
title: "Caché con NGINX"
date: 2016-05-18 19:54:21
category: Sysadmin
tags: nginx, cache, linux, performance
---
Recientemente en el trabajo agregamos caché a una API REST con bastante tráfico, al final decidimos usar la caché de NGINX en lugar de usar un servicio extra (p.e. [Varnish][2]) por todo lo que conlleva mantener algo extra, y como eran pocos endpoints podíamos hacer la prueba.


Para activar la caché primero hay que agregar dos directivas (ambas en el contexto `http`): `proxy_cache_path` que indica en donde será almacenada y otros parametros como el tamaño del indice y máximo tamaño de la caché así como la expiración de ésta; y `proxy_cache_key` para generar una key para la petición.

```
http {
	...
	proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m inactive=24h max_size=80m;
	proxy_cache_key "$host$request_uri";
	...
}
```

La parte que nos interesa para los siguientes pasos es `keys_zone` ya que así haremos referencia a esa configuración en donde queramos aplicarlo, en este caso un `location`, también añadimos un header extra para verificar el estado de la caché:

```
location {
	proxy_cache	mycache;
	add_header	X-Proxy-Cache $upstream_cache_status;
	...
}
```

## Caché por URL

Si por algún motivo no se puede poner la condición dentro del `location` (p.e. cuando tienes demasiadas URL que *cachear*), se puede usar un `map` para definir las URLs que se *cachearán*:

```
map $uri $skip_cache {
	default			1;
	~^/cacheable/path	0;
	~^/another/with/cache	0;
}
```

Para evitar que algo entre a la caché se usan dos directivas: `proxy_cache_bypass` y `proxy_no_cache`, por esa razón en el `map` anterior marcamos como default que todas las URL por defecto saltarán la caché a menos que las añadamos ahí mismo, entonces tendríamos que modificar el `location` un poco:

```
location {
	proxy_cache		mycache;
	proxy_cache_bypass	$skip_cache;
	proxy_no_cache		$skip_cache;
	add_header		X-Proxy-Cache $upstream_cache_status;
	...
}
```	

## Verificando

Para verificar si se está usando o no la caché correctamente podemos revisar si hay archivos en `/var/cache/nginx` pero también podemos hacerlo por petición con ayuda del header que añadimos anteriormente y `curl`:

```
$ curl -XGET -I http://host/cacheable/path
HTTP/1.1 200 OK
Server: nginx
...
X-Proxy-Cache: HIT
```

```
$ curl -XGET -I http://host/no/cacheable/path
HTTP/1.1 200 OK
Server: nginx
...
X-Proxy-Cache: BYPASS
```

Si es la primera vez que se llama al path seleccionado entonces se creará la caché entonces lo más probable es que nos devuelva un `MISS` en el estatus de la caché (para ver todos los estatus posibles revisa la [documentación del módulo `ngx_http_upstream_module`][1]).

## UWSGI
En mi caso estaba aplicando esto a un servicio en Python así que usabamos UWSGI para esto, sí ese es el caso, NGINX provee de directivas equivalentes para el manejo de caché:

* `uwsgi_cache_path`
* `uwsgi_cache_key`
* `uwsgi_cache`
* `uwsgi_cache_bypass`
* `uwsgi_no_cache`

Hay que tomar en cuenta que una misma directiva `*_cache_path` solo puede usarse con el mismo tipo de proxy, por ejemplo no puedes cachear un `proxy_pass` y un `uwsgi_pass` al mismo tiempo.

## Más performance
Si queremos afinar un poco esto, podemos almacenar la caché en una unidad virtual que use la RAM como almacenamiento, aunque esto puede dar un performance con peticiones pequeñas y variadas (p.e. muchos distintos parametros `GET`), puede que no sea lo más adecuado para tu aplicación e incluso no afecte en absoluto ( el sistema operativo hace también su parte), se puede hacer una prueba de esta manera:

```bash
mount -t tmpfs -o size=100M tmpfs /var/cache/nginx
```

Y en caso de que sequiera aplicar permamentemente habrá que editar el archivo `/etc/fstab`y agregar la siguiente linea:

```
tmpfs /var/cache/nginx tmpfs defaults,size=100M 0 0
```

## Referencias

* [A Guide to Caching with NGINX][3]

[1]: http://nginx.org/en/docs/http/ngx_http_upstream_module.html#variables
[2]: https://varnish-cache.org/
[3]: https://www.nginx.com/blog/nginx-caching-guide/
