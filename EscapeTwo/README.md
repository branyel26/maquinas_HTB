# HTB Write-Up - EscapeTwo

**Dificultad:** Easy  
**Sistema operativo:** Windows  
**Autor del writeup:** estifenso  
**Fecha:** 10 de junio de 2026

EscapeTwo es una máquina Windows de dificultad fácil enfocada en el compromiso completo de un dominio. Yo todavía soy nuevo atacando Active Directory, pero justo por eso este tipo de labs me emocionan tanto: se aprende un montón en cada paso y te obliga a entender bien lo que estás viendo.

La ruta fue directa, pero muy buena para practicar. Empecé con conectividad, luego saqué las credenciales iniciales, después encontré un archivo Excel dañado que me dio más información, y con eso hice password spraying para llegar a MSSQL. A partir de ahí seguí enumerando el sistema hasta encontrar más piezas útiles.

## Reconocimiento

Primero validé que la máquina respondiera con ping.

![Ping a la máquina](1.png)

Después hice un escaneo TCP con Nmap para empezar a ver qué servicios estaban abiertos.

![Escaneo TCP inicial](2.png)

En esta captura se ve mejor la salida completa del escaneo.

![Salida ampliada del escaneo](3.png)

Ya con los puertos abiertos, quedó claro que estaba frente a un entorno de Active Directory.

![Pistas de Active Directory](4.png)

La captura inicial de la máquina la dejé arriba porque ahí se ve la IP y el contexto general.

![Máquina EscapeTwo](5.png)

Como ya tenía el dominio y el host, lo añadí a `/etc/hosts` para que resolviera bien por DNS local.

![Ajuste de /etc/hosts](4.png)

## Credenciales iniciales y shares

HTB entrega unas credenciales válidas al inicio, así que las usé para empezar a enumerar con acceso autenticado.

![Credenciales iniciales](6.png)

Con NetExec confirmé que esas credenciales sí funcionaban.

![Validación con NetExec](7.png)

Luego enumeré los shares para ver si había algo útil.

![Enumeración de shares](8.png)

También volví a usar NetExec para enumerar usuarios con más calma.

![Enumeración de usuarios](33.png)

Con la misma sesión seguí revisando shares y encontré algo interesante para bajar.

![Más shares](34.png)

Ahí apareció un archivo que valía la pena revisar con más detalle.

![Archivo interesante](32.png)

Lo guardé en mi carpeta de trabajo para irlo organizando.

![Archivo organizado](9.png)

## Archivo Excel dañado

El archivo resultó ser de Excel, así que revisé su estructura interna porque estos documentos suelen ir en XML.

![Archivo Excel](10.png)

Al abrirlo vi que estaba dañado, pero todavía se podía leer parte del contenido.

![Contenido XML](11.png)

Después limpié todo el ruido para quedarme con lo importante.

![Limpieza del XML](12.png)

Ahí aparecieron credenciales nuevas, que fue el hallazgo más útil de esta parte.

![Credenciales recuperadas](13.png)

## Password spraying y MSSQL

Con esas credenciales empecé a cruzar usuarios y contraseñas.

![Cruce de credenciales](14.png)

Oscar fue uno de los usuarios que validó correctamente.

![Usuario Oscar](14.png)

También apareció `sa`, que me interesó bastante porque apuntaba a MSSQL.

![Cuenta sa](15.png)

Con `sa` me conecté a MSSQL usando Impacket.

![Conexión a MSSQL](16.png)

Ya dentro, comprobé que `xp_cmdshell` estaba disponible.

![xp_cmdshell](16.png)

A partir de ahí seguí enumerando el sistema desde el contexto de MSSQL.

![Enumeración desde MSSQL](17.png)

El siguiente paso fue subir `netcat` con `certutil` para enviarme una reverse shell.

![Subida de netcat](36.png)

Con eso logré conseguir acceso por la reverse shell.

![Reverse shell](42.png)

Después seguí enumerando y encontré un archivo de configuración, que casi siempre es buena idea revisar porque suele guardar credenciales en texto plano.

![Archivo de configuración](35.png)

Ahí encontré las credenciales de un usuario llamado `sql_svc`.

![Credenciales de sql_svc](18.png)

Con NetExec comprobé que esas credenciales eran válidas.

![Validación de sql_svc](18.png)

Probé la misma contraseña con otro usuario y también funcionó para `ryan`.

![Mismo password para ryan](19.png)

Después validé WinRM con Evil-WinRM y me apareció `Pwn3d!`.

![WinRM con Evil-WinRM](40.png)

## Enumeración del dominio

Ya dentro de la máquina, subí SharpHound para mapear el AD.

![Subida de SharpHound](39.png)

Ejecuté `SharpHound.exe`.

![Ejecución de SharpHound](38.png)

Con SharpHound me descargué el `.zip` generado.

![ZIP de SharpHound](21.png)

Luego abrí BloodHound para revisar la data que había sacado del AD.

![BloodHound abierto](22.png)

Era la primera vez que usaba la herramienta, así que fui revisando con calma lo que mostraba.

![Uso inicial de BloodHound](23.png)

![Uso inicial de BloodHound](24.png)

![Uso inicial de BloodHound](26.png)

Para BloodHound tuve que cambiar la password por defecto y editar el archivo JSON en `/etc/bhapi/bhapi.json`.

![Cambio de password de BloodHound](45.png)

![Edición del JSON](47.png)

Después cargué el `.zip` que había sacado con SharpHound.

![Carga del ZIP en BloodHound](27.png)

Quise ver qué más podía hacer para comprometer el sistema siendo `ryan`.

![Más acciones con ryan](28.png)

Ahí encontré algo relacionado con certificados.

![Hallazgo de certificados](30.png)

Resultó que el usuario pertenecía a un grupo que me dejaba publicar certificados.

![Grupo relacionado con certificados](52.png)

También vi que podía llegar al usuario `CA_SVC` abusando de `WriteOwner`.

![WriteOwner sobre CA_SVC](51.png)

`CA_SVC` es una cuenta de servicio de la Autoridad de Certificados, y `WriteOwner` me permitía cambiar el propietario de un objeto en AD.

Con eso ya quedaba claro que, si encontraba una plantilla vulnerable, podía seguir avanzando hacia el DC.

## Movimiento a ADCS

Pedí ayuda a mi IA para abusar de `WriteOwner` de un usuario a otro y terminé adueñándome de `ca_svc`.

![Abuso de WriteOwner](49.png)

Después importé PowerView, que sirve para enumerar y analizar el entorno de AD.

![PowerView](50.png)

Usé NetExec por SMB para comprobar que todo había quedado bien.

![Prueba con NetExec SMB](43.png)

Con Certipy encontré una plantilla de certificados vulnerable llamada `DunderMifflinAuthentication`.

Al analizarla, vi que era un escenario ESC4, donde un usuario tiene permisos suficientes para modificar la configuración de la plantilla.

Eso permite ajustar sus parámetros de seguridad y convertirla en una vía para pedir certificados con otra identidad dentro del dominio.

Si al intentar actualizar la plantilla sale error, puede tocar editar manualmente el archivo generado por Certipy.

En mi caso, modifiqué la línea 23 de `msPKI-Certificate-Name-Flag`, dejé el valor en `1` y así pude habilitar que el solicitante definiera la identidad del certificado al pedirlo.

![Ajuste manual de la plantilla](54.png)

Aquí se puede leer la referencia completa sobre la escalada con Certipy: [Certipy Wiki - Privilege Escalation](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation).

## Compromiso del DC

Con todo eso pude obtener el hash NTLM del usuario `Administrator`.

![Hash del Administrator](47.png)

Luego me conecté como `Administrator` vía Evil-WinRM usando ese hash.

![Acceso como Administrator](48.png)

Finalmente localicé la flag `root.txt`, cerrando la máquina con el control total del DC.

![root.txt](53.png)

La IP de la víctima cambió a mitad del write-up porque apagué la máquina después de sacar la primera flag y seguí el laboratorio más tarde.

## Cierre

EscapeTwo me gustó porque mezcla enumeración, credenciales, MSSQL, BloodHound y ADCS en una sola ruta.

También me sirvió bastante porque sigo aprendiendo a atacar Active Directory, y este tipo de labs me ayudan un montón a ir entendiendo mejor cada paso.

*Write-up by estifenso | HTB Profile: estifenso | 10 de junio de 2026*
