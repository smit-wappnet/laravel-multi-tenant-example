# To Make This Project a Tenant Based project we have to follow this steps...

Before staring the setup make sure we have proper databse credentials in .env file

Install Package via Composer.
```bash
composer require stancl/tenancy
```

After this we have to make tenant setup by executing this command
```bash
php artisan tenancy:install
```

create Tenant Model (App\Models\Tenant.php)
```php
<?php

namespace App\Models;

use Stancl\Tenancy\Database\Models\Tenant as BaseTenant;
use Stancl\Tenancy\Contracts\TenantWithDatabase;
use Stancl\Tenancy\Database\Concerns\HasDatabase;
use Stancl\Tenancy\Database\Concerns\HasDomains;

class Tenant extends BaseTenant implements TenantWithDatabase
{
    use HasDatabase, HasDomains;
}
```

Add Central Domain to .env File
```
APP_CENTERAL_DOMAIN=localhost
```


Alter the config\tenancy.php
```php
...
'tenant_model' => App\Models\Tenant::class,
...
'central_domains' => [
    env('APP_CENTERAL_DOMAIN', 'localhost')
],
...
'prefix' => 'tenant', //Here you can define prifix for tenant database
...
```

Add TenancyServiceProvider in config\app.php
```php
'providers' => [
    ...
    App\Providers\TenancyServiceProvider::class,
],
```

Edit RouteServiceProvider.php file:
Add this functions
```php
protected function mapWebRoutes()
{
    foreach ($this->centralDomains() as $domain) {
        Route::middleware('web')
            ->domain($domain)
            ->namespace($this->namespace)
            ->group(base_path('routes/web.php'));
    }
}

protected function mapApiRoutes()
{
    foreach ($this->centralDomains() as $domain) {
        Route::prefix('api')
            ->domain($domain)
            ->middleware('api')
            ->namespace($this->namespace)
            ->group(base_path('routes/api.php'));
    }
}

protected function centralDomains(): array
{
    return config('tenancy.central_domains', []);
}
```
Replace the boot() method as follow.
```php
ppublic function boot()
{
    $this->configureRateLimiting();

    $this->routes(function () {
        $this->mapWebRoutes();
        $this->mapApiRoutes();
    });
}
```

To auto apply tenant to each user we edit the users migration and users model.
```php
// users migration
Schema::create('users', function (Blueprint $table) {
    ...
    $table->string('domain')->unique();
    ...
});

```
```php
// User Model
public static function booted()
{
    static::created(function($user){
        $userTenant = Tenant::create(['id' => $user->domain]);
        $userTenant->domains()->create(['domain' => $user->domain . '.' . env('APP_CENTERAL_DOMAIN', 'localhost')]);
    });
}
```

Now You can migrate your central database
```bash
php artisan migrate
```

To create any tenant migrations you have to put those migrations in database\migrations\tenant folder
to migrate tenant database...
```bash
php artisan tenant:migrate
```

Let Create one user and migrate the params table to the tenant database
Create params migraiton
```bash 
php artisan make:migration create_params_table
```

Move this migration to the tenant table and edit it.
```php
Schema::create('params', function (Blueprint $table) {
    $table->string('key')->primary();
    $table->text('value');
    $table->timestamps();
});

```

Creating new User
```bash
php artisan tinker
$user = new App\Models\User();
$user->name = "Demo";
$user->email = "demo@demodomain.com"
$user->password = "Password"
$user->domain = "demo";
$user->save();