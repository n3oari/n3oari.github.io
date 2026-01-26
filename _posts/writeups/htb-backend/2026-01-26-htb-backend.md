---
title: Write Up - Hack The Box - Backend
date: 2026-01-26 06:00:00 +0100
categories: [writeups]
tags: [pentesting, hackthebox,cybersecurity,writeups]
pin: true
---

#### Resumen de la máquina

>Esta máquina presenta una infraestructura basada en una **API (FastAPI)** que expone diversos endpoints. La explotación comienza con la enumeración de rutas ocultas mediante fuzzing, lo que revela vulnerabilidades de **IDOR**. Tras escalar privilegios a nivel de API mediante el secuestro de la cuenta de administrador, se descubre un **LFI** y un **RCE** condicionado por una variable de depuración en el token JWT.

#### Heramientas utilizadas:

>nmap
>wfuzz
>burpsuite
>jwt_tool

### Reconocimiento:
^reconocimiento

El escaneo inicial revela un servicio web en el puerto 22,80 asociado al dominio `backend.htb`.

### Enumeración y explotación de vulnerabilidades:
^enumeracion


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

Encontramos `
```
/api/v1/user/login 
/api/v1/user/singup
```

Enumeración de usuarios via IDOR


![Screenshot1](/assets/writeups/backend/1.png)

Campos requeridos para loggearse

![Screenshot2](/assets/writeups/backend/2.png)

Registramos usuario en el endpoint que encontramso anteriormente
`curl -X POST "http://backend.htb/api/v1/user/signup" -H "Content-Type: application/json" -d '{"email":"foo2@bar.com","password":"foo"}'`

Nos logeamos
` curl "http://backend.htb/api/v1/user/login" -d "username=foo@bar.com&password=foo"`

![Screenshot3](/assets/writeups/backend/3.png)

Seteamos el acces token a la funcionalidad de burpsuite match and replace para que cada solicitud que realicemos use el token

![Screenshot4](/assets/writeups/backend/4.png)


Ahora tenemos acceso a la api **Fast Api** donde obtenemos acceso a diferentes funcionalidades.

La primera que hacemos es obtener la user flag


![Screenshot5](/assets/writeups/backend/5.png)