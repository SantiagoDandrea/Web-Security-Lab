# Vulnerabilidades de DVWA
## SQL Injection - Low

**Categoría OWASP:** A05:2025 - Injection

**Descripción:** Manipular la consulta SQL del módulo de SQL Injection para obtener información de múltiples usuarios.

**Análisis**

Primero probé el funcionamiento normal introduciendo:

```
1
```

La aplicación devolvió los datos correspondientes al usuario con id = 1.

![](./images/DVWAp1.png)

La consulta utilizada por la aplicación es:

```sql
SELECT first_name, last_name FROM users WHERE user_id = '$id';
```

El problema, al igual que se vio en las inyecciones de Juice Shop, es que el valor proporcionado por el usuario se concatena directamente dentro de la consulta SQL.

**Payload / exploit**

Utilicé el siguiente payload:

```
1' OR '1'='1' #
```

El payload cierra la comilla del parámetro id, agrega una condición siempre verdadera y utiliza # para comentar el resto de la consulta, entonces el WHERE es verdadero en todas las filas y la aplicación devuelve todos los usuarios.

![](./images/DVWAp2.png)

## SQL Injection - Medium

**Categoría OWASP:** A05:2025 - Injection

**Descripción:** Explotar la protección implementada en el nivel Medium para obtener información de múltiples usuarios.

**Análisis**

A diferencia de Low, el formulario utiliza un menú desplegable y envía el parámetro mediante `POST`. Además, el código aplica `mysqli_real_escape_string()` sobre el valor recibido.

La petición manda el `id` junto con un string `Submit` que podemos modificar.

![](./images/DVWAsqliM1.png)

Sin embargo, la consulta se construye de la siguiente manera:

```sql
SELECT first_name, last_name FROM users WHERE user_id = $id;
```

A diferencia de Low, el parámetro id no está entre comillas. Por lo tanto, aunque se aplique mysqli_real_escape_string(), todavía es posible introducir una expresión SQL válida.

**Payload / exploit**

Modifiqué la petición POST para enviar:

```
1 OR 1=1
```
![](./images/DVWAsqliM2.png)

La consulta resultante queda:

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = 1 OR 1=1;
```

Como 1=1 siempre es verdadero, la aplicación devolvió los datos de los cinco usuarios.

**Resultado**

Fue posible modificar la condición WHERE y obtener los datos de todos los usuarios.

![](./images/DVWAsqliM2res.png)

**Mitigación**

La vulnerabilidad se puede evitar utilizando consultas parametrizadas / prepared statements, de manera que la entrada del usuario sea tratada como un dato y no pueda modificar la estructura de la consulta SQL.

Referencia: OWASP A05:2025 – Injection

## SQL Injection - UNION-based - Medium

Descripción: Utilizar una inyección basada en UNION para obtener información adicional de la tabla users.

**Análisis**

Una vez comprobada la inyección anterior, utilicé UNION SELECT para consultar otras columnas de la tabla users.

Como la consulta original devuelve dos columnas (first_name y last_name), el UNION también debía devolver dos columnas.

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

De esta forma, la primera columna del resultado muestra el nombre de usuario y la segunda la contraseña almacenada.

![](./images/UNIONsqliM1.png)

**Resultado**

Fue posible utilizar la vulnerabilidad para obtener los nombres de usuario y las contraseñas almacenadas de los cinco usuarios de la base de datos.

**Mitigación**

La mitigación es la misma indicada en el apartado anterior: utilizar consultas parametrizadas / prepared statements para evitar que la entrada del usuario pueda modificar la estructura de la consulta SQL.

Referencia: OWASP A05:2025 – Injection


