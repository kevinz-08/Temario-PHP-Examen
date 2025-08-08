# Guía de Estudio - Slim Framework

## 1. Instalación con Composer para Slim

### Comandos de instalación paso a paso

```bash
# 1. Crear directorio del proyecto
mkdir mi-proyecto-slim
cd mi-proyecto-slim

# 2. Inicializar Composer
composer init

# 3. Instalar Slim Framework
composer require slim/slim:"4.*"

# 4. Instalar PSR-7 implementation (para HTTP messages)
composer require slim/psr7

# 5. Instalar dependencias adicionales recomendadas
composer require php-di/php-di         # Para inyección de dependencias
composer require monolog/monolog       # Para logging
```

### Explicación de cada comando

- **`composer init`**: Crea el archivo `composer.json` interactivamente
- **`slim/slim:"4.*"`**: Instala la última versión estable de Slim 4
- **`slim/psr7`**: Implementación de PSR-7 para manejo de HTTP requests/responses
- **`php-di/php-di`**: Container de inyección de dependencias robusto
- **`monolog/monolog`**: Biblioteca de logging para el manejo de errores

### Estructura básica del archivo `index.php`

```php
<?php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

require __DIR__ . '/vendor/autoload.php';

$app = AppFactory::create();

$app->get('/', function (Request $request, Response $response) {
    $response->getBody()->write("¡Hola, Slim!");
    return $response;
});

$app->run();
```

---

## 2. Definición de Rutas

### Métodos HTTP principales

#### GET - Obtener datos
```php
$app->get('/users', function (Request $request, Response $response) {
    $data = ['users' => ['Juan', 'María', 'Pedro']];
    $response->getBody()->write(json_encode($data));
    return $response->withHeader('Content-Type', 'application/json');
});

// Ruta con parámetros
$app->get('/users/{id}', function (Request $request, Response $response, $args) {
    $userId = $args['id'];
    $data = ['user' => "Usuario ID: $userId"];
    $response->getBody()->write(json_encode($data));
    return $response->withHeader('Content-Type', 'application/json');
});
```

#### POST - Crear recursos
```php
$app->post('/users', function (Request $request, Response $response) {
    $body = $request->getBody()->getContents();
    $data = json_decode($body, true);
    
    // Procesar creación del usuario
    $newUser = [
        'id' => rand(1, 1000),
        'name' => $data['name'],
        'email' => $data['email']
    ];
    
    $response->getBody()->write(json_encode($newUser));
    return $response
        ->withHeader('Content-Type', 'application/json')
        ->withStatus(201); // Created
});
```

#### PUT - Actualizar completamente
```php
$app->put('/users/{id}', function (Request $request, Response $response, $args) {
    $userId = $args['id'];
    $body = $request->getBody()->getContents();
    $data = json_decode($body, true);
    
    // Actualización completa del usuario
    $updatedUser = [
        'id' => $userId,
        'name' => $data['name'],
        'email' => $data['email'],
        'updated_at' => date('Y-m-d H:i:s')
    ];
    
    $response->getBody()->write(json_encode($updatedUser));
    return $response->withHeader('Content-Type', 'application/json');
});
```

#### PATCH - Actualizar parcialmente
```php
$app->patch('/users/{id}', function (Request $request, Response $response, $args) {
    $userId = $args['id'];
    $body = $request->getBody()->getContents();
    $data = json_decode($body, true);
    
    // Solo actualizar campos presentes
    $userUpdates = array_filter($data, function($value) {
        return $value !== null && $value !== '';
    });
    
    $response->getBody()->write(json_encode([
        'message' => 'Usuario actualizado parcialmente',
        'updated_fields' => array_keys($userUpdates)
    ]));
    
    return $response->withHeader('Content-Type', 'application/json');
});
```

#### DELETE - Eliminar recursos
```php
$app->delete('/users/{id}', function (Request $request, Response $response, $args) {
    $userId = $args['id'];
    
    // Lógica de eliminación
    $result = [
        'message' => "Usuario $userId eliminado correctamente",
        'deleted_at' => date('Y-m-d H:i:s')
    ];
    
    $response->getBody()->write(json_encode($result));
    return $response
        ->withHeader('Content-Type', 'application/json')
        ->withStatus(200);
});
```

### Diferencias clave entre métodos

| Método | Propósito | Idempotente | Cacheable | Body requerido |
|--------|-----------|-------------|-----------|---------------|
| GET    | Obtener   | ✓          | ✓         | ✗            |
| POST   | Crear     | ✗          | ✗         | ✓            |
| PUT    | Actualizar completo | ✓ | ✗    | ✓            |
| PATCH  | Actualizar parcial | ✗  | ✗     | ✓            |
| DELETE | Eliminar  | ✓          | ✗         | ✗            |

---

## 3. Uso de Middlewares

### Middleware para parseo de datos de entrada

```php
// Middleware personalizado para parsear JSON
$jsonMiddleware = function (Request $request, $handler) {
    $contentType = $request->getHeaderLine('Content-Type');
    
    if (strpos($contentType, 'application/json') !== false) {
        $body = file_get_contents('php://input');
        $data = json_decode($body, true);
        
        if (json_last_error() === JSON_ERROR_NONE) {
            $request = $request->withParsedBody($data);
        }
    }
    
    return $handler->handle($request);
};

// Aplicar middleware globalmente
$app->add($jsonMiddleware);
```

### Middleware para formato de respuestas JSON

```php
$jsonResponseMiddleware = function (Request $request, $handler) {
    $response = $handler->handle($request);
    
    // Asegurar que todas las respuestas sean JSON
    return $response->withHeader('Content-Type', 'application/json');
};

$app->add($jsonResponseMiddleware);
```

### Middleware de validación de entrada

```php
$validationMiddleware = function (Request $request, $handler) {
    $method = $request->getMethod();
    
    if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
        $data = $request->getParsedBody();
        
        if (empty($data)) {
            $response = new \Slim\Psr7\Response();
            $response->getBody()->write(json_encode([
                'error' => 'Datos requeridos',
                'message' => 'El cuerpo de la petición no puede estar vacío'
            ]));
            return $response
                ->withHeader('Content-Type', 'application/json')
                ->withStatus(400);
        }
    }
    
    return $handler->handle($request);
};
```

### Aplicar middleware a rutas específicas

```php
$app->group('/api', function ($group) {
    $group->get('/users', function (Request $request, Response $response) {
        // Lógica de la ruta
    });
})->add($jsonResponseMiddleware);
```

---

## 4. Inyección de Dependencias (DI Container)

### ¿Qué es la Inyección de Dependencias?

La **Inyección de Dependencias** es un patrón que permite que los objetos reciban sus dependencias desde el exterior, en lugar de crearlas internamente. Esto mejora la testabilidad, flexibilidad y mantenibilidad del código.

### ¿Por qué usarlo?

- **Desacoplamiento**: Las clases no dependen de implementaciones concretas
- **Testabilidad**: Facilita el uso de mocks en pruebas
- **Flexibilidad**: Permite cambiar implementaciones sin modificar código
- **Principio de Inversión de Dependencias**: Depende de abstracciones, no de concreciones

### Configuración del DI Container

```php
<?php
use DI\Container;
use DI\ContainerBuilder;
use Slim\Factory\AppFactory;

require __DIR__ . '/vendor/autoload.php';

// Crear container
$containerBuilder = new ContainerBuilder();

// Configurar dependencias
$containerBuilder->addDefinitions([
    // Configuración de base de datos
    PDO::class => function () {
        $dsn = 'mysql:host=localhost;dbname=test;charset=utf8';
        return new PDO($dsn, 'user', 'password');
    },
    
    // Repository
    UserRepositoryInterface::class => DI\create(UserRepository::class),
    
    // Services
    UserService::class => DI\autowire()
        ->constructorParameter('repository', DI\get(UserRepositoryInterface::class)),
]);

$container = $containerBuilder->build();

// Configurar Slim con el container
AppFactory::setContainer($container);
$app = AppFactory::create();
```

### Ejemplo de uso en controladores

```php
interface UserRepositoryInterface 
{
    public function findAll(): array;
    public function findById(int $id): ?User;
    public function save(User $user): void;
}

class UserRepository implements UserRepositoryInterface 
{
    private PDO $pdo;
    
    public function __construct(PDO $pdo) {
        $this->pdo = $pdo;
    }
    
    public function findAll(): array {
        $stmt = $this->pdo->query('SELECT * FROM users');
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Más métodos...
}

class UserService 
{
    private UserRepositoryInterface $repository;
    
    public function __construct(UserRepositoryInterface $repository) {
        $this->repository = $repository;
    }
    
    public function getAllUsers(): array {
        return $this->repository->findAll();
    }
}

// En las rutas
$app->get('/users', function (Request $request, Response $response) use ($container) {
    $userService = $container->get(UserService::class);
    $users = $userService->getAllUsers();
    
    $response->getBody()->write(json_encode($users));
    return $response->withHeader('Content-Type', 'application/json');
});
```

---

## 5. Manejo de Errores Personalizado con Handlers

### Error Handler básico

```php
use Slim\Exception\HttpNotFoundException;
use Slim\Exception\HttpMethodNotAllowedException;

$errorHandler = function (
    ServerRequestInterface $request,
    Throwable $exception,
    bool $displayErrorDetails
) {
    $statusCode = 500;
    $errorMessage = 'Error interno del servidor';
    
    // Manejar diferentes tipos de excepciones
    if ($exception instanceof HttpNotFoundException) {
        $statusCode = 404;
        $errorMessage = 'Recurso no encontrado';
    } elseif ($exception instanceof HttpMethodNotAllowedException) {
        $statusCode = 405;
        $errorMessage = 'Método no permitido';
    }
    
    $errorResponse = [
        'error' => true,
        'message' => $errorMessage,
        'code' => $statusCode,
        'timestamp' => date('Y-m-d H:i:s')
    ];
    
    if ($displayErrorDetails) {
        $errorResponse['details'] = [
            'file' => $exception->getFile(),
            'line' => $exception->getLine(),
            'trace' => $exception->getTraceAsString()
        ];
    }
    
    $response = new \Slim\Psr7\Response();
    $response->getBody()->write(json_encode($errorResponse));
    
    return $response
        ->withHeader('Content-Type', 'application/json')
        ->withStatus($statusCode);
};

// Agregar el error handler
$app->addErrorMiddleware(true, true, true)
    ->setDefaultErrorHandler($errorHandler);
```

### Excepciones personalizadas

```php
class ValidationException extends Exception 
{
    private array $errors;
    
    public function __construct(array $errors, string $message = 'Errores de validación') {
        parent::__construct($message);
        $this->errors = $errors;
    }
    
    public function getErrors(): array {
        return $this->errors;
    }
}

class BusinessLogicException extends Exception 
{
    public function __construct(string $message, int $code = 400) {
        parent::__construct($message, $code);
    }
}

// Handler específico para excepciones de validación
$validationErrorHandler = function (
    ServerRequestInterface $request,
    ValidationException $exception
) {
    $response = new \Slim\Psr7\Response();
    $response->getBody()->write(json_encode([
        'error' => true,
        'message' => $exception->getMessage(),
        'validation_errors' => $exception->getErrors(),
        'code' => 422
    ]));
    
    return $response
        ->withHeader('Content-Type', 'application/json')
        ->withStatus(422);
};
```

### Ejemplo práctico de uso

```php
$app->post('/users', function (Request $request, Response $response) {
    try {
        $data = $request->getParsedBody();
        
        // Validación
        $errors = [];
        if (empty($data['email'])) {
            $errors['email'] = 'Email es requerido';
        }
        if (empty($data['name'])) {
            $errors['name'] = 'Nombre es requerido';
        }
        
        if (!empty($errors)) {
            throw new ValidationException($errors);
        }
        
        // Lógica de negocio
        if (userExists($data['email'])) {
            throw new BusinessLogicException('El email ya está en uso');
        }
        
        // Crear usuario
        $user = createUser($data);
        
        $response->getBody()->write(json_encode($user));
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withStatus(201);
            
    } catch (ValidationException $e) {
        return $validationErrorHandler($request, $e);
    } catch (BusinessLogicException $e) {
        $response->getBody()->write(json_encode([
            'error' => true,
            'message' => $e->getMessage()
        ]));
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withStatus($e->getCode());
    }
});
```

---

## 6. Autoloading con PSR-4

### Configuración en `composer.json`

```json
{
    "name": "mi-usuario/mi-proyecto",
    "description": "Proyecto con Slim Framework",
    "require": {
        "slim/slim": "4.*",
        "slim/psr7": "^1.0",
        "php-di/php-di": "^6.0"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Controllers\\": "src/Controllers/",
            "App\\Models\\": "src/Models/",
            "App\\Services\\": "src/Services/",
            "App\\Repositories\\": "src/Repositories/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

### Estructura de directorios

```
proyecto/
├── src/
│   ├── Controllers/
│   │   ├── UserController.php
│   │   └── ProductController.php
│   ├── Models/
│   │   ├── User.php
│   │   └── Product.php
│   ├── Services/
│   │   ├── UserService.php
│   │   └── EmailService.php
│   ├── Repositories/
│   │   ├── UserRepository.php
│   │   └── ProductRepository.php
│   └── DTOs/
│       ├── UserDTO.php
│       └── ProductDTO.php
├── vendor/
├── composer.json
└── index.php
```

### Ejemplos de clases con namespaces

```php
<?php
// src/Models/User.php
namespace App\Models;

class User 
{
    private int $id;
    private string $name;
    private string $email;
    
    public function __construct(int $id, string $name, string $email) {
        $this->id = $id;
        $this->name = $name;
        $this->email = $email;
    }
    
    // Getters y setters
    public function getId(): int { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email; }
}
```

```php
<?php
// src/Controllers/UserController.php
namespace App\Controllers;

use App\Services\UserService;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

class UserController 
{
    private UserService $userService;
    
    public function __construct(UserService $userService) {
        $this->userService = $userService;
    }
    
    public function index(Request $request, Response $response): Response {
        $users = $this->userService->getAllUsers();
        $response->getBody()->write(json_encode($users));
        return $response->withHeader('Content-Type', 'application/json');
    }
}
```

### Regenerar el autoloader

```bash
# Después de modificar composer.json
composer dump-autoload
```

---

## 7. Patrones de Diseño Recomendados

### DAO (Data Access Object)

El patrón **DAO** encapsula el acceso a los datos y proporciona una interfaz uniforme para interactuar con diferentes fuentes de datos.

```php
<?php
// src/Models/User.php
namespace App\Models;

class User 
{
    private ?int $id = null;
    private string $name;
    private string $email;
    private \DateTime $createdAt;
    
    public function __construct(string $name, string $email) {
        $this->name = $name;
        $this->email = $email;
        $this->createdAt = new \DateTime();
    }
    
    // Getters y setters
    public function getId(): ?int { return $this->id; }
    public function setId(int $id): void { $this->id = $id; }
    public function getName(): string { return $this->name; }
    public function setName(string $name): void { $this->name = $name; }
    public function getEmail(): string { return $this->email; }
    public function setEmail(string $email): void { $this->email = $email; }
    public function getCreatedAt(): \DateTime { return $this->createdAt; }
    
    // Método para convertir a array
    public function toArray(): array {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->createdAt->format('Y-m-d H:i:s')
        ];
    }
    
    // Método estático para crear desde array
    public static function fromArray(array $data): self {
        $user = new self($data['name'], $data['email']);
        if (isset($data['id'])) {
            $user->setId($data['id']);
        }
        if (isset($data['created_at'])) {
            $user->createdAt = new \DateTime($data['created_at']);
        }
        return $user;
    }
}
```

### Repository Pattern

El patrón **Repository** abstrae la capa de acceso a datos y proporciona una interfaz más orientada a objetos para las consultas.

#### Ventajas del Repository Pattern:
- **Separación de responsabilidades**: Lógica de negocio separada del acceso a datos
- **Testabilidad**: Fácil mockeo para pruebas unitarias
- **Flexibilidad**: Cambio de fuente de datos sin afectar la lógica de negocio
- **Reutilización**: Consultas complejas centralizadas

```php
<?php
// src/Repositories/UserRepositoryInterface.php
namespace App\Repositories;

use App\Models\User;

interface UserRepositoryInterface 
{
    public function findAll(): array;
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function save(User $user): User;
    public function delete(int $id): bool;
    public function findWithPagination(int $page, int $limit): array;
}
```

```php
<?php
// src/Repositories/UserRepository.php
namespace App\Repositories;

use App\Models\User;
use PDO;

class UserRepository implements UserRepositoryInterface 
{
    private PDO $pdo;
    
    public function __construct(PDO $pdo) {
        $this->pdo = $pdo;
    }
    
    public function findAll(): array {
        $stmt = $this->pdo->query('SELECT * FROM users ORDER BY created_at DESC');
        $data = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        return array_map(function($row) {
            return User::fromArray($row);
        }, $data);
    }
    
    public function findById(int $id): ?User {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = :id');
        $stmt->execute(['id' => $id]);
        $data = $stmt->fetch(PDO::FETCH_ASSOC);
        
        return $data ? User::fromArray($data) : null;
    }
    
    public function findByEmail(string $email): ?User {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE email = :email');
        $stmt->execute(['email' => $email]);
        $data = $stmt->fetch(PDO::FETCH_ASSOC);
        
        return $data ? User::fromArray($data) : null;
    }
    
    public function save(User $user): User {
        if ($user->getId() === null) {
            // INSERT
            $stmt = $this->pdo->prepare('INSERT INTO users (name, email, created_at) VALUES (:name, :email, :created_at)');
            $stmt->execute([
                'name' => $user->getName(),
                'email' => $user->getEmail(),
                'created_at' => $user->getCreatedAt()->format('Y-m-d H:i:s')
            ]);
            $user->setId((int)$this->pdo->lastInsertId());
        } else {
            // UPDATE
            $stmt = $this->pdo->prepare('UPDATE users SET name = :name, email = :email WHERE id = :id');
            $stmt->execute([
                'name' => $user->getName(),
                'email' => $user->getEmail(),
                'id' => $user->getId()
            ]);
        }
        
        return $user;
    }
    
    public function delete(int $id): bool {
        $stmt = $this->pdo->prepare('DELETE FROM users WHERE id = :id');
        return $stmt->execute(['id' => $id]);
    }
    
    public function findWithPagination(int $page, int $limit): array {
        $offset = ($page - 1) * $limit;
        $stmt = $this->pdo->prepare('SELECT * FROM users LIMIT :limit OFFSET :offset');
        $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
        $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        
        $data = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        return array_map(function($row) {
            return User::fromArray($row);
        }, $data);
    }
}
```

### DTOs (Data Transfer Objects)

Los **DTOs** son objetos que transportan datos entre diferentes capas de la aplicación, especialmente útiles para validación y formateo.

```php
<?php
// src/DTOs/UserDTO.php
namespace App\DTOs;

use DateTime;

class UserDTO 
{
    private string $name;
    private string $email;
    private ?string $phone = null;
    private ?DateTime $birthDate = null;
    
    public function __construct(array $data) {
        $this->validate($data);
        $this->hydrate($data);
    }
    
    private function validate(array $data): void {
        $errors = [];
        
        // Validación de nombre
        if (empty($data['name']) || !is_string($data['name'])) {
            $errors['name'] = 'El nombre es requerido y debe ser una cadena';
        } elseif (strlen($data['name']) < 2) {
            $errors['name'] = 'El nombre debe tener al menos 2 caracteres';
        }
        
        // Validación de email
        if (empty($data['email']) || !is_string($data['email'])) {
            $errors['email'] = 'El email es requerido';
        } elseif (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            $errors['email'] = 'El email debe tener un formato válido';
        }
        
        // Validación de teléfono (opcional)
        if (isset($data['phone']) && !empty($data['phone'])) {
            if (!preg_match('/^\+?[1-9]\d{1,14}$/', $data['phone'])) {
                $errors['phone'] = 'El formato del teléfono no es válido';
            }
        }
        
        // Validación de fecha de nacimiento
        if (isset($data['birth_date']) && !empty($data['birth_date'])) {
            try {
                $date = new DateTime($data['birth_date']);
                $now = new DateTime();
                if ($date > $now) {
                    $errors['birth_date'] = 'La fecha de nacimiento no puede ser futura';
                }
            } catch (\Exception $e) {
                $errors['birth_date'] = 'Formato de fecha inválido (use YYYY-MM-DD)';
            }
        }
        
        if (!empty($errors)) {
            throw new \App\Exceptions\ValidationException($errors);
        }
    }
    
    private function hydrate(array $data): void {
        $this->name = trim($data['name']);
        $this->email = strtolower(trim($data['email']));
        
        if (isset($data['phone']) && !empty($data['phone'])) {
            $this->phone = $data['phone'];
        }
        
        if (isset($data['birth_date']) && !empty($data['birth_date'])) {
            $this->birthDate = new DateTime($data['birth_date']);
        }
    }
    
    // Getters
    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email; }
    public function getPhone(): ?string { return $this->phone; }
    public function getBirthDate(): ?DateTime { return $this->birthDate; }
    
    // Formateo de datos
    public function getFormattedBirthDate(): ?string {
        return $this->birthDate ? $this->birthDate->format('d/m/Y') : null;
    }
    
    public function getAge(): ?int {
        if (!$this->birthDate) {
            return null;
        }
        
        $now = new DateTime();
        return $this->birthDate->diff($now)->y;
    }
    
    // Conversión a array para persistencia
    public function toArray(): array {
        return [
            'name' => $this->name,
            'email' => $this->email,
            'phone' => $this->phone,
            'birth_date' => $this->birthDate ? $this->birthDate->format('Y-m-d') : null
        ];
    }
    
    // Método estático para validación de POST
    public static function fromPostRequest(array $postData): self {
        // Sanitización adicional
        $sanitized = [
            'name' => isset($postData['name']) ? strip_tags($postData['name']) : '',
            'email' => isset($postData['email']) ? filter_var($postData['email'], FILTER_SANITIZE_EMAIL) : '',
            'phone' => isset($postData['phone']) ? preg_replace('/[^+0-9]/', '', $postData['phone']) : null,
            'birth_date' => isset($postData['birth_date']) ? $postData['birth_date'] : null
        ];
        
        return new self($sanitized);
    }
}
```

### Strategy Pattern para Cálculos

El patrón **Strategy** permite definir una familia de algoritmos y hacer que sean intercambiables en tiempo de ejecución.

```php
<?php
// src/Strategies/PriceCalculationInterface.php
namespace App\Strategies;

use App\Models\Product;

interface PriceCalculationInterface 
{
    public function calculate(Product $product, int $quantity): float {
        $regularPrice = $product->getPrice() * $quantity;
        
        if ($quantity >= $this->minimumQuantity) {
            return $regularPrice * (1 - $this->discountPercentage / 100);
        }
        
        return $regularPrice;
    }
    
    public function getDescription(): string {
        return "Descuento del {$this->discountPercentage}% para compras de {$this->minimumQuantity} o más unidades";
    }
}

// src/Strategies/SeasonalDiscountStrategy.php
class SeasonalDiscountStrategy implements PriceCalculationInterface 
{
    private float $discountPercentage;
    private DateTime $startDate;
    private DateTime $endDate;
    
    public function __construct(float $discountPercentage, DateTime $startDate, DateTime $endDate) {
        $this->discountPercentage = $discountPercentage;
        $this->startDate = $startDate;
        $this->endDate = $endDate;
    }
    
    public function calculate(Product $product, int $quantity): float {
        $regularPrice = $product->getPrice() * $quantity;
        $now = new DateTime();
        
        if ($now >= $this->startDate && $now <= $this->endDate) {
            return $regularPrice * (1 - $this->discountPercentage / 100);
        }
        
        return $regularPrice;
    }
    
    public function getDescription(): string {
        return "Descuento estacional del {$this->discountPercentage}% válido del {$this->startDate->format('d/m/Y')} al {$this->endDate->format('d/m/Y')}";
    }
}
```

#### Contexto para usar las estrategias

```php
<?php
// src/Services/PriceCalculatorService.php
namespace App\Services;

use App\Strategies\PriceCalculationInterface;
use App\Models\Product;

class PriceCalculatorService 
{
    private PriceCalculationInterface $strategy;
    
    public function __construct(PriceCalculationInterface $strategy) {
        $this->strategy = $strategy;
    }
    
    public function setStrategy(PriceCalculationInterface $strategy): void {
        $this->strategy = $strategy;
    }
    
    public function calculatePrice(Product $product, int $quantity): array {
        $finalPrice = $this->strategy->calculate($product, $quantity);
        $regularPrice = $product->getPrice() * $quantity;
        $savings = $regularPrice - $finalPrice;
        
        return [
            'regular_price' => $regularPrice,
            'final_price' => $finalPrice,
            'savings' => $savings,
            'discount_applied' => $this->strategy->getDescription(),
            'quantity' => $quantity
        ];
    }
}
```

#### Ejemplo de uso en controlador

```php
$app->post('/calculate-price', function (Request $request, Response $response) use ($container) {
    $data = $request->getParsedBody();
    $productId = $data['product_id'];
    $quantity = $data['quantity'];
    $discountType = $data['discount_type'] ?? 'regular';
    
    $productRepository = $container->get(ProductRepositoryInterface::class);
    $product = $productRepository->findById($productId);
    
    // Seleccionar estrategia según el tipo
    $strategy = match($discountType) {
        'bulk' => new BulkDiscountStrategy(10, 5), // 10% descuento para 5+ items
        'seasonal' => new SeasonalDiscountStrategy(15, new DateTime('2024-12-01'), new DateTime('2024-12-31')),
        default => new RegularPriceStrategy()
    };
    
    $calculator = new PriceCalculatorService($strategy);
    $result = $calculator->calculatePrice($product, $quantity);
    
    $response->getBody()->write(json_encode($result));
    return $response->withHeader('Content-Type', 'application/json');
});
```

---

## 8. Pruebas Manuales con Postman

### Estructura de pruebas recomendada

#### Organización de colecciones
```
API Tests/
├── Authentication/
│   ├── Login
│   ├── Register
│   └── Refresh Token
├── Users/
│   ├── GET All Users
│   ├── GET User by ID
│   ├── POST Create User
│   ├── PUT Update User
│   ├── PATCH Partial Update
│   └── DELETE User
└── Products/
    ├── GET All Products
    ├── POST Create Product
    └── Calculate Price
```

#### Configuración de Environment Variables

**Variables de entorno para testing:**
```json
{
  "base_url": "http://localhost:8080",
  "api_version": "v1",
  "auth_token": "",
  "user_id": "",
  "product_id": ""
}
```

#### Ejemplos de requests documentados

**1. GET All Users**
```
GET {{base_url}}/api/{{api_version}}/users
Headers:
  Content-Type: application/json
  Authorization: Bearer {{auth_token}}

Tests:
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Response is JSON", function () {
    pm.response.to.be.json;
});

pm.test("Response has users array", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property('users');
    pm.expect(jsonData.users).to.be.an('array');
});
```

**2. POST Create User**
```
POST {{base_url}}/api/{{api_version}}/users
Headers:
  Content-Type: application/json

Body (raw JSON):
{
  "name": "Juan Pérez",
  "email": "juan.perez@example.com",
  "phone": "+57300123456",
  "birth_date": "1990-05-15"
}

Pre-request Script:
// Generar email único para cada prueba
const timestamp = Date.now();
const uniqueEmail = `test.user.${timestamp}@example.com`;
pm.environment.set("unique_email", uniqueEmail);

Tests:
pm.test("User created successfully", function () {
    pm.response.to.have.status(201);
    
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property('id');
    pm.expect(jsonData.email).to.equal(pm.environment.get("unique_email"));
    
    // Guardar ID para pruebas posteriores
    pm.environment.set("user_id", jsonData.id);
});

pm.test("Response time is less than 1000ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(1000);
});
```

**3. PUT Update User**
```
PUT {{base_url}}/api/{{api_version}}/users/{{user_id}}
Headers:
  Content-Type: application/json

Body (raw JSON):
{
  "name": "Juan Pérez Actualizado",
  "email": "juan.actualizado@example.com",
  "phone": "+57300654321",
  "birth_date": "1990-05-15"
}

Tests:
pm.test("User updated successfully", function () {
    pm.response.to.have.status(200);
    
    const jsonData = pm.response.json();
    pm.expect(jsonData.name).to.equal("Juan Pérez Actualizado");
    pm.expect(jsonData.email).to.equal("juan.actualizado@example.com");
});
```

**4. Validación de errores**
```
POST {{base_url}}/api/{{api_version}}/users
Headers:
  Content-Type: application/json

Body (raw JSON):
{
  "name": "",
  "email": "invalid-email"
}

Tests:
pm.test("Validation errors handled correctly", function () {
    pm.response.to.have.status(422);
    
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property('error');
    pm.expect(jsonData.error).to.be.true;
    pm.expect(jsonData).to.have.property('validation_errors');
    
    // Verificar errores específicos
    pm.expect(jsonData.validation_errors).to.have.property('name');
    pm.expect(jsonData.validation_errors).to.have.property('email');
});
```

### Scripts útiles para automatización

#### Cleanup Script (para ejecutar después de pruebas)
```javascript
// Pre-request o Test script
pm.sendRequest({
    url: pm.environment.get("base_url") + "/api/v1/users/" + pm.environment.get("user_id"),
    method: 'DELETE',
    header: {
        'Content-Type': 'application/json'
    }
}, function (err, response) {
    if (!err) {
        console.log("Test user deleted successfully");
        pm.environment.unset("user_id");
    }
});
```

#### Test Data Generator
```javascript
// Pre-request Script
const testData = {
    randomName: pm.variables.replaceIn("{{$randomFirstName}} {{$randomLastName}}"),
    randomEmail: pm.variables.replaceIn("{{$randomEmail}}"),
    randomPhone: "+57300" + Math.floor(Math.random() * 1000000),
    pastDate: new Date(Date.now() - Math.random() * 1000000000000).toISOString().split('T')[0]
};

pm.environment.set("test_name", testData.randomName);
pm.environment.set("test_email", testData.randomEmail);
pm.environment.set("test_phone", testData.randomPhone);
pm.environment.set("test_birth_date", testData.pastDate);
```

### Documentación de casos de prueba

#### Casos de prueba esenciales por endpoint:

**GET /users**
- ✅ Retorna lista de usuarios con status 200
- ✅ Respuesta en formato JSON válido
- ✅ Paginación funcional (si aplica)
- ✅ Filtros funcionan correctamente

**POST /users**
- ✅ Creación exitosa con datos válidos (201)
- ✅ Validación de campos requeridos (422)
- ✅ Validación de formato de email (422)
- ✅ Manejo de emails duplicados (400)
- ✅ Validación de fecha de nacimiento

**PUT /users/{id}**
- ✅ Actualización completa exitosa (200)
- ✅ Usuario no encontrado (404)
- ✅ Validación de datos (422)

**DELETE /users/{id}**
- ✅ Eliminación exitosa (200)
- ✅ Usuario no encontrado (404)
- ✅ Verificación de eliminación

---

## Buenas Prácticas Generales

### Estructura de proyecto recomendada
```
proyecto/
├── config/
│   ├── dependencies.php
│   ├── routes.php
│   └── middleware.php
├── src/
│   ├── Controllers/
│   ├── Services/
│   ├── Repositories/
│   ├── Models/
│   ├── DTOs/
│   ├── Strategies/
│   └── Exceptions/
├── tests/
├── public/
│   └── index.php
└── composer.json
```

### Convenciones de código
- **Nombres de clases**: PascalCase (`UserController`, `ProductService`)
- **Nombres de métodos**: camelCase (`findById`, `calculatePrice`)
- **Constantes**: UPPER_SNAKE_CASE (`MAX_RETRY_ATTEMPTS`)
- **Rutas**: kebab-case (`/api/users`, `/calculate-price`)

### Manejo de fechas
```php
// Siempre usar DateTime para fechas
$now = new DateTime();
$formatted = $now->format('Y-m-d H:i:s');

// Para fechas en diferentes zonas horarias
$bogota = new DateTime('now', new DateTimeZone('America/Bogota'));
```

### Validación de entrada
- Siempre validar y sanitizar datos de entrada
- Usar DTOs para encapsular validación
- Manejo consistente de errores
- Respuestas estructuradas en JSON

### Performance
- Lazy loading en repositories
- Paginación para listados grandes
- Cacheo de consultas frecuentes
- Índices en base de datos

### Seguridad
- Validación de entrada estricta
- Uso de prepared statements
- Manejo seguro de errores (no exponer información sensible)
- Headers de seguridad apropiados, int $quantity): float;
    public function getDescription(): string;
}
```

```php
<?php
// src/Strategies/RegularPriceStrategy.php
namespace App\Strategies;

use App\Models\Product;

class RegularPriceStrategy implements PriceCalculationInterface 
{
    public function calculate(Product $product, int $quantity): float {
        return $product->getPrice() * $quantity;
    }
    
    public function getDescription(): string {
        return 'Precio regular sin descuentos';
    }
}

// src/Strategies/BulkDiscountStrategy.php
class BulkDiscountStrategy implements PriceCalculationInterface 
{
    private float $discountPercentage;
    private int $minimumQuantity;
    
    public function __construct(float $discountPercentage, int $minimumQuantity) {
        $this->discountPercentage = $discountPercentage;
        $this->minimumQuantity = $minimumQuantity;
    }
    
    public function calculate(Product $product
```