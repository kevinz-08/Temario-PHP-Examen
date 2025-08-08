# Apuntes de Estudio: Slim Framework, Arquitectura y Buenas Prácticas PHP

## Tabla de Contenidos

1. [Instalación con Composer para Slim](#instalacion-composer)
2. [Definición de rutas (GET, POST, PUT, etc.)](#definicion-rutas)
3. [Uso de Middlewares](#middlewares)
    - [Parseo de datos de entrada](#parseo-datos)
    - [Respuestas en JSON](#respuestas-json)
4. [Inyección de Dependencias (DI Container)](#di-container)
5. [Manejo de Errores Personalizado con Handlers](#handlers-errores)
6. [Autoloading con PSR-4](#psr4)
7. [Patrones de Diseño Recomendados](#patrones)
    - [DAO](#dao)
    - [Repository Pattern](#repository)
    - [DTOs](#dtos)
    - [Strategy Pattern](#strategy)
8. [Pruebas Manuales con Postman](#postman)
9. [Buenas Prácticas Generales](#buenas-practicas)

---

## 1. Instalación con Composer para Slim  <a name="instalacion-composer"></a>

**Composer** es el gestor de dependencias estándar para PHP. Permite instalar Slim y sus librerías asociadas fácilmente.

### Pasos de Instalación

1. **Instala Composer** (si no lo tienes):
   ```bash
   curl -sS https://getcomposer.org/installer | php
   mv composer.phar /usr/local/bin/composer
   ```
2. **Crea el proyecto y entra en el directorio:**
   ```bash
   mkdir mi-proyecto-slim
   cd mi-proyecto-slim
   ```
3. **Inicializa Composer en el proyecto (opcional):**
   ```bash
   composer init
   ```
4. **Instala Slim Framework y dependencias útiles:**
   ```bash
   composer require slim/slim:"^4.0" slim/psr7 php-di/php-di monolog/monolog
   ```
   - `slim/slim`: Framework principal
   - `slim/psr7`: Implementación PSR-7 para peticiones/respuestas HTTP
   - `php-di/php-di`: Contenedor de inyección de dependencias (DI)
   - `monolog/monolog`: Logging avanzado

5. **Estructura recomendada (ejemplo):**
   ```
   /public           # DocumentRoot (index.php)
   /src              # Código fuente (controllers, services, etc.)
   /vendor           # Librerías instaladas por Composer
   composer.json     # Configuración de dependencias
   ```

---

## 2. Definición de Rutas (GET, POST, PUT, etc.) <a name="definicion-rutas"></a>

Las rutas definen cómo responde la aplicación a distintas URLs y métodos HTTP.

### Ejemplo de definición de rutas en Slim 4

```php
use Slim\Factory\AppFactory;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

// GET
$app->get('/usuarios', function ($request, $response, $args) {
    $usuarios = [/* ... */];
    $response->getBody()->write(json_encode($usuarios));
    return $response->withHeader('Content-Type', 'application/json');
});

// POST
$app->post('/usuarios', function ($request, $response, $args) {
    $data = json_decode($request->getBody()->getContents(), true);
    // Procesar $data...
    $response->getBody()->write(json_encode(['mensaje' => 'Usuario creado']));
    return $response->withStatus(201)->withHeader('Content-Type', 'application/json');
});

// PUT
$app->put('/usuarios/{id}', function ($request, $response, $args) {
    $id = $args['id'];
    $data = json_decode($request->getBody()->getContents(), true);
    // Actualizar usuario...
    return $response->withHeader('Content-Type', 'application/json');
});

// DELETE
$app->delete('/usuarios/{id}', function ($request, $response, $args) {
    $id = $args['id'];
    // Eliminar usuario...
    return $response->withStatus(204);
});

$app->run();
```

**Diferencias principales entre métodos:**
- **GET:** Solicita datos.
- **POST:** Envía y crea datos.
- **PUT:** Modifica datos existentes.
- **DELETE:** Elimina datos.

---

## 3. Uso de Middlewares <a name="middlewares"></a>

Los middlewares permiten ejecutar lógica antes o después de procesar una petición.

### Ejemplo de Middleware de Logging

```php
$app->add(function ($request, $handler) {
    // Antes de la petición
    $response = $handler->handle($request);
    // Después de la petición
    return $response;
});
```

### Parseo de datos de entrada usando `file_get_contents('php://input')` <a name="parseo-datos"></a>

En Slim, puedes usar `$request->getBody()->getContents()` pero para proyectos puros PHP:

```php
$input = file_get_contents('php://input');
$data = json_decode($input, true);
```

### Formato de Respuestas en JSON <a name="respuestas-json"></a>

Siempre especifica el header `Content-Type: application/json`:

```php
$response->getBody()->write(json_encode($datos));
return $response->withHeader('Content-Type', 'application/json');
```

#### Buenas Prácticas:
- Usar `json_encode` con `JSON_UNESCAPED_UNICODE` para mantener caracteres especiales.
- Enviar siempre el header apropiado.

---

## 4. Inyección de Dependencias (DI Container) <a name="di-container"></a>

### ¿Qué es y por qué usarlo?

Es un patrón que permite desacoplar dependencias (servicios, repositorios, etc.) facilitando la gestión y pruebas del código.

**Ventajas:**
- Facilita el testing (mocking).
- Mantiene bajo acoplamiento.
- Centraliza la configuración de servicios y objetos.

### Implementación con PHP-DI en Slim

1. **Configura el contenedor:**
   ```php
   use DI\Container;
   use Slim\Factory\AppFactory;

   $container = new Container();
   AppFactory::setContainer($container);
   $app = AppFactory::create();
   ```

2. **Agrega dependencias:**
   ```php
   $container->set(MyService::class, function() {
       return new MyService();
   });
   ```

3. **Uso en controladores o middlewares:**
   ```php
   $app->get('/demo', function ($request, $response, $args) {
       $servicio = $this->get(MyService::class);
       // ...
   });
   ```

#### Ejemplo visual:

```
[Controlador] <-- MyService (inyectado desde el contenedor)
```

---

## 5. Manejo de Errores Personalizado con Handlers <a name="handlers-errores"></a>

En Slim puedes definir un error handler para personalizar las respuestas a excepciones.

### Ejemplo Práctico

```php
use Slim\Exception\HttpNotFoundException;

$errorMiddleware = $app->addErrorMiddleware(true, true, true);

$errorMiddleware->setErrorHandler(HttpNotFoundException::class, function ($request, Throwable $exception, bool $displayErrorDetails) use ($app) {
    $payload = ['error' => 'Recurso no encontrado'];
    $response = $app->getResponseFactory()->createResponse();
    $response->getBody()->write(json_encode($payload));
    return $response->withStatus(404)->withHeader('Content-Type', 'application/json');
});
```

**Recomendaciones:**
- Devuelve siempre mensajes claros.
- No muestres trazas de error en producción.

---

## 6. Autoloading con PSR-4 <a name="psr4"></a>

### ¿Qué es PSR-4?

Estándar para autoloading de clases basado en namespaces y estructura de carpetas.

### Configuración en `composer.json`

```json
"autoload": {
    "psr-4": {
        "App\\": "src/"
    }
}
```

- Namespace `App\` apunta a la carpeta `/src`.

**Después de modificar:**  
Ejecuta `composer dump-autoload` para regenerar el autoloader.

### Estructura de ejemplo

```
src/
  Controllers/
    UserController.php  // namespace App\Controllers;
  Models/
    User.php            // namespace App\Models;
```

---

## 7. Patrones de Diseño Recomendados <a name="patrones"></a>

### DAO (Data Access Object) <a name="dao"></a>

**Concepto:**  
Patrón que separa la lógica de acceso a datos de la lógica de negocio.

#### Ejemplo de modelado de entidad y DAO

```php
// src/Models/User.php
namespace App\Models;

class User {
    public $id;
    public $nombre;
    public $email;
    // ...
}

// src/DAO/UserDAO.php
class UserDAO {
    public function findById($id) { /* ... */ }
    public function save(User $user) { /* ... */ }
}
```

### Repository Pattern <a name="repository"></a>

- **Ventajas:**  
  - Separa la lógica de persistencia del resto de la app.
  - Facilita tests y cambios de tecnología de base de datos.
- **Estructura:**
  ```
  /Domain/Repository/UserRepository.php      // Interfaz
  /Infrastructure/Persistence/UserRepositoryImpl.php
  ```

#### Ejemplo básico de interfaz y repositorio

```php
// Dominio
interface UserRepository {
    public function findUserOfId(int $id): User;
    public function findAll(): array;
}

// Infraestructura
class EloquentUserRepository implements UserRepository {
    public function findUserOfId(int $id): User { /* ... */ }
    public function findAll(): array { /* ... */ }
}
```

### DTOs (Data Transfer Object) <a name="dtos"></a>

**Definición:**  
Objeto plano usado para transportar datos entre capas.

**Validación de datos en POST (ejemplo):**

```php
class UserDTO {
    public $name, $email, $date;

    public function __construct(array $data) {
        // Validar datos
        if (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Email inválido");
        }
        $this->date = (new \DateTime($data['date']))->format('Y-m-d');
        // ...
    }
}
```

**Manipulación de fechas:**
```php
$date = new \DateTime($data['fecha']);
$dto->fecha = $date->format('d/m/Y');
```

### Strategy Pattern (para cálculos) <a name="strategy"></a>

**Explicación:**  
Permite cambiar el algoritmo de cálculo en tiempo de ejecución.

**Diagrama:**
```
[Context] --> [StrategyA]
          --> [StrategyB]
```

**Ejemplo práctico:**

```php
interface CalculoEstrategia {
    public function calcular($a, $b);
}

class Suma implements CalculoEstrategia {
    public function calcular($a, $b) { return $a + $b; }
}

class Multiplicacion implements CalculoEstrategia {
    public function calcular($a, $b) { return $a * $b; }
}

class Contexto {
    private $estrategia;
    public function __construct(CalculoEstrategia $e) { $this->estrategia = $e; }
    public function ejecutar($a, $b) { return $this->estrategia->calcular($a, $b); }
}

// Uso:
$ctx = new Contexto(new Suma());
echo $ctx->ejecutar(2, 3); // 5
```

---

## 8. Pruebas manuales con Postman <a name="postman"></a>

### Estructura de pruebas

1. **Colección de pruebas:**  
   Agrupa endpoints por recurso (usuarios, productos, etc.)

2. **Documenta cada prueba:**
    - URL
    - Método (GET, POST, etc.)
    - Headers necesarios (Content-Type, Authorization)
    - Body de ejemplo (JSON)
    - Respuesta esperada (status, estructura)

#### Ejemplo de petición POST documentada

- **URL:** `POST /usuarios`
- **Headers:** `Content-Type: application/json`
- **Body:**
  ```json
  {
    "name": "Alice",
    "email": "alice@example.com"
  }
  ```
- **Respuesta esperada:**
  ```json
  {
    "mensaje": "Usuario creado",
    "id": 123
  }
  ```

**Consejo:**  
Agrega aserciones en Postman para validar status y estructura de respuesta automáticamente.

---

## 9. Buenas Prácticas Generales <a name="buenas-practicas"></a>

- Usa namespaces y autoload PSR-4.
- Mantén tus controladores delgados; la lógica de negocio debe ir en servicios.
- Valida y sanitiza toda entrada de usuario.
- Usa DTOs para transportar datos entre capas.
- Maneja los errores y excepciones de forma centralizada.
- Documenta tu API (OpenAPI/Swagger recomendado).
- No expongas información sensible en los mensajes de error.
- Mantén tu código modular y cubierto por pruebas.

---