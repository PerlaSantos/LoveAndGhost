# LoveAndGhost — Demostración de SQL Injection

**Instituto Politécnico Nacional**
**Escuela Superior de Cómputo (ESCOM)**
**Asignatura: Bases de Datos**

| Integrantes | 
|---|
| Alarcon Herrera Julio Alexis |
| Cedillo Baeza Martha Clara |
| Santos Martinez Perla |

---

## Descripción del proyecto

LoveAndGhost es una aplicación web ficticia de citas desarrollada con el propósito académico de ilustrar, de forma práctica y visual, una de las vulnerabilidades de seguridad más comunes y peligrosas en el desarrollo de software: la inyección SQL (SQL Injection). El proyecto no tiene fines de producción ni uso real; su único objetivo es servir como herramienta educativa para comprender cómo funciona este tipo de ataque y cuáles son las medidas correctivas que deben aplicarse.

---

## Que es la SQL Injection

La inyección SQL (SQLi, por sus siglas en inglés) es una técnica de ataque que permite a un actor malicioso interferir en las consultas que una aplicación envía a su base de datos. Ocurre cuando una aplicación construye sus consultas SQL concatenando directamente el texto ingresado por el usuario, sin ningún proceso de validación o sanitización previo. Al explotar esta vulnerabilidad, un atacante puede leer datos no autorizados, modificar o eliminar registros, eludir mecanismos de autenticación e incluso obtener privilegios de administrador sobre el servidor de base de datos.

La Open Web Application Security Project (OWASP) clasifica la inyección SQL dentro de su lista OWASP Top 10, que agrupa las vulnerabilidades más críticas en aplicaciones web. En su edición de 2021, la categoría de inyección fue detectada en el 94% de las aplicaciones analizadas, con tasas de incidencia de hasta 19%, lo que la convierte en una de las amenazas más extendidas en entornos reales.

La gravedad de un ataque de inyección SQL depende de las capacidades del atacante y de las medidas de defensa implementadas. En general, se considera una vulnerabilidad de impacto alto, ya que puede comprometer la confidencialidad, integridad y disponibilidad de la información almacenada.

---

## Como funciona la inyeccion SQL

El mecanismo del ataque se fundamenta en la forma en que un backend vulnerable construye sus consultas. Considerese el siguiente ejemplo de un formulario de inicio de sesión cuyo código del lado del servidor genera la siguiente consulta:

```sql
SELECT * FROM usuarios WHERE nombre = 'valor_usuario' AND contrasena = 'valor_contrasena';
```

Cuando el desarrollador construye esta consulta mediante concatenación de cadenas, el texto que el usuario ingresa en el formulario se inserta directamente dentro de la instrucción SQL. Esto le otorga al usuario la capacidad de manipular la estructura lógica de la consulta.

### El payload clasico: tautologia con OR '1'='1'

Una tautología, en lógica, es una afirmación que siempre es verdadera. En el contexto de SQL Injection, un atacante aprovecha esta propiedad para modificar la cláusula WHERE de una consulta de modo que su evaluación siempre resulte en verdadero, independientemente de los valores originalmente esperados.

Si un atacante ingresa el siguiente texto en el campo de contraseña:

```
' OR '1'='1
```

La consulta que el servidor construye y ejecuta queda de la siguiente forma:

```sql
SELECT * FROM usuarios WHERE nombre = 'luna' AND contrasena = '' OR '1'='1';
```

El operador AND tiene precedencia sobre OR en SQL. Por lo tanto, la evaluación lógica de la cláusula WHERE procede así:

1. Se evalua `nombre = 'luna' AND contrasena = ''`, que resulta en falso porque la contraseña no coincide.
2. Se evalua el resultado anterior `OR '1'='1'`.
3. Como la condicion `'1'='1'` es siempre verdadera, el operador OR devuelve verdadero.
4. La consulta retorna registros de la tabla, lo que el servidor interpreta como una autenticacion exitosa.

El motor de base de datos devuelve uno o más registros, y la aplicación interpreta eso como que las credenciales son válidas, concediendo acceso sin que el atacante haya necesitado conocer la contraseña real.

---

## Estructura del proyecto

El repositorio contiene tres archivos principales:

- **`loveandghost.html`** — Aplicación web completa en un solo archivo. Incluye las páginas de inicio de sesión, registro y página principal con los candidatos. Integra el cliente de Supabase para operaciones reales contra la base de datos.
- **`loveandghost_db.sql`** — Script SQL compatible con el editor de Supabase (PostgreSQL). Crea las tablas `usuarios` y `perfiles`, inserta registros de prueba, y define dos funciones almacenadas: `login_vulnerable` y `login_seguro`, para demostrar el contraste entre un backend inseguro y uno correctamente parametrizado.
- **`README.md`** — Este documento.

---

## Como se muestra la vulnerabilidad en la pagina web

La aplicación implementa intencionalmente un backend simulado vulnerable con el fin educativo de hacer visible el problema. El flujo de la demostración es el siguiente:

**Paso 1.** Al ingresar cualquier texto en los campos de usuario y contraseña, la aplicación construye y muestra en pantalla la consulta SQL que un servidor vulnerable generaría con esos valores. Esto permite al observador ver en tiempo real cómo el input del usuario se inserta dentro de la instrucción SQL.

**Paso 2.** Si el texto ingresado contiene fragmentos propios de un payload de inyección (como comillas simples, el operador OR, guiones dobles u otras palabras reservadas de SQL), la aplicación muestra un aviso visual señalando que se detectó un posible payload.

**Paso 3.** Al ingresar el payload `' OR '1'='1` en el campo de contraseña, la aplicación muestra la consulta manipulada resultante y despliega un mensaje explicando que la vulnerabilidad fue explotada exitosamente: la condición tautológica hace que la autenticación sea omitida por completo.

**Paso 4.** A modo de contraste, el inicio de sesión real contra la base de datos Supabase se realiza mediante el método `.eq()` del cliente de Supabase, que internamente utiliza consultas parametrizadas (prepared statements). Al aplicar el mismo payload contra esta función, el texto se trata como un literal de cadena y no como código SQL, por lo que el ataque no tiene efecto.

**Paso 5.** En la base de datos, la función `login_vulnerable` replica el comportamiento del backend inseguro mediante concatenación de strings en PL/pgSQL, mientras que `login_seguro` utiliza parámetros vinculados. Ejecutar ambas con el mismo payload en el editor SQL de Supabase permite observar directamente la diferencia en el resultado.

---

## Medidas de prevencion

Las siguientes prácticas eliminan o mitigan de forma efectiva la vulnerabilidad de inyección SQL:

- **Consultas parametrizadas (prepared statements):** separan el código SQL de los datos del usuario. El motor de base de datos recibe el valor como un parámetro de tipo, no como código ejecutable, por lo que no puede alterar la estructura de la consulta.
- **ORMs y clientes de base de datos seguros:** herramientas como el cliente de Supabase, Sequelize, Prisma o Hibernate abstraen la construcción de consultas y aplican parametrización de forma automática.
- **Validacion y sanitizacion de entradas:** verificar el tipo, longitud y formato del input antes de procesarlo reduce la superficie de ataque.
- **Principio de minimo privilegio:** la cuenta de base de datos utilizada por la aplicación debe tener únicamente los permisos estrictamente necesarios para su operación, limitando el daño en caso de una explotación exitosa.
- **Web Application Firewall (WAF):** puede detectar y bloquear patrones de ataque conocidos como capa adicional de defensa.

---

## Tecnologias utilizadas

- HTML5, CSS3 y JavaScript (vanilla) para la interfaz de usuario
- Supabase (PostgreSQL) como base de datos y backend
- Supabase JavaScript Client v2 para las operaciones de base de datos parametrizadas
- Google Fonts (Dancing Script, Nunito) para la tipografia

---

## Referencias

OWASP Foundation. (s.f.). *SQL injection*. OWASP.
https://owasp.org/www-community/attacks/SQL_Injection

OWASP Foundation. (s.f.). *SQL injection prevention cheat sheet*. OWASP Cheat Sheet Series.
https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

OWASP Foundation. (s.f.). *Testing for SQL injection*. OWASP Web Security Testing Guide.
https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05-Testing_for_SQL_Injection

Acunetix. (2025, febrero 17). *What is SQL injection (SQLi) and how to prevent attacks*.
https://www.acunetix.com/websitesecurity/sql-injection/

PortSwigger. (s.f.). *Using SQL injection to bypass authentication*.
https://portswigger.net/support/using-sql-injection-to-bypass-authentication

Moxso. (2024, diciembre 9). *Tautology — SQL injection glossary*.
https://moxso.com/blog/glossary/tautology

RangeForce. (2026, febrero 24). *Why SQL injection remains one of the most persistent cyber threats*.
https://www.rangeforce.com/blog/sql-injection-isnt-going-anywhere

Vaadata. (2023, junio 27). *SQLi: principles, impacts and security best practices*.
https://www.vaadata.com/blog/sql-injections-principles-impacts-exploitations-security-best-practices/

Wikipedia. (s.f.). *SQL injection*.
https://en.wikipedia.org/wiki/SQL_injection
