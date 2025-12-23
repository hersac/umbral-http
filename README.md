# Paquete HTTP para Umbral

Librería para gestionar solicitudes HTTP síncronas y asíncronas en Umbral.

## Instalación

```bash
ump add http
```

## Clases exportadas

- **Request**: Realizar solicitudes HTTP
- **Response**: Generar respuestas HTTP con códigos de estado
- **Router**: Gestionar rutas y middlewares
- **Next**: Control de flujo en middlewares

---

## Request

Clase para realizar solicitudes HTTP (GET, POST, PUT, DELETE, PATCH, OPTIONS).

### Métodos HTTP

```umbral
equip { Request } origin 'http';

!! GET
c: respuesta = awa Request.get("https://api.ejemplo.com/usuarios", nulo);

!! POST
c: datos = ["nombre"=>"Juan", "edad"=>25];
c: respuesta = awa Request.post("https://api.ejemplo.com/usuarios", datos, nulo);

!! PUT
c: datosActualizados = ["nombre"=>"Juan Carlos"];
c: respuesta = awa Request.put("https://api.ejemplo.com/usuarios/1", datosActualizados, nulo);

!! DELETE
c: respuesta = awa Request.delete("https://api.ejemplo.com/usuarios/1", nulo);

!! PATCH
c: cambios = ["edad"=>26];
c: respuesta = awa Request.patch("https://api.ejemplo.com/usuarios/1", cambios, nulo);

!! OPTIONS
c: respuesta = awa Request.options("https://api.ejemplo.com/usuarios", nulo);
```

### Opciones personalizadas

```umbral
c: opciones = [
    "headers"=>["Authorization"=>"Bearer token123", "Content-Type"=>"application/json"]
];

c: respuesta = awa Request.get("https://api.ejemplo.com/protegido", opciones);
```

### Propiedades de instancia

```umbral
c: req = n: Request("https://api.ejemplo.com/usuarios", "GET");

!! Acceder a propiedades
tprint(req.url);
tprint(req.metodo);
tprint(req.headers);
tprint(req.body);
tprint(req.params);
tprint(req.query);

!! Establecer headers
req.establecerHeader("Authorization", "Bearer token123");

!! Establecer parámetros
req.establecerParam("id", "123");

!! Establecer query
req.establecerQuery("page", "1");
```

---

## Response

Clase para generar respuestas HTTP con códigos de estado y mensajes.

### Métodos de respuesta

```umbral
equip { Response } origin 'http';

!! 200 OK
c: resp = Response.ok(["usuario"=>"Juan"], nulo);

!! 201 Creado
c: resp = Response.creado(["id"=>123], nulo);

!! 204 Sin contenido
c: resp = Response.sinContenido(nulo);

!! 400 Solicitud incorrecta
c: resp = Response.solicitudIncorrecta(["error"=>"Datos inválidos"], nulo);

!! 401 No autorizado
c: resp = Response.noAutorizado(nulo, nulo);

!! 403 Prohibido
c: resp = Response.prohibido(nulo, nulo);

!! 404 No encontrado
c: resp = Response.noEncontrado(nulo, nulo);

!! 500 Error interno
c: resp = Response.errorInterno(["error"=>"Error de base de datos"], nulo);

!! Código personalizado
c: resp = Response.status(418, ["mensaje"=>"Soy una tetera"], "I'm a teapot");
```

### Mensajes personalizados

```umbral
!! Mensaje por defecto
c: resp = Response.ok(["datos"=>"valor"], nulo);

!! Mensaje personalizado
c: resp = Response.ok(["datos"=>"valor"], "Operación exitosa");
```

### Convertir a JSON

```umbral
c: resp = Response.ok(["usuario"=>"Juan", "edad"=>25], nulo);
c: json = resp.aJson();

!! Resultado: ["codigo"=>200, "mensaje"=>"OK", "datos"=>["usuario"=>"Juan", "edad"=>25]]
```

---

## Router

Clase para gestionar rutas HTTP con soporte para middlewares.

### Definir rutas

```umbral
equip { Router, Request, Response } origin 'http';

c: router = n: Router();

!! GET
router.get("/usuarios", asy f: (req) {
    c: usuarios = ["usuario1", "usuario2"];
    r: (Response.ok(usuarios, nulo).aJson());
}, nulo);

!! POST
router.post("/usuarios", asy f: (req) {
    c: nuevoUsuario = req.body;
    r: (Response.creado(nuevoUsuario, nulo).aJson());
}, nulo);

!! PUT
router.put("/usuarios/:id", asy f: (req) {
    c: id = req.params["id"];
    c: datosActualizados = req.body;
    r: (Response.ok(datosActualizados, nulo).aJson());
}, nulo);

!! DELETE
router.delete("/usuarios/:id", asy f: (req) {
    r: (Response.sinContenido(nulo).aJson());
}, nulo);
```

### Middlewares

```umbral
!! Middleware de autenticación
c: autenticar = asy f: (req) {
    i: (req.headers["Authorization"] == nulo) {
        r: (falso);
    }
    r: (verdadero);
};

!! Ruta con middleware
router.get("/protegido", asy f: (req) {
    r: (Response.ok(["mensaje"=>"Acceso concedido"], nulo).aJson());
}, autenticar);
```

### Ejecutar rutas

```umbral
c: req = n: Request("/usuarios", "GET");
c: respuesta = awa router.ejecutar("/usuarios", "GET", req);
tprint(respuesta);
```

### Listar rutas

```umbral
c: rutas = router.listarRutas();
tprint(rutas);
```

---

## Next

Clase para controlar el flujo de middlewares.

### Uso básico

```umbral
equip { Next } origin 'http';

c: next = n: Next();

!! Continuar al siguiente middleware
next.continuar();

!! Verificar si fue ejecutado
i: (next.fueEjecutado()) {
    tprint("Next fue llamado");
}
```

### Manejo de errores

```umbral
c: next = n: Next();

!! Pasar un error
next.conError(["mensaje"=>"Error de validación"]);

!! Verificar si hay error
i: (next.tieneError()) {
    c: error = next.obtenerError();
    tprint("Error: &error");
}
```

### Reiniciar

```umbral
c: next = n: Next();
next.continuar();
next.reiniciar();

!! Ahora next.fueEjecutado() retorna falso
```

---

## Ejemplo completo

```umbral
equip { Request, Response, Router, Next } origin 'http';

!! Crear router
c: router = n: Router();

!! Middleware de logging
c: logger = asy f: (req) {
    tprint("Solicitud: &req.metodo &req.url");
    r: (verdadero);
};

!! Ruta GET con middleware
router.get("/api/usuarios", asy f: (req) {
    c: usuarios = [
        ["id"=>1, "nombre"=>"Juan"],
        ["id"=>2, "nombre"=>"María"]
    ];
    r: (Response.ok(usuarios, "Usuarios obtenidos exitosamente").aJson());
}, logger);

!! Ruta POST
router.post("/api/usuarios", asy f: (req) {
    c: nuevoUsuario = req.body;
    r: (Response.creado(nuevoUsuario, "Usuario creado").aJson());
}, nulo);

!! Ejecutar ruta
c: req = n: Request("/api/usuarios", "GET");
c: respuesta = awa router.ejecutar("/api/usuarios", "GET", req);
tprint(respuesta);

!! Hacer solicitud externa
c: datos = awa Request.get("https://jsonplaceholder.typicode.com/users", nulo);
tprint(datos);
```

