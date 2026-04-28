# HTB Write-Up - Cap

- **Dificultad:** Easy
- **Sistema operativo:** Linux
- **Autor del writeup:** estifenso
- **Fecha:** 27 de abril de 2026

Cap es una máquina Linux de dificultad fácil con un servidor web administrativo que permite capturas de red. Un fallo IDOR da acceso a capturas de otros usuarios, donde se encuentran credenciales en texto plano para obtener acceso inicial. Luego, una Linux Capability mal configurada permite escalar privilegios hasta root.

## Tabla de contenido

1. [Reconocimiento](#reconocimiento)
2. [Enumeración](#enumeración)
3. [Explotación](#explotación)
4. [Acceso inicial](#acceso-inicial)
5. [Escalada de privilegios](#escalada-de-privilegios)
6. [Flags](#flags)
7. [Resumen](#resumen)

## Reconocimiento

### Conectividad

Primero validé que la máquina respondiera con ICMP. El TTL en 63 confirmó que estaba frente a un sistema Linux.

![Prueba de conectividad](Screenshot%202026-04-27%20at%207.55.51%E2%80%AFPM.png)

### Escaneo TCP inicial

Después hice un escaneo con Nmap para identificar puertos abiertos.

```bash
nmap -p- --open --min-rate 5000 -sS -Pn -v -n 10.10.11.244 -oN ports.txt
```

En este comando, `-p-` revisa todos los puertos TCP, `--open` muestra solo los que responden, `--min-rate 5000` fuerza un ritmo mínimo de 5000 paquetes por segundo, `-sS` realiza un SYN scan, `-Pn` evita la comprobación previa de host activo, `-v` activa salida detallada, `-n` desactiva la resolución DNS y `-oN` guarda el resultado en formato normal.

![Escaneo TCP inicial](Screenshot%202026-04-27%20at%208.01.02%20PM.png)

### Resultados del primer escaneo

Esta captura solo muestra la salida del escaneo, así que la dejé como evidencia del reconocimiento inicial.

![Resultados del primer escaneo](Screenshot%202026-04-27%20at%208.03.20%20PM.png)

### Escaneo más profundo

Con los puertos ya identificados, ejecuté un escaneo más profundo para obtener versiones, scripts de Nmap y cualquier pista útil sobre los servicios expuestos.

```bash
nmap -p22,21,80 -sCV -Pn -v -n 10.10.11.244 -oN services.txt
```

![Escaneo más profundo](Screenshot%202026-04-27%20at%208.02.57%20PM.png)

### Revisión del resultado

Aquí muestro únicamente la salida del cat sobre el escaneo.

![Salida del escaneo](Screenshot%202026-04-27%20at%208.03.20%20PM.png)

## Enumeración

### FTP y Web

Siguiendo con la enumeración, intenté validar si el acceso anónimo por FTP estaba permitido, pero no fue posible. Después continué con `whatweb` para perfilar el servidor web.

![Intento de acceso anónimo y whatweb](Screenshot%202026-04-27%20at%208.09.55%20PM.png)

### Interfaz web principal

Desde Firefox revisé qué alojaba la aplicación y encontré una página orientada a monitoreo de red.

![Vista de la aplicación web](Screenshot%202026-04-27%20at%208.14.24%20PM.png)

### Sección de capturas

Seguí explorando y vi que la aplicación mostraba capturas de tráfico TCP, UDP y otros datos mediante identificadores, por ejemplo `data/3`.

![Listado de capturas](Screenshot%202026-04-27%20at%208.14.42%20PM.png)

### Posible IDOR

Curioseando un poco más, probé con otro ID y noté que el `0` también devolvía una captura descargable. Eso ya apuntaba claramente a un IDOR.

![Prueba con ID 0](Screenshot%202026-04-27%20at%208.15.07%20PM.png)

### Análisis del archivo

La descarga resultó ser un archivo `.pcap`, así que lo analicé con `tshark` para ver su contenido.

![Archivo PCAP](Screenshot%202026-04-27%20at%208.18.09%20PM.png)

### Credenciales en claro

Dentro de la captura apareció algo muy importante: credenciales FTP en texto plano.

![Credenciales FTP en texto plano](Screenshot%202026-04-27%20at%208.18.59%20PM.png)

### Guardado de credenciales

Con esa información, guardé las credenciales para probarlas más adelante.

![Guardando credenciales](Screenshot%202026-04-27%20at%208.19.40%20PM.png)

![Guardando credenciales](Screenshot%202026-04-27%20at%208.20.34%20PM.png)

## Explotación

### Confirmación de las credenciales

Quise confirmar que funcionaran, y efectivamente pude acceder vía FTP con el usuario `nathan`. Desde ahí también encontré la flag de usuario.

![Acceso FTP con nathan](Screenshot%202026-04-27%20at%208.23.59%20PM.png)

![Flag de usuario](Screenshot%202026-04-27%20at%208.24.12%20PM.png)

### Reutilización de contraseña

Como `nathan` no tenía acceso privilegiado, probé si reutilizaba la misma contraseña en SSH. Es un comportamiento bastante común y, en este caso, también funcionó.

![Acceso SSH con las mismas credenciales](Screenshot%202026-04-27%20at%208.26.03%20PM.png)

## Acceso inicial

El acceso inicial quedó resuelto con las credenciales obtenidas desde la captura de red. En la práctica, un simple IDOR fue suficiente para exponer información sensible y abrir la puerta de entrada al sistema.

## Escalada de privilegios

### Búsqueda de vectores

Ya dentro como `nathan`, revisé la raíz del sistema, permisos SUID y posibles binarios útiles para escalar privilegios, pero no encontré nada interesante por esa vía.

![Búsqueda de SUID y permisos](Screenshot%202026-04-27%20at%208.28.59%20PM.png)

### Linux Capability mal configurada

Después busqué capabilities y encontré `cap_setuid` sobre `python3`, algo especialmente peligroso porque permite cambiar el UID del proceso.

![Capabilities del sistema](Screenshot%202026-04-27%20at%208.30.26%20PM.png)

### Escalada a root

Usando Python, importé `os`, ejecuté `setuid(0)` y luego lancé una shell. Con eso la sesión quedó con UID 0, es decir, root.

![Escalada a root](Screenshot%202026-04-27%20at%208.31.47%20PM.png)

### Verificación final

Una vez obtenido root, comprobé el usuario actual y busqué la flag final para cerrar la máquina.

![Root y flag final](Screenshot%202026-04-27%20at%208.32.37%20PM.png)

## Flags

### User flag

La flag de usuario se obtuvo tras el acceso FTP inicial con `nathan`.

### Root flag

La flag de root se obtuvo después de aprovechar la capability mal configurada en `python3`.

## Resumen

Cap fue una máquina sencilla, pero me gustó porque deja claro que un descuido pequeño, como un IDOR, puede convertirse en la puerta de entrada de un atacante al sistema. A partir de una captura de red expuesta se obtuvieron credenciales en texto plano, luego se confirmó reutilización de contraseña y finalmente una capability mal configurada permitió llegar a root.

*Write-up by estifenso | HTB Profile: 1989444 | @estifenso*
