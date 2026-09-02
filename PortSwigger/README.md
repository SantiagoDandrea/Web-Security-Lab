# Labs PortSwigger

## Reflected XSS

### Reflected XSS into HTML context with nothing encoded

**Descripción:** Realizar un ataque XSS reflejado que permita ejecutar la función `alert()`.

**Análisis**

La página tiene un buscador que refleja directamente el contenido de la búsqueda dentro del HTML de la respuesta, sin realizar ningún tipo de codificación.

Al no tratar el contenido ingresado como texto, es posible insertar etiquetas HTML que contengan código JavaScript.

**Payload / exploit**

Escribí dentro del buscador:

```text
<script>alert()</script>
```

La respuesta de la aplicación quedó conceptualmente:

```html
<p><script>alert()</script></p>
```

Al interpretar el HTML, el navegador reconoce la etiqueta `<script>` y ejecuta el código JavaScript contenido en ella.

![](./images/PortSwiggerReflectedXSS1.png)

**Resultado**

Fue posible ejecutar código JavaScript mediante una entrada reflejada en la respuesta de la aplicación, confirmando la existencia de una vulnerabilidad de Reflected XSS.

**Mitigación**

La principal mitigación es realizar **output encoding** de los datos antes de insertarlos en la respuesta, de manera que el contenido proporcionado por el usuario sea interpretado como texto y no como código HTML o JavaScript.

En este caso, caracteres especiales como `<` y `>` deberían codificarse antes de incluir el valor de búsqueda en el HTML.

También es recomendable utilizar una **Content Security Policy (CSP)** como capa adicional de defensa, aunque no sustituye al output encoding adecuado.

Referencia: [PortSwigger Web Security Academy – Reflected XSS](https://portswigger.net/web-security/cross-site-scripting/reflected)
