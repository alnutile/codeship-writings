# Interacting with your CodeShip Build

In this article I will continue from my previous article [Laravel and Behat Using Selenium and Headless Chrome](https://blog.codeship.com/laravel-and-behat-using-selenium-and-headless-chrome/) and explain what you can do when troubleshooting a build that has failed.  

CodeShip's SSH feature is amazing and comes in handy for troubleshooting. In some cases, like the example I will use in this article, I needed a bit more than headless Chrome to figure out what was wrong with my build. So using SSH and SauceLabs Connect I will show how I interacted with my running PHP server on CodeShip!

The example I left off at in that previous article was a real example where I had everything passing locally but when I ran the build on CodeShip it failed. It was a test that required the UI since there was JavaScript involved. The issue came down to one word `true`.

```
<script src="{{ asset("/js/app.js", true) }}"></script>
```

On my local build I always run my environment and my tests under `https` but on CodeShip I have a habit of running them `http` since I start the PHP server with this line.

```
php artisan serve --port=8080 -n -q &
```

So the addition of the word `true` in the `asset` method forced it to serve the assets under a secure URL which will, in Chrome, cause an error if the site is running in a non-secure url. 

None of my efforts before this worked, screenshots on fail, dumping the HTML with `And print last response`, etc. What I really needed was to see the console tool panel my browser offers, in this case Chrome. But I could not do this easily with headless Chrome.

That is when I setup [SauceLabs Connect](https://wiki.saucelabs.com/display/DOCS/Setting+Up+Sauce+Connect) to interact my build! 

Let me explain how this works step by step.

## SSH Into the Failing Build

On the right hand side of a build in CodeShip is an option to SSH for debugging.

![ssh](https://dl.dropboxusercontent.com/s/ijstbyetkfhui1w/ssh_option.png?dl=0)

Once you log in you just need to set things up again since it is not the same state as when your build just failed. For me this means running these few lines to get things going.

```
cd clone
export SAUCELABS_NAME=YOUR_NAME_FOR_SAUCELABS
export SAUCELABS_KEY=YOUR_KEY_FOR_SAUCELABS
phpenv local 5.6
composer install
cp .env.codeship .env
php artisan serve --port=8080 -n -q &
sleep 3
```

>@NOTE ideally this is just a script in my `ci` folder.

So now I have a PHP site running on port `8080` for me to interact with from SauceLabs. 

As far as the SauceLabs info you can get these from logging in to [https://saucelabs.com](https://saucelabs.com). It will be the username you see when you click "My Account"

![username](https://dl.dropboxusercontent.com/s/4rrmwgxsoiw21g2/user_name.png?dl=0)

and the KEY when you scroll down and unlock it.

![key](https://dl.dropboxusercontent.com/s/ykf1emd7ndnykkh/key.png?dl=0)

We will use that next.

## Setting up a SauceLabs tunnel.

This is a really neat feature of Linux and Unix like systems that SauceLabs has put to good use. If you go their [Wiki](https://wiki.saucelabs.com/display/DOCS/Setting+Up+Sauce+Connect) you will see a script you can download. Typically I store this in a folder for my miscellaneous build scripts `ci`. In their I keep a copy of this file to I can more easily just make the connection. The bash script looks like this.

```
#!/bin/sh
ci/bin/sc -u ${SAUCELABS_NAME} -k ${SAUCELABS_KEY} --shared-tunnel -i ci &
sleep 5
```

In a moment you will have tunnel to your SauceLabs account. You will know it is connected because when you log into your account you will see this "Tunnels 1 Active" in Green.

![green](https://dl.dropboxusercontent.com/s/ta0p269uztr3n48/tunnel_label.png?dl=0)

Now on the SauceLabs Dashboard start a "Manual Tests"

![manual test](https://dl.dropboxusercontent.com/s/zab4i3jh76d5xao/manu_test.png?dl=0)

This will pop open a new screen. Here I setup the remaining info including the websites url and port.

![](https://dl.dropboxusercontent.com/s/lqt1r5jicd3uep6/manual_test.png?dl=0)

In a few moments I will get a full desktop (thanks to [VNC](https://en.wikipedia.org/wiki/Virtual_Network_Computing)) that they are using the browser!

![](https://dl.dropboxusercontent.com/s/oy08xaocwzht46v/full_screen.png?dl=0)

And finally we are ready, our server is running, the tunnel is setup so let's interact with the site.

As normal would when working locally, I can click around the desktop **in the browser** and open up Developer Tools in Chrome and figure out what is wrong.

![](https://dl.dropboxusercontent.com/s/jxj1ply1k47i4qj/manual_access.png?dl=0)
 
No more guessing, no more headless Chrome troubleshooting!

So as we experienced in the first article Headless Chrome is great and fast, even for local work. But when things go wrong it is best to open up a regular Chrome session and dig in as normal to see what is going on.  And we see tools like SauceLabs and SauceLabs Connect with CodeShip make this amazingly easy to do.



## Side Note: How about Running Tests on SauceLabs

The above is an great way to troubleshoot your build. But sometimes I might want to run my build via SauceLabs instead of a local Selenium. This can also be good for troubleshooting but it really is key for testing against the many different browsers they offer and even mobile.  And with the above section you are half way there.

For this to work I need to have a new section in my `behat.yml` file. This will be a new `profile` called `saucelabs`

```
saucelabs:
  extensions:
    Laracasts\Behat:
      env_path: .env.codeship
    Behat\MinkExtension:
      base_url: http://127.0.0.1:8080
      default_session: laravel
      selenium2:
        browser: chrome
        wd_host: 'ACCOUNT_NAME:ACCOUNT_KEY@ondemand.saucelabs.com/wd/hub'
        capabilities:
          platform: 'Windows 8.1'
          browser: chrome
          version: '52'

```

>@NOTE you really need to save your settings above `ACCOUNT_NAME:ACCOUNT_KEY`
>if this is a public repo that is too risky.
>You can pass it is using `BEHAT_PARAMS` [^1] but I find 
>it hard to really setup for multiple profiles.

Then I run the SauceLabs Connect `sc` BUT slightly different than above 

```
#!/bin/sh
ci/bin/sc -u ${SAUCELABS_NAME} -k ${SAUCELABS_KEY} --shared-tunnel &
```

I do not define the tunnel name `-i ci` so that all requests default now through the tunnel.

This means when I finally run 

```
behat --profile=saucelabs
```

The local behat will talk to the remote Selenium server and that will tunnel back to my localhost webserver via this tunnel/port forwarding that is going on.

![](https://dl.dropboxusercontent.com/s/okgwv47hs208myq/watching_test.png?dl=0)

## Next Time...

This series will continue as I cover how to run parallel tests on CodeShip. I will cut down my PHPUnit and Behat tests in half!


## Links
[https://wiki.saucelabs.com/display/DOCS/Test+Configuration+Options](https://wiki.saucelabs.com/display/DOCS/Test+Configuration+Options)

Example Code [here](https://github.com/alnutile/codeship-behat)

## Footnotes

[^1]: Behat Params for passing this in see above code snippet

```
export BEHAT_PARAMS="{
    \"extensions\": {
        \"Behat\\\MinkExtension\": {
            \"selenium2\": {
                \"wd_host\": \"http://${SAUCELABS_NAME}:${SAUCELABS_KEY}@localhost:4445/wd/hub\"
              }
          }
      }
  }" && vendor/bin/behat --profile=saucelabs
```

