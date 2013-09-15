PHP5, Symfony2 y otros conceptos en Drupal 8
===========================================

Éste repositorio contiene documentación y ejemplos sobre conceptos que Drupal 8 implementa.

# PHP5

Drupal 7 requiere PHP 5.2.5, mientras que Drupal 8 requiere [PHP 5.3.0](http://php.net/releases/5_3_0.php),
versión que introduce, entre otros, los siguientes conceptos:


## [Namespaces](http://es1.php.net/namespaces)

Un namespace define la localización de de clases, interfaces, funciones y constantes. En Java se usan desde
hace mucho tiempo.

Por el momento, PHP soporta Namespaces sólo para clases, no para funciones.

```php
// core/modules/block/lib/Drupal/block/BlockRenderController.php
namespace Drupal\block;

use Drupal\Core\Entity\EntityRenderControllerInterface;
use Drupal\Core\Entity\EntityInterface;

/**
 * Provides a Block render controller.
 */
class BlockRenderController implements EntityRenderControllerInterface {
```

Gracias a esto, podemos implementar un Autoloader que, cuando instanciamos una clase,
busque el archivo que la contiene y la cargue automáticamente.

En Drupal 7, autoload se realiza a través de los .info. Por ejemplo,
`files[] = filter.test`

En Drupal 8, con definir el namespace y colocar el archivo en el
directorio adecuado, basta:

```php
// core/modules/filter/lib/Drupal/filter/Tests/FilterCrudTest.php
namespace Drupal\filter\Tests;

use Drupal\simpletest\DrupalUnitTestBase;

/**
 * Tests for text format and filter CRUD operations.
 */
class FilterCrudTest extends DrupalUnitTestBase {
```

La razón por la cuál las clases dentro de un módulo de Drupal 8 están en
lib/Drupal/[módulo] es para permitir a Drupal registrar las clases en el
Autoloader:

```php
// core/lib/Drupal/core/DrupalKernel.php
    
// Get a list of namespaces and put it onto the container.
$namespaces = $this->getModuleNamespaces($this->getModuleFileNames());
// Add all components in \Drupal\Core and \Drupal\Component that have a
// Plugin directory.
foreach (array('Core', 'Component') as $parent_directory) {
  $path = DRUPAL_ROOT . '/core/lib/Drupal/' . $parent_directory;
  foreach (new \DirectoryIterator($path) as $component) {
    if (!$component->isDot() && is_dir($component->getPathname() . '/Plugin')) {
      $namespaces['Drupal\\' . $parent_directory  .'\\' . $component->getFilename()] = DRUPAL_ROOT . '/core/lib';
    }
  }
}
```

## [Funciones anónimas](http://es1.php.net/manual/en/functions.anonymous.php)

También conocidas como funciones lambda o closures, permiten definir una función on the fly.

Ésto es muy común en JavaScript:

```javascript
$.post('ajax/test.html', function(data) {
  $('.result').html(data);
});
```
Hasta ahora, cuando teníamos que definir una callback function en PHP 5.2, teníamos que hacerlo
de éste modo:

```php
// http://php.net/manual/en/function.array-map.php#example-4915
function cube($n) {
  return($n * $n * $n);
}

$a = array(1, 2, 3, 4, 5);
$b = array_map("cube", $a);
print_r($b);
```

Mientras que en PHP 5.3 podemos encapsular la función en una variable. Por si queremos
reutilizarla en el contexto en el que se encuentra.
```php
$cube = function($value) {
    return $value * $value * $value;
};

$a = array(1, 2, 3, 4, 5);
$b = array_map($cube, $a);
print_r($b);
```

O podemos implementarla directamente:
```php
$a = array(1, 2, 3, 4, 5);
$b = array_map(function($value) { return $value * $value * $value; }, $a);
```

## Orientación a Objetos en Drupal 8

Drupal 7 implementa una orientación a objetos muy básica. Como mucho vemos clases, herencia,
y algún interfaz.

Los hooks en Drupal 7 se implementan siguiendo una convención en el nombre
de la función dentro de un módulo

```php
// modules/user/user.module
$items['user/logout'] = array(
  'title' => 'Log out',
  'access callback' => 'user_is_logged_in',
  'page callback' => 'user_logout',
  'weight' => 10,
  'menu_name' => 'user-menu',
  'file' => 'user.pages.inc',
);

```

Mientras que en Drupal 8 se hace uso extensivo de herencia e interfaces para implementar
eventos en vez de hooks.

Por ejemplo, el evento `authentication_provider` permite definir métodos para
autentificar una petición (por defecto se mira si hay una cookie con un usuario).
El módulo OAuth se registra a este evento mediante la opción `tags`:

```
http://drupalcode.org/project/oauth.git/blob/refs/heads/8.x-1.x:/oauth.services.yml
services:
  authentication.oauth:
    class: Drupal\oauth\Authentication\Provider\OAuthDrupalProvider
    tags:
      - { name: authentication_provider, priority: 100 }
```

Luego, el Event dispatcher de Symfony2 avisa a todas las clases registradas
a un evento en particular:

```php
// core/vendor/symfony/event-dispatcher/Symfony/Component/EventDispatcher/EventDispatcher.php
/**
 * @see EventDispatcherInterface::dispatch
 *
 * @api
 */
public function dispatch($eventName, Event $event = null)
{
  if (null === $event) {
    $event = new Event();
  }

  $event->setDispatcher($this);
  $event->setName($eventName);

  if (!isset($this->listeners[$eventName])) {
    return $event;
  }

  $this->doDispatch($this->getListeners($eventName), $eventName, $event);
    return $event;
  }
}
```

Nuestra clase en OAuth implementa un interfaz, el cual define qué métodos debe
implementar para comportarse correctamente:

```php
// http://drupalcode.org/project/oauth.git/blob/refs/heads/8.x-1.x:/lib/Drupal/oauth/Authentication/Provider/OAuthDrupalProvider.php

namespace Drupal\oauth\Authentication\Provider;

use Drupal\Core\Authentication\AuthenticationProviderInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpKernel\Event\GetResponseForExceptionEvent;
use \OauthProvider;

/**
 * Oauth authentication provider.
 */
class OAuthDrupalProvider implements AuthenticationProviderInterface {

  /**
   * An authenticated user object.
   *
   * @var \Drupal\user\UserBCDecorator
   */
  protected $user;

  /**
   * {@inheritdoc}
   */
  public function applies(Request $request) {
    // Only check requests with the 'authorization' header starting with OAuth.
    return preg_match('/^OAuth/', $request->headers->get('authorization'));
  }

  /**
   * {@inheritdoc}
   */
  public function authenticate(Request $request) {
    // Here we authenticate the request.
  }

}
```




