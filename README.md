# freebsd-puma

A robust init script for running puma on FreeBSD. Based entirely on the excellent [freebsd-unicorn] script by [Caleb]. All I have done is to replace unicorn specifics with puma specifics, and update this readme.

This rc script works with either RVM and RBENV installed in your deploy user's directory, or with a globally installed ruby.

Simply place the `puma` script in your `/usr/local/etc/rc.d` directory, modify it if necessary, and configure your application via variables in `/etc/rc.conf`

This has been tested on **FreeBSD 10.4**


## Make sure puma starts after your database launches!

The only thing you might need to configure in the rc script is to change the `REQUIRE` line to specify your database (I use PostreSQL so that's what's in the repo)

For example, if you were using MySQL, you would change

    # REQUIRE: LOGIN postgresql

to

    # REQUIRE: LOGIN mysql-server

You might need to add other services to this list if your Rails application requires them.

Depending on your ruby package, you may need to change procname. This is used to match the process name to the pid, to determine whether Puma is running. If you installed ruby 2.4 on FreeBSD, you would change

    # procname="ruby"
    
to

    # procname="ruby24"


## Quick Setup

To get up and running quickly, adjust the `REQUIRE` line like above, and add edit your `/etc/rc.conf`:

For Capistrano or Capistrano-like directory layouts:

    puma_enable="YES"

    # this is the path to where your application is deployed via Capistrano
    # (the parent directory of the `current` directory)
    puma_directory="/u/application"


For Non-Capistrano-like layouts:

    puma_enable="YES"
    puma_command="/u/application/bin/puma_rails"
    puma_rackup="/u/application/config.ru"
    puma_pidfile="/u/application/tmp/pids/puma.pid"
    puma_old_pidfile="/u/application/tmp/pids/puma.oldbin"
    puma_listen="/u/application/tmp/sockets/puma.sock"
    puma_config="/u/application/config/puma.rb"
    puma_chdir="/u/application"
    puma_user="deploy"

    # Uncomment this if using a different RAILS_ENV/RACK_ENV than production
    #puma_env="production"


## Starting/Stopping/Restarting and Upgrading puma

You can now start puma like any other FreeBSD service:

    /usr/local/etc/rc.d/puma start

There's also a handy `show` command to look at your final puma configuration:

    /usr/local/etc/rc.d/puma show

You can do an old-fashioned restart, or a [zero-downtime upgrade][puma-0-downtime] (untested) with these commands:

    /usr/local/etc/rc.d/puma restart
    /usr/local/etc/rc.d/puma upgrade

And when you're done riding pumas, you can shut it down

    /usr/local/etc/rc.d/puma stop


## `/etc/rc.conf` Details


### Using a Capistrano directory layout

The rc script does as much as possible to help you out. If you are using Capistrano, or a Capistrano-like directory structure, then you can just specify the directory of your application (the parent directory of `current`):

    puma_enable="YES"
    puma_directory="/u/application"

This infers all sorts of information about your app (you can always run `/usr/local/etc/rc.d/puma show` to see what your configuration is. **Note** the variable names listed here are without the leading `puma_` prefix that you would need to specify in `/etc/rc.conf`):

    #
    # puma Configuration for application
    #

    command:        /u/application/current/bin/puma_rails
    command_args:   /u/application/current/config.ru
    rackup:         /u/application/current/config.ru
    pidfile:        /u/application/shared/tmp/pids/puma.pid
    old_pidfile:    /u/application/shared/tmp/pids/puma.pid.oldbin
    listen:         
    config:         /u/application/current/config/puma.conf.rb
    init_config:    
    bundle_gemfile: /u/application/current/Gemfile
    chdir:          /u/application/current
    user:           deploy
    nice:           
    env:            production
    flags:          -E production -c /u/application/current/config/puma.conf.rb -D

    start_command:

    su -l deploy -c "export BUNDLE_GEMFILE=/u/application/current/Gemfile && cd /u/application && /u/application/current/bin/puma_rails -E production -c /u/application/current/config/puma.conf.rb -D /u/application/current/config.ru"

Let's look at these settings one by one:

`command`: By default, it uses the `current/bin/puma_rails` [bundler binstub][binstub] located in your project to ensure your gems are loaded. `command` comes from FreeBSD's `rc.subr` init system functions.

`command_args`: This is the standard FreeBSD's `rc.subr` variable that holds the arguments to the above `command`. The `rackup` setting is prepended to the beginning of this setting. Typically you don't need to set this.

`rackup`: This is the `rackup` file that puma uses. By default this is set to `current/config.ru` for capistrano projects.

`pidfile`: This is also part of FreeBSD's `rc.subr` system. This is where the built in functions will look for the pid of the process. By default, this rc script looks in the `shared/tmp/pids/puma.pid` file.

`old_pidfile`: This is the pidfile used by puma to perform zero-downtime upgrades. [Procedure to replace a running puma executable][puma-0-downtime]. This rc script uses puma's default convention of appending `.oldbin` to the end of the `pidfile`

`listen`: This is the port or socket for puma to listen on. This rc script assumes that you will specify it in your project's puma config file (see the next variable).

e.g.

    listen "#{app}/shared/sockets/puma.sock", :backlog => 64

`config`: This is the path to puma's config file where puma will find it's settings. By default this rc script looks for a file called `current/conf/puma.conf.rb`

`init_config`: This is a shell script file that is included in the environment before puma is executed. In this file you can include `export VAR=value` statements to pass environment variables into your rails app. By default, this init script looks for a file called `current/.env` and uses that. If that file doesn't exist, this rc script will skip this functionality (as seen in the above example).

This could be used in conjunction with [dotenv][dotenv] in development since dotenv accepts lines beginning with `export`


`bundle_gemfile`: This is the path to the `Gemfile` of your project. This rc script sets the `BUNDLE_GEMFILE` environment variable to this value. By default it looks to `current/Gemfile`. This is required so that puma uses the most current `Gemfile` (rather than the one in the specific deployment directory) when an upgrade is performed. This is a safeguard, you should really put this in your puma.conf.rb:

    before_exec do |server|
      ENV["BUNDLE_GEMFILE"] = "/path/to/app/current/Gemfile"
    end

`chdir`: This is the directory we `cd` into before running puma. By default it's the currently deployed version of your application "`current/`"

`user`: This is the user that puma will be run as. By default we do like [Passenger][passenger] and use the owner of the `puma_directory`.

`nice`: The `nice` level to run puma at. Usually you'll leave this alone.

`flags`: This is a variable defined by FreeBSD's `/etc/rc.subr` init system, and contains the flags passed to the command (puma in this case) when run. This variable is built up from the variables above, but you could manually specify `puma_flags` in your `/etc/rc.conf` to override them.

`start_command`: Here you can see the full command that will be run when you start this service. It's a beaut' isn't it?

You can override any of these parameter in your `/etc/rc.conf` by simply specifying the variables like you see below (you can pick and choose which to override).


### Using a custom directory layout

Using your own layout is easy, you can just leave the `puma_directory` variable out of your `/etc/rc.conf` and specify all of the above variables manually. Here's a list of those variables for your convenience:

    puma_command:      The path to the puma command
    puma_command_args: The non-flag arguments passed to the above command. This is usually the path to the Rack config.ru file, and the rackup parameter is prepended to the beginning of this setting. Typically you do not need to set this.
    puma_rackup:       This is the path to the Rack config file (typically called `config.ru`). This setting is prepended to the `command_args` parameter
    puma_pidfile:      The path where puma will put its pid file
    puma_old_pidfile:  The path where puma will put its `old` pid file (usually the same as the sbove with `.oldbin` appended)
    puma_listen:       The path to the socket or port to listen on. You can leave this blank to specify where to listen in the puma config
    puma_config:       The path to the puma config file
    puma_chdir:        The path where this script will `cd` to before starting puma
    puma_user:         The user to run puma as
    puma_nice:         The `nice` level to run puma as. Leave blank to run un-niced
    puma_env:          The RAILS_ENV (or RACK_ENV) to run your application as. (default: production)
    puma_flags:        The flags passed in to puma when starting (not counting the puma_command_args specified above). Override this for complete control of how to start puma.


### Deploying multiple applications

This is all find and dandy, but some of you might have multiple applications running on the same server (even if it's just a staging and production version of your app).

This rc script can work with multiple applications. It works similarly to how postgresql's rc script works on FreeBSD.

You simply specify your profiles in your `/etc/rc.conf` with the `puma_profiles` variable. This is a space separated list of application names.

Then you can customize each application by specifying variables in this form:

    puma_<application-name>_variable=VALUE

Here's a simple example (I can leave the _env variable out of the production declaration since it's the default value)

    puma_enable="YES"
    puma_profiles="application_staging application_production"

    puma_application_staging_enable="YES"
    puma_application_staging_directory="/u/application_staging"
    puma_application_staging_env="staging"

    puma_application_production_enable="YES"
    puma_application_production_directory="/u/application_production"

You can use the simplified Capistrano `directory`-based configuration like above, or you can specify all of the variable's separately, for a fully custom setup

### Customizing the script

If you want to customize the default, calculated values you want to look in the `_setup_directory()` function. This is what is called when you specify that you want to use a Capistrano-like directory layout by specifying `puma_directory` or `puma_<application-name>_directory` in your `/etc/rc.conf`.

If you use a different deployment strategy than Capistrano, you could adjust the default values to work with your system.




[passenger]: https://www.phusionpassenger.com
[puma-0-downtime]: https://github.com/puma/puma/blob/master/docs/restart.md
[dotenv]: https://github.com/bkeepers/dotenv
[binstub]: https://github.com/sstephenson/rbenv/wiki/Understanding-binstubs
[freebsd-unicorn]: https://github.com/caleb/freebsd-unicorn
[Caleb]: https://github.com/caleb
