---
title: Leaker Tracker Simulator - Java & SQL
date: 2026-01-22 12:00:00 +0100
categories: [proyectos]
tags: [programming, java]
pin: true
---

## 🛡️ Descripción del Proyecto

Este proyecto nació durante mi primer año en **DAM** como una solución para concienciar sobre la ciberseguridad. Se trata de un simulador de búsqueda de credenciales filtradas, inspirado en herramientas reales de monitorización de brechas de datos.

La aplicación permite gestionar y consultar grandes volúmenes de información comprometida, diferenciando claramente entre un acceso público para usuarios y un panel de control avanzado para administradores.

> 💡 **Nota personal:** Este fue uno de los primeros proyectos "serios" que desarrollé. Aunque soy consciente de que el código no es impecable y hoy lo haría de otra forma, le guardo mucho cariño por ser el reto con el que realmente empecé a entender la integración entre lógica y bases de datos.

---

### 📸 Vista previa

![Desktop View](/assets/img/tracker.png)
_Interfaz principal del simulador de filtraciones_

---

### 👤 Funcionalidades del Usuario

El sistema está diseñado para ser intuitivo y rápido, permitiendo al usuario final:

- **Verificación de Brechas**: Comprobar si sus credenciales personales han sido expuestas.
- **Estadísticas Globales**: Consultar el volumen total de registros comprometidos.
- **Acceso Seguro**: Portal de login dedicado para personal administrativo.

### 🔐 Panel de Administración (Gestión de Datos)

El rol de administrador cuenta con herramientas potentes para el mantenimiento:

- **Operaciones CRUD**: Añadir, actualizar y borrar filtraciones.
- **Consola de Consultas**: Ejecución de consultas SQL directas sobre la tabla de usuarios.
- **Mantenimiento**: Funciones para limpiar datos, modificar columnas y optimizar tablas.
- **Auditoría**: Historial para consultar todas las modificaciones realizadas.

---

## 🛠️ Stack Tecnológico

- **Lenguaje**: Java
- **Base de Datos**: MySQL / MariaDB

Puedes encontrar un video demostrativo y el código fuente completo en mi repositorio de GitHub.

> [🔗 Enlace al Proyecto en GitHub](https://github.com/n3oari/FiltracionesTracker-AdminAPP-TrabajoClase)
