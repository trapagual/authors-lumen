TUTORIAL API LUMEN
==================

Siguiendo: https://auth0.com/blog/developing-restful-apis-with-lumen/

# INSTALACIÓN

    C:\wamp\www>composer create-project --prefer-dist laravel/lumen authors

# CONFIGURACIÓN

## Comprobación

    cd authors

    php -S localhost:8000 -t public

Debe aparecer la versión de lumen: `Lumen (6.0.2) (Laravel Components ^6.0)`        

Editar `bootstrap/app.php`, descomentar `// app->withEloquent`. Con esto tenemos Eloquent para acceso a datos, según esté configurado en `.env`.

Descomentar también `//$app->withFacades();`, para tener acceso a las [Facades](https://laravel.com/docs/6.x/facades) de Laravel.

# BASE DE DATOS, MODELOS, MIGRACIONES

    php artisan make:migration create_authors_table

Modificarlo para meterle los atributos que queremos para la tabla de autores.

```php
$table->string('name');
$table->string('email');
$table->string('github');
$table->string('twitter');
$table->string('location');
$table->string('latest_article_published');
```            

    php artisan migrate

Crear el modelo.
Creo que hay paquetes que te incluyen lo necesario para hacer php artisan make:model como en laravel. De momento lo vamos a hacer a mano.

`app/Author.php`      Ver en proyecto

Básicamente ponemos qué atributos van a poder asignarse en masa y cuáles vamos a omitir en las respuestas.

# RUTAS

Las rutas se configuran en `routes/web.php`. Aquí vamos a crear un grupo de rutas para el path 'authors'. Ver en proyecto.

En las rutas se declaran los verbos, la plantilla de atributos y los métodos de controlador que las atienden.
Aquí también decidimos que todas la rutas tengan el prefijo `/api/`.

# CONTROLADOR

`app/Http/Controllers/AuthorController.php`. Aquí también lo creamos a mano pero se aplica lo del paquete que vimos antes para el modelo.

Ver en el proyecto. Se crean los métodos a los que se referían las rutas, que a su vez tiran de los métodos adecuados del modelo `Author` que heredamos de Eloquent.

Todos devuelven un objeto `response` (que es un helper global que obtiene una instancia de la factoría de responses). `response->json()` devuelve la respuesta en formato JSON.

# PROBAR CON POSTMAN

O cualquier otro cliente REST

# VALIDACION

Lumen viene con el helper $this->validate que nos permite asegurarnos de que no nos cuelan cosas raras en el cuerpo de la petición.

Las modificaciones están en `app/Html/Controllers/AuthController.php`.

De nuevo podemos comprobarlo con POSTMAN, que nos rechazará la petición si no se cumplen las normas que hemos definido.

OJO que hay un error en el texto original en el que me baso. La validación para el campo email dice:

    'email' => 'required|email|unique:users',

lo que tiene todo el sentido en una aplicación completa (el email del autor --que ha de ser usuario-- tiene que ser único en la tabla users). Aquí en el tutorial no tenemos tabla users, así que si queremos comprobar que salta la validación al duplicar un email,tenemos que cambiar la tabla por authors. Así quedaría:

    'email' => 'required|email|unique:authors',

Aparte de todas las personalizaciones que podemos hacer, Lumen hereda de Laravel [todas estas validaciones](https://laravel.com/docs/6.x/validation#available-validation-rules).

# SECURIZACIÓN CON Auth0

Deberíamos restringir al menos POST, DELETE y PUT para que sólo puedan usarlo los usuarios registrados.

Aquí vamos a usar JWT (JSON Web Tokens) para securizar nuestras rutas. JWT espera un token, que suele ponerse en la cabecera de la solicitud y que se valida en varios pasos. 
Pueden usarse muchos sistemas para estas validaciones. En este tutorial usaremos Auth0, que tiene un modo gratuito que nos permite hacer pruebas. Hay otros proveedores de JWT que pueden usarse.

Nos creamos una cuenta gratuita en Auth0, e instalamos la librería cliente:

    composer require auth0/auth0-php:~5.0

Creamos un middleware ad-hoc, que irá en `app/Html/Middleware/Auth0Middleware.php`. Ver en el proyecto.

En la función `retrieveAndValidateToken($token)`, validamos el token recibido en la cabecera, poniendo en `AUTH0_API_AUDIENCE` y en `AUTH0_DOMAIN` los valores correspondientes a la cuenta y al API que hemos creado en Auth0 (o a los que tengamos de otro proveedor).

Ahora, asignamos el middleware a nuestras rutas. En bootstrap/app.php, descomentamos la parte de routemiddleware y sustituimos el nombre de clase que viene por el nuestro. Esto es registrar el middleware.

Queda así:

```php
$app->routeMiddleware([
    'auth' => App\Http\Middleware\Auth0Middleware::class,
]);
```
Ahora ya podemos usar el middleware recien registrado en nuestras rutas:

```php
$router->group(['prefix' => 'api', 'middleware' => 'auth'], function () use ($router) {
```
Si volvemos a POSTMAN tal cual, sin incluir token en la cabecera, el resultado será 'Authorization Header not found'.

Para probar con un token válido, tenemos que irnos a nuestro dashboard en nuestra cuenta de Auth0, pestaña Test, copiamos el token.
El mío es así:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6Ik5 ... lIhVw9mPOoCm2lKGSVITLm6MGiLX_T0HzYPoFaOK-bfG6h0b5QLo7_rF_auJUQ ... c4TFJTT9CazQ
```

Y lo usamos en POSTMAN como una cabecera de autenticación.

    Headers -> Authentication -> Bearer eyJ0eXAiOi...


## Errores en el tutorial de referencia

*OJO*: de nuevo al tutorial le faltan cosas y tal cual *no funciona*. Aquí te encontrarías un error de autenticación (no podemos confiar en un token emitido por ... ) porque el campo `authorized_iss` tiene que acabar en una barra. Así:

```php
'authorized_iss' => ['https://dev-o-2hzg9q.eu.auth0.com/'
```

Esto no es todo. Solucionado esto, te aparecerá otro error, este local, de cURL, diciendo que no encuentra el certificado del cliente.

Para este tienes que:

1. Bajarte la base de datos de autoridades de certificación de [aquí](https://curl.haxx.se/ca/cacert.pem) al disco duro.
1. Editar el `php.ini` y añadir `openssl.cafile=c:/cacert.pem`, apuntando al fichero que te has descargado.


Hecho esto, ahora sí funciona metiendo el token en la cabecera HTTP de la petición como un token de tipo Bearer. Recuerda reiniciar el servidor de desarrollo y el POSTMAN para que te cojan la nueva configuración.

Ahora puedes hacer peticiones REST a un API escrita en PHP con muy pocas líneas de código, utilizando Eloquent para la gestión de tablas y relaciones más complejas que estas, y preautorizando a las aplicaciones que vayan a poder pedirla. El cliente puede ser de cualquier tipo, mientras sea un cliente REST.

wt 09/2019


