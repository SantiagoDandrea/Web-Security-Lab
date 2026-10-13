# Web Application Security Lab

Este repositorio centraliza mis prácticas y laboratorios de análisis de vulnerabilidades web, seguridad en el desarrollo y validación de datos. El objetivo principal es comprender cómo las fallas en el procesamiento de inputs y la mala configuración de aplicaciones permiten la explotación de sistemas, haciendo foco en la comprensión de la causa raíz y el análisis de su remediación técnica (Secure Coding).

## Estructura del Entorno

El laboratorio está dividido en tres plataformas de práctica, cada una con su propia documentación y evidencias (capturas y análisis):

### 1. [OWASP Juice Shop](./OWASP%20Juice%20Shop/README.md)
Análisis de vulnerabilidades en arquitecturas web modernas (Single Page Applications).
* **Foco:** Cross-Site Scripting (DOM XSS), exposición de datos sensibles y control de acceso roto (Broken Access Control).
* **Casos documentados:** Evasión de login para acceso a cuentas privilegiadas, inyecciones de código en integraciones de terceros, acceso a material sensible y manipulación de lógica de negocio.

### 2. [DVWA - Damn Vulnerable Web App](./DVWA/README.md)
Prácticas de inyección y fallas lógicas en entornos tradicionales (PHP/MySQL).
* **Foco:** Inyecciones SQL (Blind y UNION-based), fuerza bruta y vulnerabilidades en la validación de carga de formularios.
* **Casos documentados:** Extracción de esquemas de bases de datos mediante SQLi y bypass de mecanismos de autenticación mediante ataques de fuerza bruta.

### 3. [PortSwigger Web Security Academy](./PortSwigger/README.md)
Resolución de laboratorios avanzados de seguridad web utilizando Burp Suite.
* **Foco:** Manipulación de JSON Web Tokens (JWT), vulnerabilidades lógicas, manipulación de sesiones y evasión de filtros XSS.
* **Foco:** Manipulación de JSON Web Tokens (JWT), vulnerabilidades lógicas, manipulación de sesiones y evasión de filtros XSS.
* **Casos documentados:** Falsificación de firmas JWT para escalamiento de privilegios, modificación de parámetros de usuario en el lado del cliente, bypass de controles para ejecución de Cross-Site Scripting (Reflected, Stored y DOM) y exfiltración de cookies de sesión mediante Burp Collaborator.

## Herramientas y Metodología de Trabajo
Para el desarrollo de los laboratorios se utilizó el siguiente stack de herramientas y metodología de análisis:
* **Entorno y Aislamiento:** Docker para el despliegue local de contenedores aislados y reproducibles (DVWA y OWASP Juice Shop).
* **Análisis de Tráfico HTTP:** Burp Suite (Proxy para intercepción y análisis de flujo de peticiones HTTP, Repeater para manipulación y reenvío de parámetros, y Burp Collaborator en entornos de laboratorio para recepción y exfiltración de interacciones fuera de banda).
* **Inspección de Cliente:** Chromium DevTools (Inspección del DOM, depuración de scripts del cliente y seguimiento de sinks/sources en vulnerabilidades DOM-based).
* **Pruebas de API y Automatización:** cURL para repetición rápida y verificación de endpoints.
* **Revisión de Código Fuente:** Análisis estático de los repositorios y controladores vulnerables (`high.php` en DVWA, controladores Sequelize/Express en Juice Shop) para identificar la causa raíz de cada fallo a nivel de código.

## Enfoque Defensivo
Para cada caso práctico detallado en los subdirectorios, el análisis responde a tres criterios fundamentales:
1. **Detección:** ¿Cómo fluyen y se manipulan los datos entre el cliente y el servidor?
2. **Explotación:** ¿Qué falla de validación o lógica de negocio permitió la ejecución de la vulnerabilidad?
3. **Mitigación:** ¿Qué control de seguridad se requiere implementar (ej. *Output Encoding*, *Prepared Statements*, parametrización segura, CSP, atributos de sesión) para resolver el problema de raíz?

## Descargo de Responsabilidad (Disclaimer Ético)
Todas las pruebas, payloads y análisis documentados en este repositorio fueron ejecutados exclusivamente dentro de entornos locales intencionalmente vulnerables o plataformas de laboratorio autorizadas (PortSwigger Web Security Academy) con fines estrictamente pedagógicos, de investigación y desarrollo de capacidades defensivas. No se promueve ni avala el uso de estas técnicas contra sistemas o aplicaciones sin autorización previa y explícita.
