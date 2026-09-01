# Vulnerabilidades de DVWA
## SQL Injection — Low

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

**Resultado**

Fue posible modificar la consulta SQL y obtener los datos de todos los usuarios de la tabla.

**Mitigación**

La vulnerabilidad se puede evitar utilizando consultas parametrizadas / prepared statements, de manera que la entrada del usuario sea tratada como un dato y no pueda modificar la estructura de la consulta SQL.

Referencia: OWASP A05:2025 – Injection
