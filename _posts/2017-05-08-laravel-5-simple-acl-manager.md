---
layout: post
title: "Laravel 5 Simple ACL manager"
description: "Simple ACL tutorial in Laravel 5 include model and controller."
tags: [laravel, acl]
---
Protect your routes with user roles. Simply add a 'role_id' to the User model, install the roles table and seed if you need some example roles to get going.

If the user has a 'Root' role, then they can perform *any* actions.

# Installation

Simply copy the files across into the appropriate directories, and register the middleware in App\Http\Kernel.php

Then specify a 'roles' middleware on the route you'd like to protect, and specify the individual roles as an array:

    Route::get('user/{user}', [
	     'middleware' => ['auth', 'roles'],
	     'uses' => 'UserController@index',
	     'roles' => ['administrator', 'manager']
    ]);

If you found this ACL manager helpful please give this repo a star, and give me a [follow](https://github.com/edwinlab). Any questions, please leave a comment.

CheckRole.php
```php
<?php namespace App\Http\Middleware;

// First copy this file into your middleware directoy

use Closure;

class CheckRole{

	/**
	 * Handle an incoming request.
	 *
	 * @param  \Illuminate\Http\Request  $request
	 * @param  \Closure  $next
	 * @return mixed
	 */
	public function handle($request, Closure $next)
	{
		// Get the required roles from the route
		$roles = $this->getRequiredRoleForRoute($request->route());

		// Check if a role is required for the route, and
		// if so, ensure that the user has that role.
		if($request->user()->hasRole($roles) || !$roles)
		{
			return $next($request);
		}
		return response([
			'error' => [
				'code' => 'INSUFFICIENT_ROLE',
				'description' => 'You are not authorized to access this resource.'
			]
		], 401);

	}

	private function getRequiredRoleForRoute($route)
	{
		$actions = $route->getAction();
		return isset($actions['roles']) ? $actions['roles'] : null;
	}

}
```

Kernel.php
```php
<?php

// Register the new route middleware

protected $routeMiddleware = [
		'auth' 			=> 'App\Http\Middleware\Authenticate',
		'auth.basic' 	=> 'Illuminate\Auth\Middleware\AuthenticateWithBasicAuth',
		'guest' 		=> 'App\Http\Middleware\RedirectIfAuthenticated',
		'roles' 		=> 'App\Http\Middleware\CheckRole',
	];
```

Role.php
```php
<?php namespace App;

use Illuminate\Database\Eloquent\Model;

class Role extends Model {

    protected $table = 'roles';

    public function users()
    {
        return $this->hasMany('App\User', 'role_id', 'id');
    }

}
```

Roles_table_migration.php
```php
<?php

use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateRolesTable extends Migration {

	/**
	 * Run the migrations.
	 *
	 * @return void
	 */
	public function up()
	{
		Schema::create('role', function($table) {
			$table->increments('id');
			$table->string('name', 40);
			$table->string('description', 255);
			$table->timestamps();
		});
	}

	/**
	 * Reverse the migrations.
	 *
	 * @return void
	 */
	public function down()
	{
		Schema::drop('role');
	}

}
```
RoleTableSeeder.php
```php
<?php



use Illuminate\Database\Seeder;
use Illuminate\Database\Eloquent\Model;
use App\Role;

class RoleTableSeeder extends Seeder{

    public function run()
    {

        if (App::environment() === 'production') {
            exit('I just stopped you getting fired. Love, Amo.');
        }

        DB::table('role')->truncate();

        Role::create([
            'id'            => 1,
            'name'          => 'Root',
            'description'   => 'Use this account with extreme caution. When using this account it is possible to cause irreversible damage to the system.'
        ]);

        Role::create([
            'id'            => 2,
            'name'          => 'Administrator',
            'description'   => 'Full access to create, edit, and update companies, and orders.'
        ]);

        Role::create([
            'id'            => 3,
            'name'          => 'Manager',
            'description'   => 'Ability to create new companies and orders, or edit and update any existing ones.'
        ]);

        Role::create([
            'id'            => 4,
            'name'          => 'Company Manager',
            'description'   => 'Able to manage the company that the user belongs to, including adding sites, creating new users and assigning licences.'
        ]);

        Role::create([
            'id'            => 5,
            'name'          => 'User',
            'description'   => 'A standard user that can have a licence assigned to them. No administrative features.'
        ]);
    }

}
```

routes.php
```php
<?php

Route::get('user/{user}', [
	'middleware' => ['auth', 'roles'], // A 'roles' middleware must be specified
	'uses' => 'UserController@index',
	'roles' => ['administrator', 'manager'] // Only an administrator, or a manager can access this route
]);
```

User.php
```php
<?php

// The User model

public function role()
	{
		return $this->hasOne('App\Role', 'id', 'role_id');
	}

	public function hasRole($roles)
	{
		$this->have_role = $this->getUserRole();

		// Check if the user is a root account
		if($this->have_role->name == 'Root') {
			return true;
		}

		if(is_array($roles)){
			foreach($roles as $need_role){
				if($this->checkIfUserHasRole($need_role)) {
					return true;
				}
			}
		} else{
			return $this->checkIfUserHasRole($roles);
		}
		return false;
	}

	private function getUserRole()
	{
		return $this->role()->getResults();
	}

	private function checkIfUserHasRole($need_role)
	{
		return (strtolower($need_role)==strtolower($this->have_role->name)) ? true : false;
	}
```