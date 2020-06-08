Symfony Bundle integrating Consistence library and JMS Serializer with Symfony 5 support
========================================================================================
![Travis CI](https://travis-ci.org/scattercz/consistence-jms-serializer-symfony.svg?branch=master)

> This is a Symfony bundle providing integration for the standalone package
[`consistence/consistence-jms-serializer`](https://github.com/consistence/consistence-jms-serializer),
if you are not using Symfony, follow instructions there.

This bundle provides integration of [Consistence](https://github.com/consistence/consistence) value objects for [JMS Serializer](http://jmsyst.com/libs/serializer) so that you can use them in your serialization mappings.

For now, the only integration which is needed is for [Enums](https://github.com/consistence/consistence/blob/master/docs/Enum/enums.md), see the examples below.

This bundle was updated so it is now working in Symfony 5 and PHP 7.4.

Usage
-----

[Enums](https://github.com/consistence/consistence/blob/master/docs/Enum/enums.md) represent predefined set of values and of course, you will want to serialize and deserialize these values as well. Since [`Enums`](https://github.com/consistence/consistence/blob/master/src/Enum/Enum.php) are objects and you only want to (de)serialize represented value, there has to be some mapping.

You can see it in this example where you want to (de)serialize sex for your `User`s:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

class Sex extends \Consistence\Enum\Enum
{

	public const FEMALE = 'female';
	public const MALE = 'male';

}
```

Now you can use the `Sex` enum in your `User` object. Type for (de)serialization is specified as `enum<Your\Enum\Class>`:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

use JMS\Serializer\Annotation as JMS;

class User extends \Consistence\ObjectPrototype
{

	// ...

	/**
	 * @JMS\Type("enum<Consistence\JmsSerializer\Example\User\Sex>")
	 * @var \Consistence\JmsSerializer\Example\User\Sex|null
	 */
	private $sex;

	// ...

	public function __construct(
		// ...
		Sex $sex = null
		// ...
	)
	{
		// ...
		$this->sex = $sex;
		// ...
	}

}
```

Now everything is ready to be used, when you serialize the object, only `female` will be returned as the value representing the enum:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

/** @var \JMS\Serializer\Serializer $serializer */
$user = new User(Sex::get(Sex::FEMALE));
var_dump($serializer->serialize($user, 'json'));

/*

{
   "sex": "female"
}

*/
```

And when you deserialize the object, you will receive the `Sex` enum object again:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

/** @var \JMS\Serializer\Serializer $serializer */
var_dump($serializer->deserialize('{
   "sex": "female"
}', User::class, 'json'));

/*

class Consistence\JmsSerializer\Example\User\User#46 (1) {
  private $sex =>
  class Consistence\JmsSerializer\Example\User\Sex#5 (1) {
    private $value =>
    string(6) "female"
  }
}

*/
```

This means that the objects API is symmetrical (you get the same type as you set) and you can start benefiting from Enums advantages such as being sure, that what you get is already a valid value and having the possibility to define methods on top of the represented values.

### Nullable by default

Both serialization and deserialization will accept `null` values, there is no special mapping for that (by `Jms Serializer`s convention), so if you are expecting a non-null value you have to enforce this by other means - either by restricting this in the API of your objects or by validation (needed especially for deserialization).

### Invalid values

While serializing, there should be no invalid values, because [Enums](https://github.com/consistence/consistence/blob/master/docs/Enum/enums.md) guarantee that the instance contains only valid values.

While deserializing, there can be an invalid value given, an exception will be thrown:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

/** @var \JMS\Serializer\Serializer $serializer */
var_dump($serializer->deserialize('{
   "sex": "FOO"
}', User::class, 'json'));

// \Consistence\Enum\InvalidEnumValueException: FOO [string] is not a valid value, accepted values: female, male
```

If you are using this in an API, make sure you will catch this exception and send the consumer a response detailing this error, you can also write a custom message, the available values are listed in `InvalidEnumValueException::getAvailableValues()`.

### XML support

Unlike in JSON, in XML value types cannot be inferred directly from the values. So if you need to deserialize XML, you have to provide this type manually. You can do this by writing the type in the type definition - for the above example it would be `string`:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

use JMS\Serializer\Annotation as JMS;

class User extends \Consistence\ObjectPrototype
{

	// ...

	/**
	 * @JMS\Type("enum<Consistence\JmsSerializer\Example\User\Sex, string>")
	 * @var \Consistence\JmsSerializer\Example\User\Sex|null
	 */
	private $sex;

	// ...

}
```

### Special support for mapped MultiEnums

Since the (de)serialization works only with the value the enum is representing, then in case of [MultiEnums](https://github.com/consistence/consistence/blob/master/docs/Enum/multi-enums.md) this would mean outputting the value of the internal bit mask. This could be useful if both the client and server use the same Enum objects, but otherwise this breaks the abstraction and is less readable for a human consumer as well.

If you are using a [MultiEum mapped to a single Enum](https://github.com/consistence/consistence/blob/master/docs/Enum/multi-enums.md#mapping-a-multienum-to-a-single-enum) there is a handy solution provided, if you add to your mapping `enum<Your\Enum\Class, as_single>` - notice the new `as_single` parameter, then the value of `MultiEnum` will be serialized as a collection of single `Enum` values:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

use Consistence\Type\ArrayType\ArrayType;

use JMS\Serializer\Annotation as JMS;

class RoleEnum extends \Consistence\Enum\Enum
{

	public const USER = 'user';
	public const EMPLOYEE = 'employee';
	public const ADMIN = 'admin';

}

class RolesEnum extends \Consistence\Enum\MultiEnum
{

	/** @var int[] format: single Enum value (string) => MultiEnum value (int) */
	private static $singleMultiMap = [
		RoleEnum::USER => 1,
		RoleEnum::EMPLOYEE => 2,
		RoleEnum::ADMIN => 4,
	];

	public static function getSingleEnumClass(): string
	{
		return RoleEnum::class;
	}

	/**
	 * Converts value representing a value from single Enum to MultiEnum counterpart
	 *
	 * @param string $singleEnumValue
	 * @return int
	 */
	protected static function convertSingleEnumValueToValue($singleEnumValue): int
	{
		return ArrayType::getValue(self::$singleMultiMap, $singleEnumValue);
	}

	/**
	 * Converts value representing a value from MultiEnum to single Enum counterpart
	 *
	 * @param int $value
	 * @return string
	 */
	protected static function convertValueToSingleEnumValue(int $value): string
	{
		return ArrayType::getKey(self::$singleMultiMap, $value);
	}

}

class User extends \Consistence\ObjectPrototype
{

	// ...

	/**
	 * @JMS\Type("enum<Consistence\JmsSerializer\Example\User\RolesEnum, as_single>")
	 * @var \Consistence\JmsSerializer\Example\User\RolesEnum
	 */
	private $roles;

	// ...

	public function __construct(
		// ...
		RolesEnum $roles
		// ...
	)
	{
		// ...
		$this->roles = $roles;
		// ...
	}

}

$user = new User(RolesEnum::getMultiByEnums([
	RoleEnum::get(RoleEnum::USER),
	RoleEnum::get(RoleEnum::ADMIN),
]));

/** @var \JMS\Serializer\Serializer $serializer */
var_dump($serializer->serialize($user, 'json'));

/*

{
   "roles": [
      "user",
      "admin"
   ]
}

*/
```

Deserialization then again works symmetrically - giving an array of single `Enum` values will produce a `MultiEnum` instance:

```php
<?php

namespace Consistence\JmsSerializer\Example\User;

/** @var \JMS\Serializer\Serializer $serializer */
var_dump($serializer->deserialize('{
   "roles": [
      "user",
      "admin"
   ]
}', User::class, 'json'));

/*

class Consistence\JmsSerializer\Example\User\User#48 (1) {
  private $roles =>
  class Consistence\JmsSerializer\Example\User\RolesEnum#37 (1) {
    private $value =>
    int(5)
  }
}

*/
```

Installation
------------

1) Install package [`scattercz/consistence-jms-serializer-symfony`](https://packagist.org/packages/consistence/consistence-jms-serializer-symfony) with [Composer](https://getcomposer.org/):

```bash
composer require scattercz/consistence-jms-serializer-symfony
```

2) Register the bundle in your application:

```php
// config/bundles.php
return [
	// ...
	Consistence\JmsSerializer\SymfonyBundle\ConsistenceJmsSerializerBundle::class => ['all' => true],
];
```

That's all, you are good to go!
