# OWASP Juice Shop - Challenges
## Challenge: Score Board
**Categoría OWASP:** A05:2021 – Security Misconfiguration

**Descripción:** Encontrar la página oculta `Score Board`, donde se muestra el progreso de los challenges completados.

**Payload/exploit:**
No se utilizó un payload. La ruta fue descubierta analizando el archivo JavaScript principal de la aplicación mediante las herramientas de desarrollo del navegador: `/#/score-board`

**Resultado:** Al acceder a la ruta, se mostró la página `Score Board` con el listado de challenges y su estado.

**Análisis:**
La aplicación incluye en el código JavaScript del lado del cliente una referencia a una ruta que no está expuesta directamente mediante la interfaz. Al ser una aplicación web cliente, el archivo `main.js` contiene información sobre las rutas disponibles, incluida `/#/score-board`.

El challenge demuestra que ocultar una funcionalidad simplemente eliminando su enlace de la interfaz no impide que pueda ser descubierta. Las rutas presentes en el código enviado al navegador pueden ser analizadas y accedidas directamente.

**Mitigación:**
No se debe confiar en que una funcionalidad está protegida por no aparecer en la interfaz o por utilizar una URL difícil de descubrir. Si una página o funcionalidad requiere restricciones de acceso, estas deben aplicarse mediante controles de autenticación y autorización.

## Challenge: DOM XSS
**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Realizar un ataque DOM-Based Cross-Site Scripting utilizando el payload indicado por el challenge.

**Payload/exploit:** ``<iframe src="javascript:alert(`xss`)">``

El payload fue introducido en el campo de búsqueda de la aplicación.

**Resultado:**
Al procesarse la búsqueda, apareció una alerta con el texto `xss`, demostrando que fue posible ejecutar JavaScript controlado por el usuario en el navegador.

![](./images/DOM-XSS.png)
**Análisis:**
Una vulnerabilidad DOM XSS ocurre cuando datos controlados por el usuario son procesados por JavaScript en el navegador y posteriormente utilizados de forma insegura para modificar el DOM.

Para entender este tipo de vulnerabilidad se pueden identificar:
- **Source:** lugar desde el que se obtiene el dato controlado por el usuario.
- **Sink:** operación o función que utiliza ese dato y puede interpretarlo como HTML o código.

Por ejemplo:
```javascript
const input = location.hash; // Source
document.write(input);       // Sink peligroso
```
Si un atacante controla el contenido de `location.hash`, puede introducir código HTML malicioso. `document.write()` lo interpreta e inserta en la página.

En Juice Shop, el input se introduce mediante el campo de búsqueda. El problema se produce porque el valor ingresado termina siendo procesado mediante:
```
bypassSecurityTrustHtml(queryParam)
```
Esta función omite la sanitización de Angular para ese contenido y permite que el input sea tratado como HTML.

El payload crea un elemento `iframe` cuyo atributo `src` utiliza el esquema `javascript:`. Cuando el navegador procesa: ``javascript:alert(`xss`)`` ejecuta el código JavaScript y muestra la alerta.

**Mitigación:**
La solución específica consiste en eliminar el bypass de sanitización. El campo de búsqueda no necesita interpretar HTML, por lo que la entrada del usuario debería tratarse únicamente como texto.

De forma general, los datos no confiables no deberían insertarse en sinks peligrosos como `innerHTML` o `document.write()`. Cuando solo es necesario mostrar texto, se deben utilizar mecanismos seguros como: `element.textContent = userInput;`

**Referencia:** [ OWASP DOM Based XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com)
## Challenge: Bonus Payload
**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Utilizar un payload alternativo en la misma vulnerabilidad DOM XSS del challenge anterior.

**Payload/exploit:**
```html
<iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/771984076&color=%23ff5500&auto_play=true&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe>
```
El payload fue introducido en el mismo campo de búsqueda utilizado para explotar la vulnerabilidad DOM XSS.

**Resultado:**
La aplicación interpretó el payload como HTML y creó un `iframe` que cargó un reproductor de SoundCloud dentro de la página.

![](./images/SoundCloud-DOMXSS.png)

**Análisis:**
Este challenge utiliza la misma vulnerabilidad DOM XSS explicada anteriormente. La diferencia está en el payload utilizado.

En lugar de ejecutar una alerta, el payload aprovecha la posibilidad de inyectar HTML en el DOM para modificar el contenido visible de la página mediante un elemento `iframe`.

Esto demuestra que el impacto de una vulnerabilidad XSS no se limita a ejecutar `alert()`, sino que puede permitir modificar el contenido de la aplicación o cargar recursos externos.

**Mitigación:**
La mitigación es la misma que en el challenge anterior: eliminar el bypass de sanitización y evitar que los datos controlados por el usuario sean interpretados como HTML cuando la funcionalidad solo requiere mostrar texto.

**Referencia:** [OWASP DOM Based XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com)
## Challenge: Confidential Documents
**Categoría OWASP:** A05:2021 – Security Misconfiguration

**Descripción:** Acceder a un documento confidencial expuesto por la aplicación.

**Payload/exploit:**
No se utilizó un payload. En la página `About Us` se encontró un enlace hacia: /ftp/legal.md

Al eliminar `legal.md` de la URL, se accedió directamente al directorio: /ftp/

El servidor mostró un listado de documentos disponibles, entre ellos: `acquisitions.md`

Finalmente, se accedió al documento: /ftp/acquisitions.md

**Resultado:**
Fue posible acceder al documento confidencial `acquisitions.md`, completando el challenge.

![](./images/SensitiveContent.png)

**Análisis:**
La aplicación exponía públicamente el directorio `/ftp/`. El enlace a `legal.md` permitió descubrir la existencia de este directorio y, al acceder directamente a él, el servidor permitió listar los archivos almacenados.

Esto permitió enumerar documentos que no estaban destinados a ser accesibles públicamente y acceder directamente a uno de ellos.

El problema no consiste únicamente en que el archivo confidencial pueda descargarse, sino también en que fue almacenado dentro de una ubicación públicamente accesible y posteriormente olvidado allí.

**Mitigación:**
Los archivos confidenciales no deberían almacenarse dentro de directorios expuestos directamente por el servidor web. Es necesario clasificar los datos según su sensibilidad y almacenar cada tipo de información en una ubicación adecuada.

Cuando un documento sensible debe estar disponible para determinados usuarios, su acceso debería realizarse mediante mecanismos de autenticación y autorización.

También se deben revisar periódicamente los archivos y directorios expuestos públicamente para detectar información olvidada, accidentalmente publicada o que ya no debería estar disponible.

En este caso, la solución específica sería eliminar el directorio `/ftp/` público y trasladar su contenido legítimo a ubicaciones adecuadas. Los documentos confidenciales no deberían poder accederse directamente mediante una URL pública.

**Referencia:** [OWASP A05:2021 – Security Misconfiguration](https://cheatsheetseries.owasp.org/cheatsheets/File_System_and_Resource_Access_Cheat_Sheet.html?utm_source=chatgpt.com)
## Challenge: Exposed Metrics
**Categoría OWASP:** A09:2021 – Security Logging and Monitoring Failures

**Descripción:** Encontrar el endpoint que expone datos de uso para ser recolectados por un sistema de monitoreo popular.

**Payload/exploit:**
El enunciado mencionaba explícitamente a Prometheus. Al investigar el funcionamiento habitual de esta herramienta, se identificó que comúnmente utiliza el endpoint: `/metrics`

Se accedió directamente a: `http://localhost:3000/metrics`

**Resultado:**
El endpoint devolvió las métricas de la aplicación, completando el challenge.

![](./images/Metrics.png
)
**Análisis técnico:**
El challenge expone públicamente un endpoint de observabilidad utilizado para proporcionar métricas de la aplicación.

La mención de Prometheus en el enunciado permitió identificar una convención conocida de esta herramienta y probar directamente la ruta `/metrics`.

El problema es que cualquier usuario con acceso a la aplicación puede consultar información de monitoreo que normalmente debería estar disponible únicamente para sistemas o usuarios autorizados.

**Mitigación:**
Los endpoints de observabilidad no deberían estar accesibles públicamente cuando exponen información interna de la aplicación.

El acceso debería restringirse mediante controles como autenticación, autorización o limitaciones de red, permitiendo únicamente que los sistemas de monitoreo autorizados consulten las métricas.

También se debe revisar qué información se expone mediante estos endpoints y evitar incluir datos sensibles o detalles internos que puedan facilitar el reconocimiento de la aplicación.

**Referencia:** [OWASP A09:2021 – Security Logging and Monitoring Failures](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html?utm_source=chatgpt.com)

## Challenge: Login Admin
**Categoría OWASP:** A05:2025 - Injection
**Descripción:** Lograr un inicio de sesión exitoso al usuario administrador.

**Payload/exploit:**
En el campo de email del formulario de login permitió poner una condición SQL de la forma `' OR 1=1 --`
Este payload modifica la lógica de la consulta original para que la condición 1=1 sea siempre verdadera, permitiendo omitir la validación normal de las credenciales.

**Resultado:**
Logré iniciar sesión utilizando la cuenta administrador.

![](./images/login-admin.png)

**Análisis técnico:**
La aplicación es vulnerable a SQL Injection porque la entrada proporcionada por el usuario puede modificar la consulta SQL ejecutada por el servidor.

En lugar de ser tratada únicamente como un valor de email, la entrada se interpreta como parte de la consulta SQL. Al introducir una condición siempre verdadera, es posible alterar la lógica de autenticación y acceder a una cuenta sin conocer sus credenciales.

**Mitigación:**
La principal medida para prevenir SQL Injection es evitar la construcción de consultas SQL mediante concatenación dinámica de strings. En su lugar, deben utilizarse consultas parametrizadas o prepared statements, de forma que los datos proporcionados por el usuario se traten únicamente como valores y no puedan modificar la estructura de la consulta.

La validación de entradas también puede ayudar a reducir el riesgo, pero no debe utilizarse como única medida de protección. Además, las cuentas utilizadas por la aplicación para acceder a la base de datos deben tener únicamente los permisos necesarios.

**Coding Challenge**: 
Para complementar el challenge, realicé el Coding Challenge asociado a la vulnerabilidad.

La vulnerabilidad se encuentra en la construcción de la consulta SQL, donde los valores enviados por el usuario se insertan directamente mediante interpolación de strings:

![](./images/cod-inseguro-LAdmin.png)

```javascript 
models.sequelize.query( `SELECT * FROM Users WHERE email = '${req.body.email || ''}' AND password = '${security.hash(req.body.password || '')}' AND deletedAt IS NULL`, { model: UserModel, plain: true})
```
El problema es que la aplicación incorpora directamente la entrada del usuario dentro de la sintaxis SQL. Por lo tanto, caracteres como comillas (`'`) pueden modificar la estructura original de la consulta en lugar de ser tratados únicamente como parte del valor ingresado.

Por ejemplo, en el challenge **Login Admin**, al introducir el payload: `' OR 1=1 --` la entrada modifica la condición utilizada para buscar al usuario. La comilla cierra el valor original del email, `OR 1=1` agrega una condición siempre verdadera y `--` comenta el resto de la consulta.

*Mitigación*:

![](./images/cod-seguro-LAdmin.png)

En este caso, la estructura de la consulta SQL queda definida por separado y los valores del usuario se insertan posteriormente como parámetros. De esta forma, aunque el usuario introduzca caracteres especiales o código SQL, estos serán tratados como datos y no como parte de la sintaxis de la consulta.

**Referencia:** [OWASP A09:2021 – Security Logging and Monitoring Failures](https://owasp.org/Top10/2025/A05_2025-Injection/)

## Challenge: Login Bender
**Categoría OWASP:** A05:2025 - Injection  
**Descripción:** Lograr un inicio de sesión exitoso utilizando la cuenta de Bender.
**Payload / exploit**: 
Al encontrar la dirección de correo de Bender en la sección **About Us**, fue posible realizar una inyección SQL añadiendo `'--` al final del email. De esta forma, el comentario SQL evita que se evalúe el resto de la consulta, permitiendo iniciar sesión como Bender.

**Resultado**: 
Logré iniciar sesión utilizando la cuenta de Bender.

![](./images/login-bender.png)
**Análisis técnico y mitigación**: 
La vulnerabilidad explotada es la misma que en el challenge **Login Admin**: una SQL Injection que permite modificar la consulta de autenticación mediante datos proporcionados por el usuario. Por este motivo, el análisis técnico y las medidas de mitigación son los mismos.

**Referencia:** [OWASP A05:2025 – Injection](https://owasp.org/Top10/2025/A05_2025-Injection/)

## Challenge: Christmas Special
**Categoría OWASP:** A05:2025 - Injection
**Descripción:** Encontrar el producto eliminado *Christmas Special* de 2014, agregarlo al carrito y completar su compra.

**Payload / exploit**
El primer paso fue analizar cómo la aplicación realizaba las búsquedas de productos. Al buscar un producto y observar las peticiones en DevTools, encontré el endpoint:
```text
/rest/products/search?q=
```
Al introducir una comilla simple (`'`) en el parámetro `q`, la aplicación devolvió un error de SQLite, indicando que la entrada del usuario estaba afectando una consulta SQL.

Para entender la estructura de la consulta, busqué la implementación del endpoint en el repositorio de Juice Shop. Allí encontré:
```typescript
models.sequelize.query(
  `SELECT * FROM Products WHERE ((name LIKE '%${criteria}%' OR description LIKE '%${criteria}%') AND deletedAt IS NULL) ORDER BY name`
)
```

La consulta incorpora directamente el valor de `criteria` mediante interpolación de strings. Además, la condición:

```
AND deletedAt IS NULL
```
es la que oculta los productos eliminados.

Al utilizar el payload:
```
'))--
```
la comilla simple cierra el string de la búsqueda, los dos paréntesis cierran la estructura `WHERE ((` y `--` comenta el resto de la consulta, incluyendo la condición `deletedAt IS NULL`.

De esta forma, fue posible visualizar también los productos eliminados y encontrar _Christmas Special 2014_.

![](./images/deleted-item.png)
**Agregar el producto al carrito**: 
Luego anoté el ID del producto _Christmas Special 2014_. Como el frontend no permitía agregar directamente un producto eliminado al carrito, agregué un producto normal y observé la petición realizada en DevTools → Network.

La aplicación utilizaba una petición:
```
POST /api/BasketItems/
```

Copié la petición como cURL, la modifiqué reemplazando el `ProductId` por el ID del producto eliminado (10) y la ejecuté desde la terminal. De esta forma, el producto eliminado fue agregado al carrito.

![](./images/normal-Post.png)

![](./images/post-deletedItem.png)

Finalmente, completé la compra del producto desde el carrito.

**Resultado**
Logré encontrar el producto eliminado _Christmas Special 2014_, agregarlo al carrito y completar su compra.

![](./images/christmas-success.png)

**Análisis técnico y mitigación**: 
La vulnerabilidad se debe a que el valor de búsqueda proporcionado por el usuario se concatena directamente dentro de la consulta SQL mediante interpolación:
```
'%${criteria}%'
```
Esto permite que caracteres especiales introducidos por el usuario modifiquen la estructura de la consulta. En este caso, el payload permitió eliminar la condición que filtraba los productos con `deletedAt IS NULL`.

La mitigación consiste en evitar la construcción dinámica de consultas mediante concatenación o interpolación de strings y utilizar consultas parametrizadas. De esta forma, la entrada del usuario se trata únicamente como un dato y no puede modificar la sintaxis SQL.

**Referencia:** [OWASP A05:2025 – Injection](https://owasp.org/Top10/2025/A05_2025-Injection/)
