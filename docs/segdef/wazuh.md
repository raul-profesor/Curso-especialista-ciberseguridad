## ¿Qué es Wazuh?

Wazuh, en palabras de su creador, el español Santiago Basset, es una proyecto open source que trata de prevenir, detectar y responder a amenazas.

Técnicamente podría considerarse un HIDS (Host Intrusion Detection System). Estos dispositivos centraban la importancia en los eventos en la red. Sin embargo ésta es una tendencia cambiante, como vemos en este gráfico de ejemplo de MITRE:

![](../img/threats.webp)

Así pues, Wazuh quizás se acerque más a un [XDR](https://wazuh.com/platform/xdr/) o un [SIEM](https://wazuh.com/platform/siem/).

## Web shells

Los cibercriminales utilizan diferentes técnicas para conseguir la persistencia en un sistema previamente comprometido. Una de estas técnicas son las web shells.

### ¿Qué son las webshells?

Son scripts web que permiten a los atacantes acceso remoto sin restricciones. Puede pasar que un atacante logra comprometer un servidor, consiguiendo el acceso inicial al servicio web mediante alguna técnica ya conocida (SQLi, XSS, RFI...).

Con este compromiso inicial, se puede intentar realizar la inyección de una web shell en el directorio del servidor y que constituirá un *backdoor* que dará paso a actividades post-explotación (ejecución de comandos, exfiltración de información, infección de malware...).

![](../img/wazuh1.png)

La mayoría de los web shells siguen los mismos principios de diseño y e intenciones. Normalmente suelen estar escritos en lenguajes soportados por los servidores web: PHP, ASP, ASP.NET, Perl, Python...

### Indicadores comunes de los web shells

+ **<u>Archivos subidos o modificados recientemente:</u>** los atacantes suelen subir sus web shells en los directorios utilizados por los servidores web o modificar archivos ya presentes en ellos. Disonancias en las fechas de modificación pueden ser un indicativo.</u>
+ **<u>Conexiones de red inusuales:</u>** las web shell puede que tengan que poner determinados puertos a la escucha para establecer *reverse shells* o shells inversas hacia los atacantes. Esto producirá tráfico inusual (TCP o UDP), que puede ser un indicador de compromiso.
+ **<u>Configuraciones extrañas/erróneas y cabeceras modificadas:</u>** cabeceras clásiscas de las peticiones HTTP son *user-agent* o *referer*. Los atacantes pueden realizar la modificación de estas cabeceras de tal forma que se intente una ejecución de comandos mediante ellas.
+ **<u>Técnicas de ofuscación:</u>** puede que el atacante emplee técnicas de codificación, compresión o sustitución para intentar ser detectado por los sitemas de seguridad.

## Manos a la obra, configuración del laboratorio

Para este laboratorio utilizaremos AWS, por la versatilidad que nos ofrece y así deshacernos de los problemas de infraestructa a los que pueda limitarnos nuestra máquina. Se van a utilizar **4 máquinas**:

1. **Amazon Linux 2:** para instalar el servidor de Wazuh.
2. **Ubuntu:** como víctima endpoint que está ejecutando un agente de Wazuh. Correrá un servidor Apache para aplicaciones PHP.
3. **Windows Server 2022:** otra víctima endpoint ejecutadno otro agente de Wazuh. Correrá un servidor IIS para aplicaciones ASP.NET.
4. **Debian:** como máquina atacante.
 
### Wazuh server en Amazon Linux 2

La instalación del servidor de Wazuh o Wazuh manager se puede realizar siguiendo [este](https://documentation.wazuh.com/current/quickstart.html) sencillo tutorial.

### Agente Wazuh en Ubuntu 22.04

Los pasos vienen detallados [aquí](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html)

!!!warning "Atención"
        Elegid la pestaña ***APT***, que es la que os dará las indicaciones para realizar las acciones con el gestor de paquetes correspondiente.

### Agente de Wazuh en Windows Server 2022

En este caso las instrucciones están [aquí](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html) y os dará la opción, en las pestañas, de elegir una instalación gráfica o por línea de comandos.

!!!Warning "Atención"
        Recordad poner como IP de Wazuh manager la de vuestro server de Wazuh.


### Servidor web Apache en endpoint Ubuntu

Pasos a seguir:

1. Instalar Apache:

    ```console
    $ sudo apt-get update
    $ sudo apt-get install apache2
    ```

2. Instalar PHP 8.1 para poder correr aplicaciones PHP

    ```console
    $ sudo apt-get install --no-install-recommends php8.1
    ```

    Así evitaremos instalar paquetes adicionales.

3. Para verificar la instalación podemos acceder a la URL: `http://IP_UBUNTU_ENDPOINT`y nos mostrará la página por defecto de Apache.


### Servidor web IIS en endpoint Windows Server

Para proceder con esta instalación:

1. En el menú de inicio de Windows, escribimos `appwiz.cpl` y le decimos *Turn Windows features on or off*:

    ![](../img/windows-server-wazuh2.png)

2. En *Server Roles* instalamos **Web Server (IIS)** conm, al menos, las siguientes funciones:
    
    ![](../img/windows-server-wazuh.png)

3. Para verificar la instalación accedemos a la URL: `http://IP_WINDOWS_ENDPOINT`

## Escenario hipotético

El endpoint Ubuntu corre un Apache con PHP instalado y el endpoint Windows Server corre un servidor web IIS, capaz de interpretar código ASP.NET.

Puesto que las web shells se consideran malware post-explotación, hemos de asumir que el atcante ya posee acceso inicial a los endpoints. Lo que el atacante desea conseguir es la persistencia en el sistema comprometido con el fin de llevar a cabo estas labores de  post-explotación.

## Técnicas de detección

Utilizaremos distintas capacidades de Wazuh para detectar la presencia de web shells en PHP o ASP.NET.

### Integridad de ficheros

Utilizaremos **FIM (File Integritiy Monitorint)** para deteta rla creación y modificación de archivos que contengan web shells.

El módulo FIM de Wazuih puede detectar, casi en tiempo real, cambios en los archivos accesibles via web y de esta forma alertar a los administradores.

Usaremos este módulo para detectar cuando se han creado o odificado archivos en `/var/wwww/html`y en `C:\inetpub\wwwroot`, directorios raiz por defecto en Ubuntu y Windows respectivamente.

Además, FIM escanea los contenidos de los archivos para monitorizar la aparción de firmas de web shells cuando los archivos se modifican.

#### Configuración de Ubuntu

1. Añadir la siguiente configuración al agente de Wazuh en el archivo `/var/ossec/etc/ossec.conf`, dentro del bloque `<syscheck>`:

    ```html
    <directories realtime="yes" check_all="yes" report_changes="yes">/var/www/html</directories>
    ```
    Esto detecta los cambios en el directorio `/var/www/html`.

2. Reiniciar el agente de Wazuh para aplicar los cambios en la configuración:

    ```console
    $ sudo systemctl restart wazuh-agent
    ```
