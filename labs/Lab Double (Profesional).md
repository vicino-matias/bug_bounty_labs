
# Bug Bounty Labs - Double

> **Autor**: vicino_matias
> **Fecha**: 11 de Julio de 2025
> **Nivel**: Profesional
> **Categoria**: Web / XSS Reflejado
> **Herramientas**: Nmap, Gobuster, Burp Suite, Whatweb, Navegador.

Hola, un placer que estén aquí leyendo esto. El día de hoy estaremos viendo como resolver la maquina Double de Bug Bounty Labs.

## 1- Descargamos e iniciamos la maquina

Existen 2 formas de hacerlo, mediante VM o Docker. Yo usare Docker:

Recuerda que, al descargarlo, tienes que descomprimirlo. Esto te generara un .py.

Este py lo corremos con este comando:

```
sudo python3 bugbountylabs_double.py
```

![](Ejecucion_Docker.png)

Al confirmar la instalación por consola, se nos hará el contenedor y nos proporcionara una IP. Esta IP es la que usaremos durante todo el laboratorio.
![](IP.png)

## 2- Enumeración

Escaneamos la ip en busca de puertos y servicios con nmap. El comando que **yo** use es este (puedes usar uno diferente si asi lo deseas):
```
sudo nmap -sCV -vvv -p- --min-rate 5000 -Pn 172.17.0.2 -oN escaneo_double
```

**Desglose**:

>**sudo nmap**: Usamos sudo para poder escanear mas allá de 1024 puertos.

>**-sCV**: es la abreviatura de -sC y -sV. Lo que hará sera usar scripts de detección por defecto e intentara detectar las versiones de los servicios que encuentre.

>**-vvv**: triple verbosidad. Ideal para saber que esta haciendo en todo momento.

>**-p- y --min-rate 5000**: Esto hace que escanee todos los puertos (desde el 1 al 65535) y fuerza al escaneo a enviar 5000 paquetes por segundo. Esto lo hace mas rápido, pero a la vez mas ruidoso.

>**-Pn**: Desactiva el ping inicial. En este caso asumimos que el host esta UP (vivo). Si no fuese el caso, esto simplemente no se agrega.

>  **-oN**: esto creara un archivo en la carpeta donde esta haciendo el escaneo para poder consultarse posteriormente si es necesario. Buena practica para no repetir escaneos.

Este comando nos dará como resultado esto:

![](Escaneo_terminado.png)

Lo que nos quiere decir es que hay un servicio web abierto (HTTP) pero lo mas importante es que esta en el puerto 5000 y que esta corriendo Python. Esto sera útil para mas adelante.

Hasta entonces, primero nos aseguraremos que hay corriendo un servicio web (y que no tiene algun otro dominio que no sea http://172.17.0.2:5000). Esto lo haremos con whatweb, una herramienta sencilla que nos permitira chequear esto (siempre podemos optar por la opcion de ingresar por el navegador y saltearnos este paso.)

![](whatweb.png)
**Importante**: usualmente whatweb es un comando que se usa como ```whatweb 172.17.0.2``` ya que usualmente los servicios web corren en el puerto 80. No es este el caso, asique tenemos que aclarar el puerto donde se ubica con ```:5000``` al final de la IP.

## 3- Análisis de la pagina web

Cuando ingresamos a la pagina web mediante nuestro navegador, nos damos cuenta que hay un simple formulario donde podemos probar inyecciones XSS (Cross-Site Scripting).

![](Form_principal.png)

Intentaremos usar algún payload de los muchos que hay (puedes buscarlos en este repositorio si lo deseas: https://github.com/payloadbox/xss-payload-list/blob/master/Intruder/xss-payload-list.txt). Yo usare este:  ```<script>alert()</script>```

![](Intento_fallido.png)

Pero vemos que no funciona, o al menos no como debería funcionar. Al parecer nuestro payload esta siendo alterado haciendo que por pantalla nos muestre un 2 y una etiqueta del Minecraft.

Podemos seguir probando con muchos payloads, pero no llegaríamos a nada. Vamos a usar Burp Suite a ver si encontramos algo interesante.

Ejecutamos Burp Suite, activamos Foxy Proxy para la captura y capturamos lo mismo que hemos enviado, luego lo enviamos al repeater de Burp:

![](Burp_Captura_Analisis.png)

Como vemos, esto al parecer sanitiza nuestra entrada por el form, haciendo que este se visualice tal cual lo mandamos (y ejecute el comando como es debido). Podemos analizar su codigo fuente con Ctrl + U en el navegador
```
document.getElementById('xss-form').addEventListener('submit', function (e) {
  e.preventDefault();
  const name = document.getElementById('name').value;
  const xhr = new XMLHttpRequest();
  xhr.open('POST', '/submit', true);
  xhr.setRequestHeader('Content-Type', 'application/json');
  xhr.send(JSON.stringify({ name }));
  xhr.onload = function () {
    if (xhr.status === 200) {
      const res = JSON.parse(xhr.responseText);
      document.getElementById('response-container').innerHTML = res.message;
    }
  };
});

```
 Y esto nos dirá que, lo que enviamos, esta siendo sanitizado por el Backend, no por el Front. Vamos al siguiente paso.

## 4- Fuzzing de rutas alternativas

Vamos a usar gobuster para intentar hacer fuzzing a la pagina principal y ver que encontramos. El comando que usare es este: 
```gobuster dir -u http://172.17.0.2:5000/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x py,webp,html,css,php,js,txt```

**Desglose**:

> **gobuster dir -u**: llamamos a gobuster y le decimos que haremos un fuzzing de la siguiente direccion (ingresamos la direccion tal cual nos sale en el navegador)
>  **-w**: usamos esta flag para indicar que usaremos una **wordlist** de nuestra eleccion, en mi caso usare una que viene en mi Parrot OS, en seclists.
>  **-x**: esta flag sirve para indicar que extensiones de rutas queremos encontrar, en este caso le pase las siguientes: py, webp, html, css, php, js y txt.  Esto hara que solo se centre en ellas e ignore el resto.

![](Gobuster_scan.png)
> **IMPORTANTE**: Terminar el escaneo previamente con Ctrl + C, porque empezara a soltar errores de rutas que no llevan a nada.

Esto nos muestra que hay corriendo una ruta que se llama app.py, por lo que nos meteremos ahí:

http://172.17.0.2/app.py

## 5- Acceso al backend

Al meternos nos encontraremos con esto:
![](backend_page.png)
## 6- Payload exitoso

Esto nos confirma que, en el backend, se esta sanitizando nuestros envíos por el form de la pagina **pero** hay un condicional IF que nos dice que, si el envío que hagamos empieza con ">">< nos devolverá un mensaje de felicitaciones, que hemos explotado el XSS correctamente. Asique mandamos el mismo payload de antes solo que iniciando con ">">< y el resultado es este:

![](Finalizacion.png)

Y listo, hemos terminado con este laboratorio.

Muchas gracias por leer y seguir hasta aquí. ¡Muy buena suerte en tu viaje como Bug Bounty Hunter!