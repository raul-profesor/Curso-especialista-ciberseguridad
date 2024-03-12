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

### Agente Wazuh en Ubuntu

Los pasos vienen detallados [aquí](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html)

!!!warning "Atención"
        Elegid la pestaña ***APT***, que es la que os dará las indicaciones para realizar las acciones con el gestor de paquetes correspondiente.

### Agente de Wazuh en Windows Server

En este caso las instrucciones están [aquí](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html) y os dará la opción, en las pestañas, de elegir una instalación gráfica o por línea de comandos.

!!!Warning "Atención"
        Recordad poner como IP de Wazuh manager la de vuestro server de Wazuh.







