# Informe de Errores y Mejoras Detectadas

Este documento detalla los errores encontrados en el repositorio, la ruta exacta de cada uno, cómo solucionarlos y posibles problemas adicionales a tener en cuenta.

---

## 1. Nombre Incorrecto de la Clase y Archivo: `ElloquentUserRepository`

- **Ruta del archivo:**  
  `src/Infrastructure/Persistence/User/ElloquentUserRepository.php`
- **Rutas donde se referencia:**  
  - `app/repositories.php`
  - Cualquier otro archivo que haga uso de este repositorio.

- **Descripción del error:**  
  El nombre correcto en inglés para el repositorio es `EloquentUserRepository`. El error de tipeo puede causar problemas de autoload, confusión y fallos en la inyección de dependencias.

- **Solución:**  
  1. Renombrar el archivo a `EloquentUserRepository.php`.
  2. Cambiar el nombre de la clase dentro del archivo a `EloquentUserRepository`.
  3. Actualizar todas las referencias en el código y archivos de configuración a `EloquentUserRepository`.

---

## 2. Uso de Variable No Definida en Settings

- **Ruta del archivo:**  
  `src/Application/Settings/Settings.php`
- **Línea:**  
  15 (aprox.)

- **Descripción del error:**  
  Se usa `$this->resolvedEntries` pero no está definida ni inicializada en la clase. Esto puede generar un error de variable indefinida.

- **Solución:**  
  Modifica el método `get` para que solo use `$this->data`:
  ```php
  public function get(string $key = '')
  {
      if (array_key_exists($key, $this->data)) {
          return $this->data[$key];
      }
      throw new NotFoundSettingException("No se encuentra la llave '$key'");
  }
  ```

---

## 3. Mensaje de Excepción como Propiedad Pública

- **Rutas de los archivos:**  
  - `src/Domain/DomainException/User/UserAlreadyExistsException.php`
  - `src/Domain/DomainException/User/UserNotFoundException.php`
- **Línea:**  
  Aproximadamente línea 9 en ambos archivos.

- **Descripción del error:**  
  El mensaje de excepción se asigna como propiedad pública `$message`. Según las mejores prácticas de PHP, debe definirse en el constructor de la excepción padre.

- **Solución:**  
  Ejemplo para `UserAlreadyExistsException`:
  ```php
  class UserAlreadyExistsException extends DomainRecordConflictException
  {
      public function __construct()
      {
          parent::__construct('El correo electrónico o nombre de usuario ya se encuentra registrado.');
      }
  }
  ```
  Aplica lo mismo para `UserNotFoundException`.

---

## 4. Posibles Errores Adicionales (Recomendaciones)

> **Nota:** Los siguientes puntos no son errores confirmados, pero son buenas prácticas y recomendaciones para prevenir futuros problemas.

- **Verificar las rutas y controladores**
  - Asegúrate de que los controladores como `UserController` y sus métodos existen y están correctamente definidos.
  - Verifica la correcta definición de rutas en `app/routes.php`.

- **Variables de entorno**
  - Verifica que el archivo `.env` contenga todas las variables necesarias (`DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASS`, etc.).

- **Autoload y Composer**
  - Asegúrate de que el autoload esté bien configurado en `composer.json` y ejecuta `composer dump-autoload` tras realizar cambios en nombres de clases o rutas.

- **Pruebas**
  - Verifica la existencia y funcionalidad de los tests en la carpeta `tests/`. Si no hay tests, es recomendable agregarlos.

- **No subir información sensible**
  - Nunca subas el archivo `.env` ni credenciales a repositorios públicos.

---

## Referencias

- [Ver búsqueda de código en GitHub](https://github.com/addsdev-campuslands/My-Garden-Php-Dev-Container/search)
- [Documentación de excepciones en PHP](https://www.php.net/manual/es/class.exception.php)
- [PSR-4: Autoloading Standard](https://www.php-fig.org/psr/psr-4/)

---

**La identificacion de errores no es totalmente perfecto, pero sirve de base para la correcion de otro tipos de errores que puedan salir en el examen del 8 de agosto de 2025.**
**Todo esto fue analizado por Kevin Gutierrez - Estudiante de Campuslands.**