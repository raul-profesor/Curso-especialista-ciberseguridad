---
title: Ataques en entornos Active Directory I

description: En este apartado vamos a realizar los primeros ataques en entornos Active Directory para la captura de hashes NTLM y su posterior cracking offline mediante hashcat y ataques de diccionario. Pentesting, hacking, AD.
---

## Setup del laboratorio

### Windows Server 

+ Administrador de dominio
+ Usuario raso 1 de dominio 
+ Copiar usuario raso 2 con otro nombre 
+ Copiar usuario administrador con otro nombre
+ Copiar el usuario copiado de administrador como cuenta de servicio SQL

Crear una compartición de archivos: `Server Manager > File and Storage Service > Shares > Task > New Share > SMB Quick`
### Clientes Windows 10 

+ En ambos, crear una carpeta compartida.
+ Unir las máquinas al dominio
+ Hacer al usuario raso 1 administrador local del primer cliente
+ Hacer al usuario raso 1 y 2 administradores local del segundo cliente también



## Responder 

Si os fijáis en [este](https://adam-toscher.medium.com/top-five-ways-i-got-domain-admin-on-your-internal-network-before-lunch-2018-edition-82259ab73aaa) artículo, aunque un poco antiguo, da una idea de las formas más básicas de ataques contra Active Directory para capturar hashes.

En la primera se habla de envenenamiento LLMNR. Este protocolo, bastante antiguo, permite identificar hosts cuando el DNS falla en esta tarea.

Se conoce como envenenamiento LLMNR aquel ataque en el cual el cliente intenta acceder a un servicio de red inexistente y que por lo tanto no tiene una resolución DNS exitosa. En este caso, se lanza un mensaje broadcast LLMNR para resolver la dirección, cosa que el atacante aprovecha para hacerse pasar por ese servicio y solicitar la pertinenete autorización al cliente/víctima.

![](../img/LLMNR-Poisoning.png)

De esta forma, es posible capturar hashes NLTMv2 y descifrarlos offline mediante diccionarios.

Para llevar a cabo un ataque de este tipo quizás la herramienta más conocida sea **Responder**, la más famosa herramienta para envenenamiento LLMNR/LLMNR, NBT-NS Y MDNS (protocolos de función similar). Responder tiene además incorporado servidores HTTP/SMB/MSSQL/FTP/LDAP falsos que sportan NTLMv1/NTLMv2/LMv2, Extended Security NTLMSSP y Basic HTTP authentication.

### Procedimiento

Responder sólo está disponible para Linux y viene preinstalado en Kali, Parrot y otras. Al ejecutarlo, podemos ver que se inician los *envenenadores* y los servidores falsos (podemos indicarlo mediante opciones por línea de comandos que se inicien o no según qué elementos):

![](../img/responder1.png)

Obviamente el atacante ha de estar en la misma red que la víctima. Esta última, de alguna manera (¿os acordáis de la práctica anterior de phising?), ha de ser convencida de intentar acceder a un servicio inexistente.

Por ejemplo, intentando acceder a un recurso compartido con un nombre inexistente, ocurre lo siguiente:

![](../img/responder2.png)

Es decir, **Responder** está simulando ser un servidor SMB que requiere autorización para su acceso. Si la víctima no se da cuenta del engaño e introduce sus credenciales:

![](../img/responder3.png)

**Responder** las captura al instante en forma de Usuario, dominio y hash:

![](../img/responder4.png)

Con esto, ya tendríamos la parte ardúa del trabajo hecha, quedaría descifrar este hash. La herramienta que nos puede ayudar en este menester es **hashcat**, por ejemplo. No obstante, lo comprobaremos.

¿Puede hashcat hacer algo con los hashes NTLMv2? Veamos que nos dice el manual:

![](../img/responder5-hashcat.png)

Todo parece indicar que hay una forma de indicarle a hashcat que se trata de un hash de este tipo y que actúe en consecuencia.

!!!task "Tarea"
    Reproduce el escenario con un password en el cliente lo suficiente débil como para que no nos lleve mucho tiempo el proceso de descifrado con hashcat (`password.123` por ej.)

    Averigua como utilizar hashcat con la opción para hashes NTLMv2 y con el diccionario rockyou.txt (presente en cualquier distribución dedicada a la seguridad).

## Aplicación en el mundo real

Aunque algo antigua, resulta útil revisitar la vulnerabilidad [CVE-2018-13417](https://www.incibe.es/incibe-cert/alerta-temprana/vulnerabilidades/cve-2018-13417). 
 
Se trata de una vulnerabilidad  XXE (XML External Entity Processing) en el cliente de Bittorrent *Vuze*. Como podemos leer [aquí](https://www.cronup.com/que-son-y-como-prevenir-los-ataques-xxe/), "una vulnerabilidad de entidad externa XML se produce cuando el servicio que analiza (o en palabras más sencillas, lee y procesa) los mensajes XML enviados por el cliente, acepta una definición externa del propio mensaje XML."

Vamos a intentar cómo explotar esta vulnerabilidad con los conceptos previos.

### La víctima

Se tratará de un Windows 10 con una instalación nueva de Vuze (<u>**ha de ser la versión 5.7.6.0**</u>):

![](../img/vuze1.png)

### Atacante

Yo he utilizado Parrot pero podréis utilizar Kali o la que más os guste. En este caso vamos a llevar a cabo un remember y haremos uso de la herramienta [Evil-ssdp](https://gitlab.com/initstring/evil-ssdp) para hacer spoofing de respuestas SSDP, creando dispositivos falsos UPnP y detectar vulnerabilidadews XXE en aplicaciones UPnP.

![](../img/vuze2.png)

### Explicación 

Cuando Vuze intenta descubrir otros dispositivos en la red local, SSDP envía un mensaje UDP multicast a la dirección 239.255.255.250 y puerto **1900**. Esto permite a Vuze localizar dispositivos UPnP. Es por esto precisamente que el atacante será capza de responder a estos paquetes con una herramienta como evil-ssdp, diciéndole al cliente que es un dispositovo compartido llamado *Device Descriptor*.

Tras esto, Vuze parsea el contenido del XML de *Device Descriptor* sobre HTTP, pudiendo así obtener archivos, hashes o inclusos shells del atacante. En nuestro caso particular, vamos a ejecutar un ataque XXE que producirá una conexión SMB, permitiéndonos capturar el hash del desafío/respuesta.

Por defecto, evil-ssdp levanta un servidor web y *Device Descriptor* está alojado en:

`http://<IP_Atacante>:8888/ssdp/device-desc.xml`

La ruta de **device-desc.xml** obtiene los datos del archivo *device.xml*, ubicado en la carpeta `/templates/xxe-smb`. Por suerte, evil-ssdp  ya nos ha preconfigurado una línea para el ataque XXE que invocará la conexión SMB por nosotros:

![](../img/vuze3.png)

![](../img/vuze4.png)

![](../img/vuze5.png)

Con todo ya dispuesto, iniciamos **responder**, iniciamos **evil-ssdp** y tras ello arrancamos Vuze en nuestra máquina víctima Windows 10:

![](../img/vuze8.png)

![](../img/vuze6.png)

![](../img/vuze7.png)

Y la magia se produce casi al instante.

### ¿Qué está ocurriendo entre bambalinas?

Si analizamos el escenario como mandan los canones con Wireshark, veremos cosas interesantes.

1. El primer paquete es un multicasting UDP vía SSSDP. También vemos su posterior respuesta informando al cliente de la localización del dispositivo *Device Descriptor*

    ![](../img/vuze9.png)

2. Después se roduce un *3-way handshake* y se lleva a cabo la petición HTTP GET por parte del cliente a *Device Descriptor*

    ![](../img/vuze10.png)

3. Se inicia entonces la conexión SMB por parte del cliente, tras una negociación a propósito de la versión SMB a utilizar. Finalmente el cliente envia su hash NTLMv2 para autenticarse en el servicio y es entonces cuando Responder lo captura:

    ![](../img/vuze11.png)

    El valor del hash mostrado en la imagen de la captura de Wireshark debe coincidir con el captura por Responder.

## SMB Relay

!!!info 
    Microsoft está empezando a hacer más enfásis en algunos aspectos históricos de su seguridad. Uno de ellos era que por defecto los mensajes SMB no estaban firmados,[ cosa que parece que empieza a cambiar](https://techcommunity.microsoft.com/t5/storage-at-microsoft/smb-signing-required-by-default-in-windows-insider/ba-p/3831704) y a requerirse que sí lo haga por defecto.

    [<u>¿Qué significa que los mensajes SMB estén firmados?</u>](https://learn.microsoft.com/es-es/troubleshoot/windows-server/networking/overview-server-message-block-**signing**)

    *La firma SMB (también conocida como firmas de seguridad) es un mecanismo de seguridad en el protocolo SMB. La firma SMB significa que cada mensaje SMB contiene una firma que se genera mediante la clave de sesión. El cliente coloca un hash de todo el mensaje en el campo de firma del encabezado SMB.*

    Para realizar un ataque de tipo **SMB Relay** es imprescindible que estos mensajes no vayan firmados, así que tenemos esta opción activada por defecto en los clientes (no en el DC), debemos desactivarla. ¿Cómo? [Así](https://techcommunity.microsoft.com/t5/storage-at-microsoft/smb-signing-and-guest-authentication/ba-p/3846679).

    En un principio esto no nos aleja de un entorno real puesto que muchos equipos tendrán aún versiones antiguas de Windows o incluso habrán desactivado las firmas.

En esta ocasión sí vamos a desactivar nuestros Windows Defender en ambos clientes Windows puesto que lo que intentamos es presentar los fundamentos de este ataque, no este ataque junto con la evasión de antivirus.

### ¿Qué es SMB Relay?
En lugar de intentar descifrar o crackear los hashes que hemos recopilado con el *responder*, podemos hacer relay de esos hashes a máquinas potencialmente vulnerables con el fin de obtener acceso a ellas.

Para conseguir nuestro fin necesitamos dos requisitos fundamentales:

+ SMB Signing tiene que estar deshabilitado en el objetivo o víctima
+ Las credenciales de las que se hace relay, que son reenviadas, deben tener privilegios de admin en la máquina (para el caso que aquí se muestra)


Los pasos a seguir son:

1. Editamos la configuración del responder para desactivar los servidores fake SMB y HTTP. Esto quiere decir que estaremos escuchando, pero no respondiendo. Por lo tanto, capturaremos la petición de acceso con responder y haremos relay de ella con otra herramienta.

    ![](../img/smbrelay1.png)

    ![](../img/smbrelay2.png)

2. Hacemos relay de la petición de autorización con `ntlmrelayx`, que utilizará un archivo de texto donde vendrán definida/s la víctima/s.

    ![](../img/smbrelay3.png)

3. Y si hemos accedido con un usuario con privilegios de administrador, veremos que se produce un dumping de la SAM:

    ![](../img/smbrelay4.png)
    ![](../img/smbrelay5.png)



### ¿Cómo puedo saber las máquinas de la red que tiene SMB sin firmar?

Una vez más podemos ayudarnos de nmap:

```sh
nmap --script=smb2-security-mode -p445 192.168.18.0/24
```

!!!task "Tarea"
    Busca la forma de conectarte remotamente a la máquina con las credenciales comprometidas, usando esas credenciales y la herramietna `psexec`.


### Mitigación del ataque SMB Relay

+ Activar el firmado de mensajes SMB (SMB signing) en todos los dispositivos
    + <u>Contrapartida</u>: puede provocar lentintud a la hora de copiar archivos
+ Deshabilitar autenticación NTLM
    + <u>Contrapartida</u>: Si Kerberos, por lo que sea, deja de estar disponible en la red, Windows volverá a NTLM
+ Jerarquización de cuentas (limitar los administradores de dominio a tareas específicas)
+ Restricción de administradores locales
    + <u>Contrapartida</u>: Con toda probabilidad aumentará las llamadas a soporte informático en la empresa


## Referencias

[Portswigger - XML external entity (XXE) injection](https://portswigger.net/web-security/xxehttps://portswigger.net/web-security/xx`)