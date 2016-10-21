# Using MySQL 5.7 On CodeShip 

For the article I will cover how to use MySQL 5.7 when running CI builds on CodeShip. Why MySQL 5.7? Because it finally has support for JSON queries as such the following

```
$results = \App\Example::where("data->foo", "here")->first();
```

You can read more about this [here](https://laravel.com/docs/5.2/queries#json-where-clauses)

That means I can store JSON data like this

```
{
  "foo" => "here",
  "bar" => "baz"
}
```

In one column.

Yes PostgreSQL has been able to do this for some time but for the time being I am still on MySQL and I really wanted to take advantage of this in a current project. In this project I had data coming in from different sources and I did not want to make unique tables and columns to store all the different data.

Instead I could dump it all in a "JSON" field type and query it as above. 

So by the time you are done reading this post you will have a very easy path to setup RDS on AWS, and use MySQL 5.7 from your local to your CI.

## Setting up MySQL on AWS Using RDS and CloudFormation

To get started I am going to make an RDS database for CodeShip to connect to. This database will be MySQL 5.7. I will use a [CloudFormation](https://aws.amazon.com/documentation/cloudformation/) script for this. CloudFormation is an amazing way to manage resources on AWS.

You can find the script [here](https://github.com/alnutile/codeship-behat/blob/master/ci/cloudformation_database.json)

All you need to do is log into your AWS account, go to CloudFormation and upload this script. You will be prompted with some questions. 

![cloudformation_questions](https://dl.dropboxusercontent.com/s/4jy963s7k9xzsyt/cloudformation_questions.png?dl=0)

After this page you can just click okay the rest of the way through.

In this example I will call the stack `codeship-testing` and give it a really solid password. Lastly I will name the database `codeship`. That is it! Let it run for 15 minutes an you will have a 5.7 MySQL instance that will allow connections.

Now let's setup our code to use this.

Then in order to make my connection secure from CodeShip to this RDS Instance, I will download the AWS pem file from [here](https://s3.amazonaws.com/rds-downloads/rds-combined-ca-bundle.pem) and save that to my `ci` folder in the root of my application. 

![pem](https://dl.dropboxusercontent.com/s/fa146olgljha6zk/pem.png?dl=0)

Then I will add that to my database settings. `config/database.php`
    
```
    'mysql_testing' => [
      'driver'    => 'mysql',
      'host'      => env('DB_HOST', 'localhost'),
      'database'  => env('DB_DATABASE', 'forge'),
      'username'  => env('DB_USERNAME', 'forge'),
      'password'  => env('DB_PASSWORD', ''),
      'charset'   => 'utf8',
      'collation' => 'utf8_unicode_ci',
      'prefix'    => '',
      'strict'    => false,
      'options'   => codeship_ssl()
    ],
    
    'mysql' => [
      'driver'    => 'mysql',
      'host'      => env('DB_HOST', 'localhost'),
      'database'  => env('DB_DATABASE', 'forge'),
      'username'  => env('DB_USERNAME', 'forge'),
      'password'  => env('DB_PASSWORD', ''),
      'charset'   => 'utf8',
      'collation' => 'utf8_unicode_ci',
      'prefix'    => '',
      'strict'    => false,
    ],
```

But instead of in the default `mysql` I am going to put this in `mysql_testing`. I am going to suggest this because I am also about to add the pem file `option` as seen here, ONLY if we are on the `codeship` database.

```
'options'   => codeship_ssl()
```

by adding the method to the top of this same file 

```
<?php

if(!function_exists('codeship_ssl')) {
    function codeship_ssl() {
        if (env('DB_DATABASE') == 'codeship')
        {
            $path_to_pem = base_path('/ci/rds-combined-ca-bundle.pem');
            return [PDO::MYSQL_ATTR_SSL_CA => $path_to_pem];
        }
    }
}

////
```

You can see the full file [here](https://github.com/alnutile/codeship-behat/blob/master/config/database.php).

Adding it to the `mysql` connection would work but I just did not want to have this extra configuration part of all my non-testing interactions.  

Then I will update my `phpunit.xml` to reference that `connection.

```
////
    </filter>
    <php>
        <env name="DB_CONNECTION" value="mysql_testing"/>
        <env name="APP_ENV" value="testing"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="SESSION_DRIVER" value="array"/>
        <env name="QUEUE_DRIVER" value="sync"/>
    </php>
</phpunit>
``` 

## Setting up CodeShip

Now to setup CodeShip so it will have the correct environment settings to trigger us to use the `mysql_testing` connection and the pem file.

Here is what our `Environment` settings for CodeShip by the time I am done.

![codeship_env](https://dl.dropboxusercontent.com/s/08fx4vy7yecydmt/codeship_variables.png?dl=0)

The `DB_DATABASE` `codeship` is the state I look for in the `config/database.php` file.

That is it for CodeShip settings!

## Example of this Working
Now I want to show this working. Since I am using Laravel I will go through the steps needed to make a simple example.

First I will make a migration to include this JSON field.

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class AddJsonExample extends Migration
{
    public function up()
    {
        Schema::create('examples', function (Blueprint $table) {
            $table->increments('id');
            $table->json('data');
        });
    }

///
```

Then I will make a factory to easily populate it from my PHPUnit test.

`database/factories/ModelFactory.php`

```
$factory->define(App\Example::class, function (Faker\Generator $faker) {

    return [
        'data' => [
	        "foo" => $faker->text,
	        "bar" => $faker->text,
        ]
    ];
});
```

And a Model to cast it to an array `app/Example.php`

```
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;

class Example extends Model
{
    public $timestamps = false;
    
    protected $casts = [
        'data' => 'array'
    ];

}
```

Lastly I will make a PHPUnit test to show it working 

`tests/QueryJsonTest.php`

```
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Illuminate\Foundation\Testing\DatabaseTransactions;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\DB;

class QueryJsonTest extends TestCase
{
    use DatabaseMigrations, DatabaseTransactions;

    public function setUp()
    {
        parent::setUp();
    }

    /**
     * @test
     */
    public function will_query_json()
    {
        factory(\App\Example::class, 5)->create(
            [
                'data' => [
                    "foo" => "baz",
                    "bar" => "boo",
                ]
            ]
        );


        factory(\App\Example::class)->create([
            'data' => [
                "foo" => "here",
                "bar" => "not-here",
            ]
        ]);
    
        $results = \App\Example::where("data->foo", "here")->first();
        
        PHPUnit_Framework_Assert::assertNotNull($results);
    }

}

```

![passing](https://dl.dropboxusercontent.com/s/66dxbeewny73m5k/passing.png?dl=0)


That is it. I now can work locally using 5.7 and have CodeShip run our tests with the same version as my local and Production.


## References

http://stackoverflow.com/questions/38710723/connect-laravel-5-to-aws-rds-with-ssl

https://laravel.com/docs/5.2/queries#json-where-clauses

http://themsaid.com/mysql-json-data-type-20160311/

