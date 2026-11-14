# Trazora FSM — Usuarios y Multi-tenancy

## 01. Arquitectura general · MS0 Usuarios

```mermaid
flowchart TB
  PERSONA["Administrador, coordinador o técnico"] --> WEB["Frontend React en AWS Amplify"]
  WEB -->|"POST /auth/login"| G0["API Gateway · dev-api-usuarios"]
  G0 --> LOGIN["Lambda · api-usuarios-dev-login"]
  LOGIN --> TU[("DynamoDB · dev-t_usuarios")]
  LOGIN --> TS[("DynamoDB · dev-t_sesiones")]
  LOGIN -->|"Entrega un token opaco"| WEB

  WEB -->|"Authorization: Bearer token"| G1["API Gateway · MS0 / MS1–MS6"]
  G1 -->|"Solicita autorización"| AUTH["Lambda Authorizer · validarToken"]
  AUTH -->|"Verifica token"| TS
  AUTH -->|"Valida usuario activo y rol"| TU
  AUTH -->|"Contexto validado: usuario_id, tenant_id, rol"| G1
  G1 -->|"Permiso concedido"| FN["Lambda del endpoint solicitado"]
  FN -->|"PK basada en tenant_id autorizado"| TABLAS[("DynamoDB exclusivo de cada microservicio")]
```

## 02. Flujo de inicio de sesión

```mermaid
sequenceDiagram
  autonumber
  actor U as Usuario
  participant W as React / Amplify
  participant G as API Gateway Usuarios
  participant L as Lambda login
  participant D as DynamoDB dev-t_usuarios
  participant S as DynamoDB dev-t_sesiones
  U->>W: Selecciona empresa y escribe credenciales
  W->>G: POST /auth/login
  G->>L: tenant_id, email, password
  L->>D: Buscar EMAIL y USUARIO dentro de TENANT#empresa
  D-->>L: Usuario, estado, rol y password_hash
  alt Credenciales correctas y usuario ACTIVO
    L->>L: Comprobar hash y generar token opaco aleatorio
    L->>S: Guardar hash(token), usuario_id, tenant_id y expiración
    S-->>L: Sesión creada
    L-->>G: Token y vencimiento
    G-->>W: HTTP 200
    W-->>U: Acceso autorizado
  else Credenciales incorrectas
    L-->>G: Credenciales inválidas
    G-->>W: HTTP 401
  end
  Note over W,S: El token es opaco; los datos del tenant y rol se resuelven en el backend.
```

## 03. Cómo protege el Authorizer las APIs

```mermaid
sequenceDiagram
  autonumber
  actor U as Usuario autenticado
  participant W as Frontend
  participant G as API Gateway
  participant A as Lambda validarToken
  participant S as dev-t_sesiones
  participant D as dev-t_usuarios
  participant F as Lambda del microservicio
  participant B as DynamoDB del servicio
  U->>W: Solicita recurso protegido
  W->>G: HTTPS + Authorization Bearer token
  G->>A: Autorizar petición
  A->>S: Buscar hash(token) y comprobar vencimiento
  S-->>A: Sesión con usuario_id y tenant_id
  A->>D: Consultar usuario activo y rol
  D-->>A: Identidad validada
  alt Token válido
    A-->>G: Contexto confiable tenant_id, usuario_id, rol
    G->>F: Ejecutar endpoint
    F->>F: Verificar rol y autorización del recurso
    F->>B: Query o GetItem con PK TENANT#tenant_id
    B-->>F: Registros solo de la empresa autorizada
    F-->>W: Respuesta de negocio
  else Token inválido o vencido
    A-->>G: Unauthorized
    G-->>W: HTTP 401
  end
```

## 04. Aislamiento multi-tenant en DynamoDB

```mermaid
flowchart TB
  ID["Lambda Authorizer<br/>entrega tenant_id validado"] --> REGLA["Lambda de negocio<br/>construye PK = TENANT#tenant_id"]

  REGLA --> C
  REGLA --> F
  REGLA --> T

  subgraph TABLA["Tabla compartida: dev-t_usuarios"]
    direction LR
    C["PK: TENANT#CLIMALIMA<br/>SK: USUARIO#... / EMAIL#..."]
    F["PK: TENANT#FRIOSUR<br/>SK: USUARIO#... / EMAIL#..."]
    T["PK: TENANT#TECNINORTE<br/>SK: USUARIO#... / EMAIL#..."]
  end

  C --> RC["Acceso solo a registros de CLIMALIMA"]
  F --> RF["Acceso solo a registros de FRIOSUR"]
  T --> RT["Acceso solo a registros de TECNINORTE"]

  SES["dev-t_sesiones<br/>token_hash -> usuario_id, tenant_id, expira_en"] -. "Permite validar la sesión" .-> ID
```

## 05. API Gateway y operaciones de Usuarios

```mermaid
flowchart TB
  API["REST API: dev-api-usuarios<br/>Stage: dev · us-east-1"]
  API --> LOGIN["POST /auth/login<br/>Lambda login · pública"]
  API --> POST["POST /usuarios<br/>Lambda crear · requiere ADMIN"]
  API --> GET["GET /usuarios<br/>Lambda listar · requiere ADMIN"]
  API --> ME["GET /usuarios/me<br/>Lambda yo · autenticado"]
  API --> PATCH["PATCH /usuarios/{usuario_id}<br/>Lambda actualizar · requiere ADMIN"]

  POST --> A["Lambda Authorizer valida sesión"]
  GET --> A
  ME --> A
  PATCH --> A
  A --> DATA[("dev-t_usuarios<br/>PK TENANT#empresa validada")]
  LOGIN --> LOGINDB[("dev-t_usuarios + dev-t_sesiones")]

  PATCH --> OFF["Cambiar rol o estado<br/>ACTIVO / INACTIVO"]
  OFF --> NO["Baja lógica ≠ DELETE<br/>No existe DELETE /usuarios en esta versión"]
```

## 06. Demostración: dos empresas y actualización real

```mermaid
flowchart TB
  subgraph CL["Ventana normal · ClimaLima"]
    CLLOGIN["Login como ADMIN de CLIMALIMA"] --> CLLIST["Usuarios: solo cuentas ClimaLima"]
  end

  subgraph FR["Ventana incógnita · FríoSur"]
    FRLOGIN["Login como ADMIN de FRIOSUR"] --> FRLIST["Usuarios: solo cuentas FríoSur"]
    FRLIST --> CREATE["Crear Usuario Prueba Expo<br/>POST /usuarios"]
    CREATE --> DDB["DynamoDB dev-t_usuarios<br/>PK TENANT#FRIOSUR"]
    DDB --> UPDATE["Cambiar rol COORDINADOR a ADMIN<br/>PATCH /usuarios/{usuario_id}"]
    UPDATE --> VER["Verificar rol = ADMIN en DynamoDB"]
    VER --> DES["Opcional: desactivar con PATCH<br/>estado = INACTIVO"]
  end

  CLLIST -. "Vistas separadas por contexto de tenant" .- FRLIST
```
