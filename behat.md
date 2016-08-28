
# Behat testing using headless Chrome

This blog post is a look at using CodeShip to do Selenium and Headless Chrome testing which is key for interacting with JavaScript features on your site. But also I hope to show how to troubleshoot the rare moments when there is an issue on the CI but not on your local build by using CodeShips super helpful SSH feature and Saucelabs remote connections.  All code can be seen [here](https://github.com/alnutile/codeship-behat)

First we need to setup CodeShip to test our app with every git push. You can see this at a previous Laravel Centric post [here](https://blog.codeship.com/getting-started-laravel-codeship/)

At this point you should have things working and PHPUnit running. We are going to then add Behat to this.

Getting started really comes down to having this work on your local environment. In this post there will be a Host (Mac) and a Guest (Linux), thanks to [Homestead](https://laravel.com/docs/5.3/homestead). I will list at the end of this article have some links for Windows as well.

Here is look at this Host Guest workflow.

![behat drives selenium](https://dl.dropboxusercontent.com/s/gsl0ube5ybxtpaq/behat_drives.png?dl=0)


## Setting up your Local Environment

The first part of this setup is based on a Laravel Behat oriented library started by [Laracast's](https://laracasts.com/) creator Jeffrey Way [https://github.com/laracasts/Behat-Laravel-Extension](https://github.com/laracasts/Behat-Laravel-Extension).  After you follow the install steps there you will have a working version of Behat that integrates with your Laravel application in some nice ways including Migrations and Transactions hooks.


But even after the above I need to take it one step further I need to get Selenium setup both as a server and a Mink Extension.


```
composer require "behat/mink-selenium2-driver":"^1.3"
```

This will get you going for Mink and Selenium Driver.


Now for the Selenium server, this will be a bit harder but really not much. Remember this is on your Host, the above was in your Guest. Basically if you visit [here](https://www.npmjs.com/package/selenium-standalone) you can easily install Selenium on any Host OS. For me, and my Mac I use `brew` to set up NodeJS and from there I follow those 3 steps to get going. When I am done I have a terminal in the background just running Selenium.

![terminal selenium](https://dl.dropboxusercontent.com/s/9656ag3f2u2t9im/teminal.png?dl=0)

## Now for my first Behat test

At this point I need t make a `behat.yml` in the root of my application and fill it with the following.


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

We will build off this in a moment to add CodeShip. But for now I have one **suite** to get started `user_auth` and one **profile** `default`. I have the `base_url` for the local site `https://codeship-behat.dev` and for now it is using my applications `.env` file as seen on this line in the `# env_path: .env.behat`. This will change for the CodeShip profile.

Notice too I used the ip of my Host to talk to Selenium from inside my Guest `http://192.168.10.1:4444/wd/hub` one more thing that will change for the CodeShip Profile.


## Initialize Behat and Writing a Test

So now to prove all of this is working I start by running 

```
vendor/bin/behat --init
```

If you did not do this in the Behat-Laravel-Extension docs already. Either way you now have a `features` folder in the root of your application and in there a `bootstrap` folder and finally, in there, a `FeatureContext.php` file. 


Now to make my `feature` file `features/user_auth.feature` 

![feature file](https://dl.dropboxusercontent.com/s/cjtu1616oog86zr/feature_file.png?dl=0)


And in there I need to fill in the details 


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

Focusing on the `@happy_path` for now we tag it `@user_auth` so I know it is part of the Suite as seen in the `behat.yml` file above `filters: { tags: '@user_auth' }`. I could have used folders to organize my suites but chose tags for now.

Now if I run 


```
vendor/bin/behat --suite user_auth --init
```
 
I get a file `features/bootstrap/UserAuthenticationContext.php` that is pretty empty and needs to be told to extend `MinkContext`

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

Alright, so we now want to take our `feature` which has some custom steps in there and have Behat stub these out in the `features/bootstrap/UserAuthenticationContext.php`

```
vendor/bin/behat --suite user_auth --append-snippets
```

Now that file will be full of stubbed out functions that have the annotations to connect to your feature's steps that throw a `PendingException` to let you know there is more work to do.

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
 
Now our test is ready to run, I am talking to the DOM in the above steps, and if I remove the `@javascript` from that test and run 


```
vendor/bin/behat --suite user_auth
```

![without js](https://dl.dropboxusercontent.com/s/ukjszl74q54n8kn/with_out_javascript.png?dl=0)

We are not talking to Selenium but to [BrowerKit](http://symfony.com/doc/current/components/browser_kit.html) note how fast it it! 

And add the tag back and run again 

![with javascript](https://dl.dropboxusercontent.com/s/3umwq03a8opfc1d/with_javascript.png?dl=0)


>So be careful to only use `@javascript` when really needed

So you see it is 0m7.64s with `@javascript` and 0m2.09s without! So be careful to only use `@javascript` when really needed e.g when the page you are testing has JavaScript that you are focusing on. For example my Behat test can have two Scenarios and one has `@javascritp` and one does not. or the entire `feature` can be marked `@javascript` if needed.

```
@javascript
Feature: User Login Area
  User can log into the site
  As an anonymous user
  So that they can see secured parts of the site
```


So now the test is passing locally let's get to CodeShip.


## Four Steps to Setup CodeShip for Headless Chrome


### Step One: Add a profile to `behat.yml` 

For CodeShip that looks like this


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


### Step Two: The `.env.codeship` file. 

Make that file in the root of your application and add to it this

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

At this point we set up our environment for the needed database and APP_URL as well as making sure QUEUE is in sync mode and MAIL_DRIVER is just log.
 

### Step Three: Update our `config/database.php` 

Modify the `config/database.php` too look like this

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

Replacing the default `mysql` settings with the above will help us swap out the settings as needed for CodeShip and it's database work.


### Step Four: Scripting the setup of Selenium and Local Laravel Server

Now we need a script to setup CodeShip for testing.

When setting up a CodeShip Project you will have a "Setup Commands" window as seen below. In here I add `ci/setup.sh`

![ci setup](https://dl.dropboxusercontent.com/s/hmgo5onyrjfcz7t/ci_setup.png?dl=0) 

This is placed into a script for a couple of reasons. One being if I have to SSH into CodeShip to recreate the environment to see what a test is failing I can just do the same one command.

Next I make the folder `ci` and then in there `setup.sh`

This will look like 


```
#!/bin/sh

###
# This is thanks to CodeShip Docs
# But I wanted a newer version of Selenium
###

SELENIUM_VERSION=${SELENIUM_VERSION:="2.53.1"}
SELENIUM_PORT=${SELENIUM_PORT:="4444"}
SELENIUM_OPTIONS=${SELENIUM_OPTIONS:=""}
SELENIUM_WAIT_TIME=${SELENIUM_WAIT_TIME:="10"}

set -e

MINOR_VERSION=${SELENIUM_VERSION%.*}
CACHED_DOWNLOAD="${HOME}/cache/selenium-server-standalone-${SELENIUM_VERSION}.jar"

wget --continue --output-document "${CACHED_DOWNLOAD}" "http://selenium-release.storage.googleapis.com/${MINOR_VERSION}/selenium-server-standalone-${SELENIUM_VERSION}.jar"
java -jar "${CACHED_DOWNLOAD}" -port "${SELENIUM_PORT}" ${SELENIUM_OPTIONS} -log /tmp/sel.log 2>&1 &
sleep "${SELENIUM_WAIT_TIME}"
echo "Selenium ${SELENIUM_VERSION} is now ready to connect on port ${SELENIUM_PORT}..."


## Now we are ready to talk to Selenium let's start the Application server

cp .env.codeship .env
php artisan serve --port=8080 -n -q &
sleep 3
```

Basically I download Selenium standalone, then run it. After that I copy over the `.env.codeship` file to `.env` so the server will run with that one so it will line up with the `behat.yml` I am using.  If I did not do this there would be no `.env` since this is not part of Git, and I need to make sure the env settings are correct for CodeShip. Keep in mind I could have placed all of this into the CodeShip Environment UI Settings but as noted I find using this comes in handy when I want to SSH in and setup CodeShip to help troubleshoot an issue.

Now to make sure the CodeShip Test Pipeline is set.

![test pipeline](https://dl.dropboxusercontent.com/s/zql8ha5pee5ic0o/test_pipeline.png?dl=0)

Here I do a migration and seed, but typically I would leave the seed out and let each Behat Feature set up the state of the application the way it needs it.
In this case I will keep it simple.

And I am running Behat with the CodeShip profile `codeship_non_sauce`.

During the setup in `CodeShip` we need to put this `.env.codeship` file in place so that when we start the server we will.

That happens in the `ci/setup.sh` script

![copy](https://dl.dropboxusercontent.com/s/rtpcxa39phl0t7y/copy_env_codeship.png?dl=0)

Now we push to Github and...

![passing test](https://dl.dropboxusercontent.com/s/irbcolbmthd2ofu/codeship_psasing.png?dl=0)

## What happens when the tests does not pass on CodeShip?

I will give you a real example of a very difficult problem I wrestled with, I had this setting in my blade file


```
    <!-- Scripts -->
    <script src="{{ asset("/js/app.js", true) }}"></script>
```

So locally at `https://codeship-behat.dev` this worked great. But on CodeShip which runs at `http://localhost:8080` e.g. not using http**S** I kept getting a fail.  At that point I could put Saucelabs into the mix to watch the test run, but this was still not enough that is when SauceConnect comes into play and I can interact with the CodeShip server!

It was only here I could open the Chrome Console and really see the error message about not being able to connect to `https://localhost:8080`


more on this next article....


## Links

[Behat-Laravel-Extension](https://github.com/laracasts/Behat-Laravel-Extension)

[Windows and Selenium](https://alfrednutile.info/posts/181)

[SauceConnect](https://wiki.saucelabs.com/display/DOCS/Setting+Up+Sauce+Connect)

[Selenium Standalone Install](https://www.npmjs.com/package/selenium-standalone)

[CodeShip Selenium Install Script](https://github.com/codeship/scripts/blob/master/packages/selenium_server.sh) updated in this post
