---
title: "Drupal 11.1 Hooks as Classes: Modernizing Your Custom Module Development"
description: "Drupal 11.1 introduces hooks as class methods with PHP attributes. Here's why this matters, how it changes module development, and why you should adopt it for new code."
date: 2026-04-19T13:21:43+01:00
image: "images/DRUPAL-EL_blue_RGB.png"
draft: false
type: "post"
tags: ["drupal", "drupal11", "php", "module-development", "hooks"]
---

For as long as Drupal has existed, hooks have been procedural functions. You write `mymodule_form_alter()` in a `.module` file, Drupal invokes it by name, and you modify a form. It works. It's familiar. And it's stuck in the procedural PHP era.

Drupal 11.1 introduces an alternative: hooks implemented as class methods with PHP attributes. Instead of procedural functions named by convention, you write object-oriented code with explicit declarations.

This isn't a breaking change - procedural hooks still work. But for new code, this approach is superior in almost every way.

## The Old Way: Procedural Hooks

Historically, you implemented hooks like this:

```php
<?php
// In my_module.module

function my_module_form_alter(&$form, &$form_state, $form_id) {
  if ($form_id === 'contact_message_form') {
    $form['#submit'][] = 'my_module_custom_submit';
  }
}

function my_module_custom_submit(&$form, &$form_state) {
  // Handle form submission
}
```

This approach has deep roots in PHP history. Hooks are identified by function name. Drupal uses naming conventions (module_name + underscore + hook_name) to find them. It's declarative by convention, not by declaration.

The trade-offs were acceptable for simple modules. But they create problems at scale:

- **Poor IDE support** - Your IDE can't statically analyse a function named by string convention
- **Hard to test** - You have to instantiate the entire Drupal system to call a hook function
- **No dependency injection** - If you need a service, you call `\Drupal::service()` within the function
- **Organisation issues** - Related hooks are scattered across the `.module` file
- **Performance** - `.module` files are always loaded, even if the hooks inside aren't invoked

## The New Way: Hooks as Classes (Drupal 11.1+)

Drupal 11.1 introduces a better pattern. [According to Drupalize.Me](https://drupalize.me/blog/drupal-111-adds-hooks-classes-history-how-and-tutorials-weve-updated), hooks can now be implemented as class methods with PHP attributes:

```php
<?php
// In src/Hook/FormAlter.php

namespace Drupal\my_module\Hook;

use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Hook\Attribute\Hook;

class FormAlter {
  #[Hook('form_alter')]
  public function formAlter(&$form, FormStateInterface $form_state, $form_id) {
    if ($form_id === 'contact_message_form') {
      $form['#submit'][] = 'my_module_custom_submit';
    }
  }

  #[Hook('form_contact_message_form_submit')]
  public function submitContactForm(&$form, FormStateInterface $form_state) {
    // Handle form submission
  }
}
```

The `#[Hook('hook_name')]` attribute tells Drupal which hook the method implements. No naming convention. No string-based discovery. Explicit declaration.

## Five Reasons This Matters

### 1. Better IDE Support and Static Analysis

Your IDE now understands hook signatures. It provides:
- Autocomplete for hook parameters
- Type checking - if you have the wrong parameter signature, your IDE flags it before you run code
- Refactoring - rename a parameter, and your IDE can update all usages
- Jump-to-definition - click on a hook name, jump to its declaration

This is something IDEs have been doing for object-oriented code for years. Now hooks get the same treatment.

### 2. Dependency Injection (No More \Drupal::service())

The real power emerges when you combine attributes with dependency injection. Drupal can now instantiate your hook class with the services it needs:

```php
<?php
namespace Drupal\my_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\my_module\MyService;

class FormAlter {
  public function __construct(private MyService $service) {}

  #[Hook('form_alter')]
  public function formAlter(&$form, &$form_state, $form_id) {
    $this->service->doSomething($form);
  }
}
```

Instead of calling `\Drupal::service('my_service')` inside the hook, you declare the service as a constructor parameter. Drupal's service container automatically injects it. You get:
- Testability - pass a mock service in unit tests
- Clarity - dependencies are explicit, not hidden in function calls
- Performance - dependency resolution happens once at container compilation, not on every hook invocation

### 3. Dramatically Improved Testability

Unit testing hook classes is straightforward:

```php
<?php
namespace Drupal\Tests\my_module\Unit\Hook;

use PHPUnit\Framework\TestCase;
use Drupal\my_module\Hook\FormAlter;
use Drupal\Tests\my_module\Stub\MyServiceStub;

class FormAlterTest extends TestCase {
  public function testFormAlterWorks() {
    $service = new MyServiceStub();
    $hook = new FormAlter($service);

    $form = [];
    $form_state = $this->createMock(FormStateInterface::class);

    $hook->formAlter($form, $form_state, 'contact_message_form');

    $this->assertArrayHasKey('#submit', $form);
  }
}
```

You instantiate the hook class directly, pass in mocked dependencies, and test the logic. No Drupal bootstrap. No setting up the full test environment. This is dramatically faster than functional tests.

### 4. Better File Organization

Related hooks can now live together in a class:

```php
<?php
// src/Hook/Forms.php - all form-related hooks in one place

namespace Drupal\my_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;

class Forms {
  #[Hook('form_alter')]
  public function alter(&$form, &$form_state, $form_id) { }

  #[Hook('form_validate')]
  public function validate(&$form, &$form_state) { }

  #[Hook('form_submit')]
  public function submit(&$form, &$form_state) { }
}
```

Instead of all hooks scattered through one `.module` file, you can organise them by concern: Forms, Entities, Views, etc. Each class has a single responsibility.

### 5. Performance: Lazy Loading

This is subtle but important. `.module` files are always loaded by Drupal, even if the hooks inside never execute on a particular request. Hook classes using PSR-4 autoloading are only loaded when needed.

On a site with dozens of modules, each with a `.module` file full of hooks, this adds up. Hook classes reduce the amount of PHP that needs to be parsed on every request.

## When to Use Each Approach

**Use hook classes for new code.** This is the modern, preferred approach. All the advantages above apply. Your team (and future maintainers) will thank you.

**Procedural hooks still work.** If you're maintaining an existing module, there's no urgent need to rewrite all your hooks. They'll continue to work. But when you add new hooks, consider implementing them as classes.

**Neither is wrong.** Drupal 11 supports both. The transition is gradual. You can mix approaches in the same module if needed (though it's cleaner to be consistent).

## The Broader Context

This change reflects Drupal's evolution. For years, Drupal has been modernising its code patterns:
- Replacing procedural database queries with ORM
- Introducing services and dependency injection
- Moving away from global state (`$_GET`, `$_POST` globals) toward request objects
- Adopting PSR standards for autoloading and coding standards

Hooks as classes is the natural culmination of these trends. It brings hook development into alignment with modern PHP practices.

## Migration: The Long View

Drupalize.Me is updating all their tutorials to reflect the new approach. This signals that the Drupal community expects this to become the standard.

For your own modules:
- Use hook classes for new code immediately
- Plan to migrate old hooks gradually (whenever you touch the code anyway)
- Don't rush a wholesale rewrite - the benefit is real, but not so urgent that it justifies rewriting working code

## What You Need to Know

**PHP version requirement:** Hook classes require PHP 8.1+ for attribute support. If you're supporting older PHP versions, you're stuck with procedural hooks for now.

**Drupal version requirement:** Hook classes are available in Drupal 11.1+. There's no backport to Drupal 10.

**Documentation:** The Drupal API documentation for hooks is being updated. Check [drupalize.me](https://drupalize.me/blog/drupal-111-adds-hooks-classes-history-how-and-tutorials-weve-updated) and the official Drupal documentation for examples specific to each hook.

## The Practical Next Step

If you're starting a new Drupal 11 module, or adding features to an existing one:

1. **Create a `src/Hook/` directory** in your module
2. **Create a class file** for related hooks (FormAlter.php, Entities.php, etc.)
3. **Add the `#[Hook('hook_name')]` attribute** to each method
4. **Use dependency injection** in the constructor for any services you need
5. **Unit test the hooks** - now that they're isolated classes, testing is straightforward

This is the future of Drupal module development. It's not mandatory yet, but it should be your default choice for new code.

## Resources

- [Drupalize.Me: Drupal 11.1 Hooks as Classes](https://drupalize.me/blog/drupal-111-adds-hooks-classes-history-how-and-tutorials-weve-updated)
- [Drupal API: Hook Attribute Documentation](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21Hook%21Attribute%21Hook.php/class/Hook/11.x)
- [Drupal Module Development Guide](https://www.drupal.org/docs/drupal-apis/module-system)
- [Drupal Service Container and Dependency Injection](https://www.drupal.org/docs/drupal-apis/services-and-dependency-injection)
- [PHP Attributes (RFC 8235)](https://wiki.php.net/rfc/attributes_v2)
