# Guion de exposición — Persona 4: Usuarios y Multi-tenancy

**Proyecto:** Trazora FSM · **Tema asignado:** MS0 Usuarios + aislamiento multi-tenant.  
**Presentación visual:** [Abrir diagramas en orden](README.md).  
**Web de tu cuenta AWS:** https://main.dzs66tep13b7u.amplifyapp.com  
**Región / stage:** `us-east-1` / `dev`.

> **Cómo usar este guion:** mantén abierto el `README.md` como presentación de diagramas. Sigue la numeración; cuando diga «En AWS» cambia a la pestaña correspondiente. No muestres contraseñas, tokens ni `password_hash`.

## 1. Mostrar DIAGRAMA 01 — Arquitectura general de Usuarios

**Abre:** [Diagrama 01 — Arquitectura general](README.md#01-arquitectura-general--ms0-usuarios).

**Di exactamente:**

> «Mi parte es el microservicio de Usuarios y el multi-tenancy. Trazora es un SaaS para varias empresas, pero todos comparten la misma aplicación. El frontend está en AWS Amplify; las solicitudes llegan a API Gateway y las procesan funciones AWS Lambda. Nuestro MS0 Usuarios tiene sus propias tablas DynamoDB: una de usuarios y otra de sesiones. Los demás microservicios usan la identidad validada por MS0 para proteger sus datos».

**Qué señalar con el cursor:** Amplify → API Gateway Usuarios → Lambda login → DynamoDB Usuarios/Sesiones → Lambda Authorizer → APIs de los demás servicios.

**En AWS, si te lo piden:** Lambda `api-usuarios-dev-login` y `api-usuarios-dev-validarToken`; tablas `dev-t_usuarios` y `dev-t_sesiones`.

## 2. Mostrar DIAGRAMA 02 — Flujo de inicio de sesión

**Abre:** [Diagrama 02 — Login](README.md#02-flujo-de-inicio-de-sesión).

**Di exactamente:**

> «Primero, el usuario selecciona su empresa e ingresa correo y contraseña. `POST /auth/login` llega a API Gateway y a la Lambda de login. Esta busca la cuenta dentro del tenant seleccionado, comprueba que esté activa y valida la contraseña contra su hash. Si es correcta, guarda una sesión en DynamoDB y entrega un token opaco con vencimiento».

**Dato técnico para defender:** el token **no contiene** directamente `tenant_id` ni `rol`. Es un identificador aleatorio; la sesión conserva `usuario_id`, `tenant_id` y expiración. La Lambda Authorizer resuelve esos datos al validar el token.

**En AWS:** API Gateway → `dev-api-usuarios` → Resources → `POST /auth/login`; después Lambda `api-usuarios-dev-login`.

## 3. Mostrar DIAGRAMA 03 — Autorización con Lambda Authorizer

**Abre:** [Diagrama 03 — Autorización](README.md#03-cómo-protege-el-authorizer-las-apis).

**Di exactamente:**

> «Luego, cada petición protegida incluye `Authorization: Bearer token`. API Gateway usa el Lambda Authorizer `validarToken`, que consulta la sesión, verifica su vencimiento y comprueba que el usuario sigue activo. Devuelve un contexto confiable con `usuario_id`, `tenant_id` y `rol`. La Lambda del endpoint comprueba además el permiso del rol y consulta solo los registros del tenant autorizado. Un token inválido se rechaza con HTTP 401».

**Aclara:** el Authorizer comprueba identidad y entrega contexto; **la Lambda de negocio** hace la validación del rol y del recurso. La interfaz no es una barrera de seguridad suficiente.

**En AWS:** Lambda → `api-usuarios-dev-validarToken`; API Gateway → una ruta protegida como `GET /usuarios`.

## 4. Mostrar DIAGRAMA 04 — Multi-tenancy en DynamoDB

**Abre:** [Diagrama 04 — Aislamiento](README.md#04-aislamiento-multi-tenant-en-dynamodb).

**Di exactamente:**

> «Usamos multi-tenancy lógico: las empresas comparten tablas, pero cada elemento pertenece a una partición lógica cuyo nombre comienza con `TENANT#`. Por ejemplo, `TENANT#CLIMALIMA`, `TENANT#FRIOSUR` y `TENANT#TECNINORTE`. Cuando una Lambda consulta, construye la clave usando el `tenant_id` obtenido del Authorizer, no un valor arbitrario enviado por el navegador. Así no puede consultar datos de otra empresa a través de la API normal».

**Qué señalar:** PK `TENANT#FRIOSUR`; SK `USUARIO#...` para el registro principal y `EMAIL#...` para encontrar la cuenta en login. **No confundas ambas entradas**.

**En AWS:** DynamoDB → Explore items → `dev-t_usuarios`. Muestra usuarios de ClimaLima y FríoSur sin abrir ni exhibir `password_hash`.

**Pregunta posible:** «¿Cada tenant tiene su propia tabla?» Respuesta: **No**; la separación es lógica, por clave y autorización. Cada **microservicio** sí tiene su tabla de negocio propia.

## 5. Mostrar DIAGRAMA 05 — API Gateway y endpoints de Usuarios

**Abre:** [Diagrama 05 — API Gateway y endpoints](README.md#05-api-gateway-y-operaciones-de-usuarios).

**Di exactamente:**

> «En API Gateway tenemos la API REST `dev-api-usuarios`. `POST /auth/login` inicia sesión; `POST /usuarios` crea un usuario; `GET /usuarios` lista solo usuarios del tenant; `GET /usuarios/me` devuelve el perfil autenticado; y `PATCH /usuarios/{usuario_id}` permite modificar rol y estado. Cada endpoint llega a su función Lambda y las operaciones administrativas exigen rol ADMIN».

**En AWS:** API Gateway → APIs → `dev-api-usuarios` → Resources. Señala los métodos y la integración Lambda correspondiente.

**Aclaración obligatoria sobre CRUD:** esta versión **no tiene** `DELETE /usuarios/{id}`. El botón **Desactivar** ejecuta un `PATCH` de `estado=INACTIVO`; es una **baja lógica, no eliminación física**. Si el docente exige demostración estricta de DELETE, no digas que desactivar equivale a borrar: requiere otro endpoint o que otro integrante muestre un DELETE real implementado.

## 6. Mostrar DIAGRAMA 06 — Demostración de dos tenants

**Abre:** [Diagrama 06 — Demo](README.md#06-demostración-dos-empresas-y-actualización-real).

**Di exactamente:**

> «Ahora lo mostramos funcionando. En una sesión entramos como administrador de ClimaLima y en otra como administrador de FríoSur. Ambas usan la misma URL de Amplify, pero la primera lista solo sus tres cuentas y la segunda sus cuentas. En FríoSur creamos un usuario desde la pantalla Usuarios; en DynamoDB verificamos que quedó con `PK=TENANT#FRIOSUR`. Después le cambiamos el rol de COORDINADOR a ADMIN desde la web y verificamos en DynamoDB el cambio del atributo `rol`».

**Orden de pantallas para la demo:**

1. Ventana de ClimaLima → pestaña **Usuarios** → mostrar cuentas ClimaLima.
2. Ventana incógnita de FríoSur → **Usuarios** → mostrar cuentas FríoSur.
3. Mostrar **Usuario Prueba Expo**, creado antes por `POST /usuarios`.
4. Mostrar el rol **ADMIN** de esa cuenta, modificado por `PATCH`.
5. En DynamoDB `dev-t_usuarios`, abrir el elemento **USUARIO#...** (no `EMAIL#...`); enseñar `tenant_id=FRIOSUR` y `rol=ADMIN`. **Ocultar el hash de contraseña**.
6. Opcional: demostrar **Desactivar**, aclarando que deja `estado=INACTIVO` y **no elimina físicamente** el usuario.

**Cuidado con la afirmación:** ver listas distintas es una demostración visual de separación, pero **no basta por sí sola para demostrar rechazo de acceso cruzado**. Si el docente pide una prueba de seguridad más fuerte, muestra una petición autenticada que intente acceder a un recurso ajeno y sea rechazada. No afirmes haber ejecutado esa prueba si no la hiciste.

## 7. Cierre de tu participación

> «En resumen: MS0 centraliza el login y las sesiones; API Gateway solicita al Authorizer validar identidad, tenant y rol; las Lambdas usan ese contexto para proteger los datos mediante claves `TENANT#...`. Por eso ClimaLima y FríoSur comparten la misma infraestructura sin consultar los registros de la otra empresa. Con esto termina Usuarios y Multi-tenancy y continúan mis compañeros con los demás módulos».

## 8. Preguntas rápidas del profesor

1. **¿Quién determina el tenant?** Durante el login se verifica la empresa indicada; en las demás peticiones se obtiene de la sesión validada por el Authorizer.
2. **¿El token lleva tenant y rol dentro?** No. Es opaco; el backend consulta la sesión y luego el usuario activo.
3. **¿Qué diferencia hay entre autenticación y autorización?** Autenticación confirma quién eres; autorización comprueba qué puedes hacer, con qué rol y sobre qué recursos.
4. **¿Dónde se guarda la contraseña?** En `dev-t_usuarios` como hash derivado mediante PBKDF2; nunca se muestra al frontend.
5. **¿Qué pasa si cambia el tenant_id en el body?** Las rutas protegidas ignoran ese valor como autoridad: usan el contexto validado.
6. **¿Cómo se protege el acceso cruzado?** Claves por tenant + comprobación de permisos y pertenencia en el backend.
7. **¿Dónde están los usuarios y sesiones?** `dev-t_usuarios` y `dev-t_sesiones`, exclusivas del MS0.
8. **¿Para qué se usa SNS/SQS aquí?** No forma parte del inicio de sesión. SNS/SQS se usa para los eventos de los microservicios de negocio; lo explican los responsables de esos módulos.
9. **¿Desactivar es eliminar?** No. Es una baja lógica por PATCH; el registro sigue en DynamoDB.
10. **¿Los tres tenants están probados?** Están configurados y hay datos de prueba; las dos sesiones demostradas visualmente fueron ClimaLima y FríoSur.

---

**Nota sobre las capturas:** enseña los datos de ejemplo, pero evita compartir `password_hash`, `Authorization`, tokens opacos, claves AWS o el archivo `.demo-credentials`. Deja la cuenta «Usuario Prueba Expo» nuevamente como `COORDINADOR` o `INACTIVO` después de la exposición para no mantener un administrador de prueba innecesario.

**Fuente del diseño:** implementación actual del repositorio del equipo `rvcuba-utec/cloud-project-v2`, módulo `backend/usuarios`, `backend/common/trazora_common/auth.py`, `docs/api.md` y capturas de la instalación propia en AWS.
