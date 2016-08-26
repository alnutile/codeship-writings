
  *  Laravel and getting started
        *  Link off to other post they made?
  *  Start basic app and branch off to Codeship
  *  Scripting it in their ui then not using their info
  *  Starting the server
  *  Https and not just be careful
  *  Example assets true
  *  .env.codeship and copy over these settings
  *  Note how it differs from local
  *  Ssh in to fix 
  *  Png output 
  *  And when all else fails saucelabs
  *  Behat running domain level
  *  Behat.yml settings
  *  Behat JavaScript tag
  *  HappyPath later schedule a daily
  


The goal of this post will to not only get you going for Selenium and Headless Chrome, which is key for JavaScript based testing, but also how to trouble shoot issues that are not happening on your local build but on the CI using CodeShips super helpful SSH feature and Saucelabs remote connections.

First we need to setup CodeShip to even setup and test our app. You can see this at a previous Laravel Centric post here https://blog.codeship.com/getting-started-laravel-codeship/

At this point you should have things working and PHPUnit running. We are going to then add Behat to this.

Getting started really comes down to having this work on your local environment. In this post there will be a Host (Mac) and a Guest (Linux) thanks to Homestead. I will list at the end of this article some links for Windows as well.

![behat drives selenium](https://dl.dropboxusercontent.com/s/gsl0ube5ybxtpaq/behat_drives.png?dl=0)


First part of this setup is based on this Laravel oriented library started by [Laracast's](https://laracasts.com/) creator Jeffrey Way [https://github.com/laracasts/Behat-Laravel-Extension](https://github.com/laracasts/Behat-Laravel-Extension).  After you follow the install steps you will have a working version of Behat that integrates with your Laravel in some nice ways including Migrations and Transactions hooks.


But even after the above I need to take it one step further I need to get Selenium setup both as a server and a Mink Extension.


```
composer require "behat/mink-selenium2-driver":"^1.3"
```

This will get you going for Mink and Selenium.

Now for Selenium, this will be a bit harder but really not much. Remember this is on your Host, the above was in your guest. Basically if you visit [here](https://www.npmjs.com/package/selenium-standalone) you can easily install Selenium on any Host OS. For me and my Mac I use `brew` to set up NodeJS and from there I follow those 3 steps to get going. When I am done I have a terminal in the background just running Selenium.

Now for my Behat test. At this point this is my `behat.yml` in the root of my Application. 


```
default:
  suites:
    user_auth:
      contexts: [ UserAuthenticationContext ]
      filters: { tags: '@user_auth' }

  extensions:
    Laracasts\Behat:
        # env_path: .env.behat
    Behat\MinkExtension:
        base_url: https://codeship-behat.dev
        default_session: laravel
        laravel: ~
        selenium2:
          wd_host: "http://192.168.10.1:4444/wd/hub"
        browser_name: chrome
```

We will build off this in a moment to add CodeShip. But for now I have one suite to get started `user_auth` and one profile `default`. I have the `base_url` for the local site `https://codeship-behat.dev` and for now it is using my Applications `.env` file `# env_path: .env.behat`. This will change for the CodeShip profile.

Notice too I used the ip of my Host to talk to Selenium from inside my guest `http://192.168.10.1:4444/wd/hub` one more thing that will change for the CodeShip Profile.

So now to prove all of this is working I start by running 


```
vendor/bin/behat --init
```

If you did not do this in the Behat-Laravel-Extension docs. Then 


Now to make my `feature` file `features/user_auth.feature` 

![feature file](https://dl.dropboxusercontent.com/s/cjtu1616oog86zr/feature_file.png?dl=0)


And from here we need to fill in the details 


```
Feature: User Login Area
  User can log into the site
  As an anonymous user
  So that they can see secured parts of the site

  @happy_path @user_auth @javascript
  Scenario: Logging in with Success
    Given I visit the login page
    And I fill in the form with my username and password and submit the form
    Then I should see "You are logged in!"

```

Focusing on the `@happy_path` for now we tag it `@user_auth` so I know it is part of the Suite as seen in the `behat.yml` file above `filters: { tags: '@user_auth' }`. I could have used folders for this but chose tags.

Now if I run 


```
vendor/bin/behat --suite user_auth --init
```
 
I get a file `features/bootstrap/UserAuthenticationContext.php` that is pretty empty and needs to be told to extend `Mink Context`

So I just need to change the "extends" section so it looks like this


```
<?php

use Behat\Behat\Hook\Scope\AfterStepScope;
use Behat\Behat\Tester\Exception\PendingException;
use Behat\Behat\Context\Context;
use Behat\Behat\Context\SnippetAcceptingContext;
use Behat\Gherkin\Node\PyStringNode;
use Behat\Gherkin\Node\TableNode;
use Behat\Mink\Driver\Selenium2Driver;
use Behat\MinkExtension\Context\MinkContext;

/**
 * Defines application features from the specific context.
 */
class UserAuthenticationContext extends MinkContext implements Context, SnippetAcceptingContext
{
    public function __construct()
    {


    }

```

Alright so we now want to take our `feature` which has some custom steps in there and have Behat stub these out in the `features/bootstrap/UserAuthenticationContext.php`

```
vendor/bin/behat --suite user_auth --append-snippets
```

Now that file will be full of stubbed out tests that have the annotations to connect to your steps and then the a `PendingException` to let you know there is more work to do.

Keep in mind `Then I should see "You are logged in!"` in this example is a Mink related step so there is nothing else I need to do but `And I fill in the form with my username and password and submit the form` is custom so I need to fill in some code there.

```
    /**
     * @Given I fill in the form with my username and password and submit the form
     */
    public function iFillInTheFormWithMyUsernameAndPasswordAndSubmitTheForm()
    {
        $this->fillField('email', 'foo@foo.com');
        $this->fillField('password', env('EXAMPLE_USER_PASSWORD'));
        $this->pressButton('Login');
    }
```
 
Now our test is ready, I am talking to the dom and if I remove the `@javascript` from that test and run 


```
vendor/bin/behat --suite user_auth
```

![without js](https://dl.dropboxusercontent.com/s/ukjszl74q54n8kn/with_out_javascript.png?dl=0)

And add the tag back and run again 

![with javascript](https://dl.dropboxusercontent.com/s/3umwq03a8opfc1d/with_javascript.png?dl=0)


So you see it is 0m7.64s with `@javascript` and 0m2.09s without! So be careful to only use `@javascript` when really needed.


So now are test is passing locally let's get to CodeShip.

This will take 4 steps

Step one is to add a profile to `behat.yml` for CodeShip that looks like this


```
codeship_non_sauce:
    extensions:
        Laracasts\Behat:
            env_path: .env.codeship
        Behat\MinkExtension:
            base_url: http://127.0.0.1:8080
            default_session: laravel
            laravel: ~
            selenium2:
              wd_host: 'http://127.0.0.1:4444/wd/hub'
            browser_name: chrome
```

This leaves our `behat.yml` looking like this


```
default:
  suites:
    user_auth:
      contexts: [ UserAuthenticationContext ]
      filters: { tags: '@user_auth' }

  extensions:
    Laracasts\Behat:
        # env_path: .env.behat
    Behat\MinkExtension:
        base_url: https://codeship-behat.dev
        default_session: laravel
        laravel: ~
        selenium2:
          wd_host: "http://192.168.10.1:4444/wd/hub"
        browser_name: chrome


codeship_non_sauce:
    extensions:
        Laracasts\Behat:
            env_path: .env.codeship
        Behat\MinkExtension:
            base_url: http://127.0.0.1:8080
            default_session: laravel
            laravel: ~
            selenium2:
              wd_host: 'http://127.0.0.1:4444/wd/hub'
            browser_name: chrome
```

What we are doing is setting up a `.env.codeship` just for CodeShip settings and also setting a new `base_url` as well as using Selenium on `127.0.0.1`


Step 2 the `.env.codeship` file. Make that file in the root of your application and add to it this

```
APP_ENV=codeship
APP_KEY=base64:w0k4ZmTt89FApLdUaAsubNXH1eQcHR8vyat/ZvmqRso=
APP_DEBUG=true
APP_LOG_LEVEL=debug
APP_URL=http://127.0.0.1:8080

DB_HOST=localhost
DB_DATABASE=test
DB_PASSWORD=test
DB_CONNECTION=mysql
DB_USERNAME=root

QUEUE_DRIVER=sync

MAIL_DRIVER=log

EXAMPLE_USER_PASSWORD=quahf1Kaib2Ienei
```

At this point we set up our environment for the needed database and APP_URL as well as making sure QUEUE is in sync mode and Mail is just log.
 

Step three we need to update our `config/database.php` so it can take in these settings.

```

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

Replacing the default `mysql` settings with the above.


Finally Step four

----- scraps ---


From here I will run a command to get started with an example test. In this example I am going to use Laravel's Auth installation to get going as seen [here](https://laravel.com/docs/5.3/authentication#authentication-quickstart).

Of course there is no JavaScript here so it is not something I would do this on since JavaScript means slower tests. But for this example we will use it.  So after I follow those instructions and the base instructions for getting a site setup in Homestead I am ready. In this case the domain is https://codeship-behat.dev for my local build.




So now that I know what I want to test I am going to run

```

```


[Results with and without JavaScript]




## Links

[Windows and Selenium](https://alfrednutile.info/posts/181)




==== notes ====

So with this lesson I will go into an app that is already working and setup on CodeShip.

What I will do is setup a local testing working with Behat mainly focusing on UI testing. In our case I will focus on two types of UI tests one that needs JavaScript and one that does not. Both will work locally and then on CodeShip. One will be quick since it will use Symfony Browser Kit.

First we will just setup the basics for behat using this library https://github.com/laracasts/Behat-Laravel-Extension/tree/master

Once you have a basic install working with this we will then go on to add selenium

```
composer require "behat/mink-selenium2-driver":"^1.3"
```

Now let us setup Selenium on our host machine

Java for Mac 
https://support.apple.com/kb/DL1572?viewlocale=en_US&locale=en_US

https://www.npmjs.com/package/selenium-standalone

I use brew on the Mac for setting up Node. If you are using Ubuntu Linux keep an eye on the odd /usr/bin/nodejs installation that you then have to symlink to /usr/bin/node 

.env.codeship
database config file

Note the use of 

```
vendor/bin/behat --init
```

Setting up URL and Selenium in `behat.yml`


Now we have a working test

![](https://dl.dropboxusercontent.com/s/cmwwo8cdci30zn4/working_test.png?dl=0)

And in this case we will have a MySQL database to add to the example. Using the standard Auth setup found in the docs and homestead setup we will put this in place


Queue Driver Note sync
