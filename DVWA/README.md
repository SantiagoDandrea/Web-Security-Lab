# Vulnerabilidades de DVWA

## Tabla de Contenidos
- [SQL Injection - Low](#sql-injection---low)
- [SQL Injection - Medium](#sql-injection---medium)
- [SQL Injection - UNION-based - Medium](#sql-injection---union-based---medium)
- [SQL Injection - UNION-based - High](#sql-injection---union-based---high)
- [SQL Injection (Blind) - High](#sql-injection-blind---high)
- [Remediación en PHP y MySQL](#remediación-en-php-y-mysql)

---

## SQL Injection - Low

**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Manipular la consulta SQL del módulo de SQL Injection para obtener información de múltiples usuarios.

**Análisis**

Primero probé el funcionamiento normal introduciendo:

```text
1
```

La aplicación devolvió los datos correspondientes al usuario con id = 1.

![](./images/DVWAp1.png)

Al revisar el código fuente, la consulta utilizada por la aplicación es:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '$id';
```

El problema es que el valor ingresado por el usuario se concatena directamente dentro de la consulta SQL sin ningún tipo de validación ni parametrización.

**Payload / exploit**

Utilicé el siguiente payload:

```text
1' OR '1'='1' #
```

El payload cierra la comilla del parámetro `id`, agrega una condición que siempre da verdadero (`'1'='1'`) y usa `#` para comentar el resto de la consulta original. Como resultado, la condición `WHERE` se cumple en todas las filas y la aplicación devuelve todos los usuarios de la base de datos.

![](./images/DVWAp2.png)

**Resultado**

Fue posible modificar la consulta y volcar la información de todos los usuarios registrados.

**Mitigación**

Implementar **consultas preparadas (Prepared Statements)**. Al preparar la consulta, la base de datos separa la estructura SQL del valor del usuario, tratando la entrada estrictamente como un dato y no como código ejecutable. *(Ver ejemplos de código al final).*

**Referencia:** [OWASP Top 10:2021 – A03: Injection](https://owasp.org/Top10/A03_2021-Injection/)

---

## SQL Injection - Medium

**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Explotar la protección implementada en el nivel Medium para obtener información de múltiples usuarios.

**Análisis**

A diferencia de Low, el formulario utiliza un menú desplegable y envía el parámetro mediante `POST`. Además, el código fuente muestra que aplica `mysqli_real_escape_string()` sobre el valor recibido.

La petición manda el `id` junto con un string `Submit` que podemos interceptar y modificar en Burp Suite:

![](./images/DVWAsqliM1.png)

Sin embargo, al mirar cómo se construye la consulta en el servidor:

```sql
SELECT first_name, last_name FROM users WHERE user_id = $id;
```

**Por qué falló la protección:** En este nivel, el parámetro `$id` se incorpora como un literal numérico sin comillas delimitadoras en la consulta (`WHERE user_id = $id`). La función `mysqli_real_escape_string()` está diseñada para escapar caracteres especiales de cadenas (como comillas simples, dobles o bytes nulos), pero **no neutraliza inyecciones cuando el valor se concatena en un contexto numérico**. Como el payload inyectado (`1 OR 1=1`) no requiere comillas ni caracteres especiales para alterar la lógica booleana de la sentencia, la función de escape no impide que la expresión SQL se ejecute directamente.

**Payload / exploit**

Modifiqué la petición POST para enviar:

```text
1 OR 1=1
```

![](./images/DVWAsqliM2.png)

La consulta resultante queda:

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = 1 OR 1=1;
```

Como `1=1` siempre es verdadero, la aplicación devolvió los datos de los cinco usuarios.

**Resultado**

Fue posible alterar la condición `WHERE` y obtener los datos de todos los usuarios sin necesitar comillas ni caracteres especiales.

![](./images/DVWAsqliM2res.png)

**Mitigación**

No confiar en funciones de escape de cadenas para proteger campos numéricos. La solución de raíz consiste en:
1. **Prepared Statements (Solución principal):** Parametrizar la consulta vinculando el valor con tipado entero estricto. Esto delega la separación entre código y datos al motor de base de datos.
2. **Validación y tipado de entrada (Defensa en profundidad):** Validar que la entrada cumpla estrictamente el formato numérico esperado (mediante `is_numeric()` o casteo explícito a entero `(int)$_POST['id']`) antes de su procesamiento. Un casteo o validación complementa la seguridad, pero no debe considerarse un sustituto universal de la parametrización de consultas.

**Referencia:** [OWASP Top 10:2021 – A03: Injection](https://owasp.org/Top10/A03_2021-Injection/)

---

## SQL Injection - UNION-based - Medium

**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Utilizar una inyección basada en `UNION` para obtener datos de otras columnas de la tabla `users`.

**Análisis**

Una vez confirmada la inyección numérica en este nivel, utilicé `UNION SELECT` para consultar datos adicionales de la base de datos.

Para que una consulta `UNION` funcione, debe devolver el mismo número de columnas que la consulta original. Como la consulta original devuelve dos columnas (`first_name` y `last_name`), el `UNION` también debe pedir exactamente dos columnas.

**Payload / exploit**

Utilicé:

```sql
1 UNION SELECT user, password FROM users
```

La consulta resultante queda conceptualmente como:

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = 1
UNION
SELECT user, password FROM users;
```

De esta forma, en el campo de `First name` la aplicación muestra el nombre de usuario (`user`) y en `Surname` muestra el hash de la contraseña almacenada (`password`).

![](./images/UNIONsqliM1.png)

**Resultado**

Fue posible extraer los nombres de usuario y los hashes de contraseñas de los cinco usuarios de la base de datos dentro de la misma respuesta.

**Mitigación**

Al implementar **consultas preparadas**, el motor de base de datos compila previamente el plan de ejecución de la consulta. La entrada del usuario se vincula exclusivamente como un valor literal para el parámetro evaluado en la cláusula `WHERE`, impidiendo que operadores como `UNION` o palabras clave SQL modifiquen la estructura de la sentencia. Cabe destacar que la parametrización protege únicamente los valores de datos; si se construyen dinámicamente identificadores (como nombres de tablas o columnas), estos deben ser validados mediante listas blancas (*allowlists*).

**Referencia:** [OWASP Top 10:2021 – A03: Injection](https://owasp.org/Top10/A03_2021-Injection/)

---

## SQL Injection - UNION-based - High

**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Explotar la SQL Injection del nivel High para obtener los hashes de contraseñas de los usuarios superando intermediaciones de sesión y la cláusula `LIMIT 1`.

**Objetivo**

El apartado **Help** indica que existen cinco usuarios en la base de datos y que el objetivo es obtener sus credenciales (hashes almacenados) mediante SQL Injection.

**Análisis**

En este nivel, el valor ingresado se envía primero a una ventana secundaria (`session-input.php`) que almacena el input en la sesión del usuario (`$_SESSION['id']`). Después, `high.php` toma ese valor de la sesión para ejecutar la consulta:

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = '$id'
LIMIT 1;
```

**Dos puntos clave de este nivel:**
1. **Guardar en sesión no sanitiza:** Guardar el dato temporalmente en `$_SESSION` no limpia el input. Como el valor sigue concatenándose directamente en la consulta, la vulnerabilidad sigue estando ahí.
2. **Eliminar el `LIMIT 1`:** La consulta intenta limitar el resultado a una sola fila con `LIMIT 1`. Pero como controlamos lo que va en `$id`, podemos usar un comentario (`-- -`) para descartar el resto de la consulta y anular ese límite.

**Payload / exploit**

Utilicé el siguiente payload:

```text
1' UNION SELECT user,password FROM users -- -
```

La consulta ejecutada en la base de datos queda:

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = '1'
UNION
SELECT user,password FROM users
-- - LIMIT 1;
```

Al utilizar `-- -`, se descarta la cláusula `LIMIT 1` de la consulta original. Esto permite que el bucle `while` en el código PHP del backend procese e imprima en pantalla todas las filas devueltas por el `UNION SELECT`.

![](./images/DVWAsqliH1.png)

**Resultado**

Fue posible obtener los cinco nombres de usuario y los hashes de contraseñas almacenados en la base de datos, cumpliendo el objetivo del ejercicio.

**Mitigación**

Cualquier dato proporcionado por el usuario o recuperado de almacenamiento intermedio (como `$_SESSION`) debe vincularse como parámetro tipado en el punto de ejecución de la consulta. El almacenamiento temporal en variables de sesión no constituye un mecanismo de sanitización ni neutraliza la inyección.

**Referencia:** [OWASP Top 10:2021 – A03: Injection](https://owasp.org/Top10/A03_2021-Injection/)

---

## SQL Injection (Blind) - High

**Categoría OWASP:** A03:2021 – Injection

**Descripción:** Utilizar una Blind SQL Injection booleana para inferir y extraer la versión exacta del software de base de datos mediante respuestas de tipo verdadero/falso.

**Objetivo**

El apartado **Help** indica que el objetivo es identificar la versión del software de base de datos mediante Blind SQL Injection.

**Análisis**

A diferencia de los niveles anteriores, la aplicación no muestra datos de la tabla ni mensajes de error de SQL en pantalla. En su lugar, toma el valor de la cookie `id` y solamente nos dice:

- Si la consulta encuentra al menos un resultado:
  ```text
  User ID exists in the database.
  ```
- Si la consulta no encuentra resultados:
  ```text
  User ID is MISSING from the database.
  ```

Esto nos da un oráculo de verdadero o falso: podemos hacer preguntas lógicas a la base de datos y deducir las respuestas según si la aplicación dice que el usuario existe o no.

**Payload / exploit**

Primero comprobé este comportamiento probando condiciones básicas conocidas como verdaderas y falsas (`' OR '1'='1` vs `' OR '1'='2`) para confirmar que la respuesta de la aplicación servía como indicador.

Luego utilicé la variable `@@version` de MySQL/MariaDB y empecé a hacer preguntas usando el operador `LIKE`:

```text
' OR @@version LIKE '1%' -- -
```

La respuesta fue `User ID exists`, indicando que la versión efectivamente empieza con `1`.

![](./images/DVWABH1.png)

A partir de ahí, fui probando carácter por carácter:

```text
10 -> exist
10. -> exist
10.1 -> exist
10.11 -> exist
10.11. -> exist
10.11.1 -> exist
```

hasta llegar a:

```text
' OR @@version LIKE '10.11.19%' -- -
```

donde la aplicación volvió a responder que el usuario existía.

De esta forma pude reconstruir el prefijo completo de la versión:

```text
10.11.19-
```

La versión identificada fue **MariaDB 10.11.19**.

![](./images/DVWABH2.png)

**Resultado**

Fue posible identificar que el motor de base de datos es **MariaDB 10.11.19** sin que la aplicación mostrara directamente el contenido de `@@version`, únicamente evaluando respuestas de verdadero y falso.

**Mitigación**

1. **Consultas parametrizadas:** Al tratar el valor ingresado estrictamente como un dato literal y no como código SQL ejecutable, la base de datos busca una coincidencia exacta para la cadena completa y no evalúa la condición lógica inyectada (`OR @@version LIKE ...`), neutralizando por completo el oráculo booleano.
2. **Principio de mínimo privilegio (Defensa en profundidad):** Configurar la cuenta de base de datos utilizada por la aplicación web con los privilegios mínimos necesarios para su operación (ej. `SELECT`, `INSERT`, `UPDATE` en tablas de negocio), restringiendo el acceso a variables del sistema, funciones administrativas o tablas de metadatos (`information_schema`).

**Referencia:** [OWASP Top 10:2021 – A03: Injection](https://owasp.org/Top10/A03_2021-Injection/)

---

## Remediación en PHP y MySQL

Para solucionar estas vulnerabilidades en el código de DVWA, la medida principal es reemplazar la concatenación dinámica por **consultas preparadas (Prepared Statements)**.

### 1. Parche directo sobre el código de DVWA (MySQLi)

En los niveles vulnerables de DVWA (`low.php` y `medium.php`), el código original toma la entrada del usuario y la concatena en la variable `$query` antes de pasarla a `mysqli_query`:

```php
<?php
// Código vulnerable original (DVWA medium.php)
$id = $_POST['id'];
$id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

$query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
$result = mysqli_query($GLOBALS["___mysqli_ston"], $query);
?>
```

Para corregirlo manteniendo la librería MySQLi que utiliza DVWA, se debe sustituir por `mysqli_prepare()` y vincular el parámetro con tipo entero (`"i"`):

```php
<?php
// Versión corregida con consultas preparadas (MySQLi)
$id = $_POST['id'];

// 1. Preparar la plantilla de la consulta con el marcador de posición '?'
$stmt = mysqli_prepare($GLOBALS["___mysqli_ston"], "SELECT first_name, last_name FROM users WHERE user_id = ?");

if ($stmt) {
    // 2. Vincular el parámetro forzando tipo entero ('i')
    mysqli_stmt_bind_param($stmt, "i", $id);
    
    // 3. Ejecutar la consulta
    mysqli_stmt_execute($stmt);
    $result = mysqli_stmt_get_result($stmt);

    while ($row = mysqli_fetch_assoc($result)) {
        echo "First name: " . htmlspecialchars($row['first_name'], ENT_QUOTES, 'UTF-8') . "<br>";
        echo "Surname: " . htmlspecialchars($row['last_name'], ENT_QUOTES, 'UTF-8') . "<br>";
    }
    
    mysqli_stmt_close($stmt);
}
?>
```

### 2. Implementación con PDO (Enfoque del nivel Impossible de DVWA)

El propio nivel **Impossible** de DVWA resuelve la inyección migrando a PDO y combinando validación de entrada (`is_numeric`) con consultas preparadas:

```php
<?php
// Enfoque del nivel Impossible de DVWA con PDO
$id = $_GET['id'];

if (is_numeric($id)) {
    $id = intval($id);

    // Preparar y ejecutar con PDO vinculando el parámetro tipado
    $stmt = $pdo->prepare('SELECT first_name, last_name FROM users WHERE user_id = :id LIMIT 1');
    $stmt->bindParam(':id', $id, PDO::PARAM_INT);
    $stmt->execute();
    $row = $stmt->fetch(PDO::FETCH_ASSOC);

    if ($row) {
        echo "First name: " . htmlspecialchars($row['first_name'], ENT_QUOTES, 'UTF-8') . "<br>";
        echo "Surname: " . htmlspecialchars($row['last_name'], ENT_QUOTES, 'UTF-8') . "<br>";
    } else {
        echo "User ID is MISSING from the database.";
    }
} else {
    echo "User ID is MISSING from the database.";
}
?>
```

### ¿Qué ocurre al enviar los payloads contra el código corregido?

Si intentamos ejecutar los mismos payloads de los niveles anteriores contra la consulta preparada:
1. Al enviar `1 OR 1=1` o `1' OR '1'='1' #`:
   * En MySQLi con `bind_param("i", ...)` o PDO con `PDO::PARAM_INT`, el motor de base de datos no interpreta el texto como operadores SQL ni como delimitadores. Lo trata estrictamente como el valor literal del campo `user_id`.
   * Si se aplica `intval()` o validación numérica, los caracteres que no sean dígitos son descartados o la consulta es rechazada.
2. Al enviar `1 UNION SELECT user, password FROM users`:
   * Como la estructura de la consulta ya fue compilada por la base de datos de forma previa, el operador `UNION` no se evalúa como una instrucción secundaria.
   * La base de datos busca un usuario cuyo identificador coincida literalmente con esa cadena completa. Al no existir ningún registro con ese ID, la consulta devuelve 0 filas, neutralizando la extracción de datos por completo.

