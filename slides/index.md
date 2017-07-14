## Don't push rocks uphill
Deploying WordPress with Capistrano and Composer

Charles Fulton (@mackensen)
[https://mackensen.github.io/uphillrocks](https://mackensen.github.io/uphillrocks)

Note: This slide deck is available online if you'd like to follow along.



## Who am I, what am I doing here?


## Me

* Senior Web Applications Developer at Lafayette College
* Moodle developer since 2008
* WordPress administrator from 2013
* Frequent train rider


## Lafayette College

- Small liberal arts college in Easton, Pennsylvania
- Enrollment of ~ 2,400
- Engineering program
- Strong commitment to open source


## Organization

- Web Development is part of IT
- Web Development owns the platform
- Communications owns the content

Note: Web Development works in partnership with the Digital Communications group within Communications.


## Web Development owns all the things

WordPress /
Drupal /
Moodle /
Pydio /
MariaDB /
WeBWorK /
Redmine /
GitLab /
MediaWiki

Note: We have to be generalists given the broad nature of our responsibilities.



## WordPress at Lafayette


## 2009

- One WordPress MU installation for faculty, staff, and student blogging


## 2010

- Launched our redesigned public-facing site on WordPress 3.0 (now with multisite!)


## 2017

- Twelve multisite installations with over 3,000 individual sites
- 100+ themes and plugins

Note: obviously no one site has even half of these, but they all have to be managed.


## Today's focus

- Managing the deployed state of each installation


## Three's not a crowd

- Managing themes and plugins with Composer
- Deploying installations with Capistrano

Note: We use Composer for the themes and plugins, Capistrano for the deployment; deployment is in a couple minutes, to any environment. Rollback is an option. This might be considered off-label use. How did we get there, and why?


## Our considerations

- No installing themes and plugins from the interface <!-- .element: class="fragment" -->
- Developers have shell access to remotes <!-- .element: class="fragment" -->
- Separate development, staging, and production environments <!-- .element: class="fragment" -->
- Generalized solution <!-- .element: class="fragment" -->

Note: our servers are hardened with SELinux; Apache cannot write to the webroot. We are, however, able to do our own deployments. We have replicas of our installations in three separate environments, which are in different network segments. Finally, as I said, previously, we're generalists. We'd like a deployment strategy that we can adapt to Moodle, Drupal, and custom projects.


## ...git is not joining us today

- There's a slide at the end with awesome Git resources

Note: This isn't a git talk or even a development talk.



## In the beginning...

- We had no version control
- We moved ZIP files around manually
- Our themes and plugins were at inconsistent versions
- Our confidence level: shaky

Note: In 2012 it took us 6-7 hours to conduct the deployment of a new WordPress release. That's not accounting for overhead.


## Our directory structure

```bash
.htaccess
wp-admin/
wp-config.php
wp-content/
	blogs.dir/
	cache/
	plugins/
		foo/
	themes/
		bar/
	uploads/
wp-includes/
```

Note: This is pretty standard, but a from a version control perspective a little annoying. You have core WordPress and your themes and plugins all mixed in, and your uploads inside the main document root.



## Idea!

An installation is a software project


## Projects have dependencies

- Core WordPress <!-- .element: class="fragment" -->
- Themes <!-- .element: class="fragment" -->
- Plugins <!-- .element: class="fragment" -->
- Random bits of junk <!-- .element: class="fragment" -->


## Possible sources of code

- WordPress.org
- Public version control systems
- Private version control systems
- Random sites on the internet

Note: That's a diverse ecosystem.


## &nbsp;
<!-- .slide: data-background-image="resources/electronic-waste.jpg" -->

Note: Artist's conception of a disorganized collection of WordPress themes and plugins from various sources. I'm not sure what core WordPress is in this metaphor.


## &nbsp;
<!-- .slide: data-background-image="resources/rail-yard.jpg" -->

Note:
This image represents an ideal state; all the packages are organized. How do we get all these different packages flowing?


## Manage the project

- Single repository for the project
- Track versions of the components
- Don't just download and commit everything

Note: Just committing everything encourages bad habits, like core and plugin hacking. Changes and change management should be purposeful.



## Pushing rocks uphill

Note: I'll start with a cautionary tale about how we first solved this problem, and why you shouldn't do that.


## Git  Submodules
<!-- .slide: data-background-image="resources/turtles-all-the-way-down.jpg" -->

It's repositories all the way down.


## A repository for every module

```bash
[submodule "wp-content/plugins/wordpress-mobile-pack"]
	path = wp-content/plugins/wordpress-mobile-pack
	url = git@git.lafayette.edu:wordpress/wordpress-mobile-pack
[submodule "wp-content/plugins/multisite-plugin-manager"]
	path = wp-content/plugins/multisite-plugin-manager
	url = git@git.lafayette.edu:wordpress/multisite-plugin-manager
[submodule "wp-content/plugins/akismet"]
	path = wp-content/plugins/akismet
	url = git@git.lafayette.edu:wordpress/akismet
[submodule "wp-content/plugins/wpmuldap"]
	path = wp-content/plugins/wpmuldap
	url = git@git.lafayette.edu:wordpress/wpmuldap
```

Note: Git stores which hash is checked out for each module.


## Structure

```bash
.gitmodules
wp-admin/
wp-content/
	plugins/
		foo/
	themes/
		bar/
wp-includes/
```

Note: The code is visible within the checked out repositories, but only the hash is committed. The upload directories and configuration files sit on the remote, untracked.


## Getting things into Git

- Private projects: already in Git
- Premium projects from the internet: manual download
- WordPress.org projects: yeah, about that...

Note: Gravity Forms is a good example of a popular plugin that isn't distributed via a version control system. You can't avoid the overhead in this case.


## WordPress.org

- Download as a ZIP
- Clone the Subversion repository
- ~~Git~~


## Git all the things!

We built a WordPress.org Subversion-to-private git repository pipeline.


## &nbsp;
[Narrator]
They shouldn't have done that.

Note:
It seemed like a good idea at the time.


## Rocks

![WordPress.org code pipeline](resources/pushing-rocks.jpg)

17th century print: _WordPress.org theme vel plugin ex git submodule_

Note: We did this for two years. The pipeline involved using the svn2git ruby gem to convert each WordPress.org subversion repository, then push the results into a self-hosted git repository. The initial clone took 3-4 hours, assuming it didn't fail because the SVN repository was in a non-standard configuration. That scenario was common with older WordPress themes. Also, there was a non-zero chance you'd get blocked for scraping.


## Deployment

- Clone the project repository to the web server
- Lots of magic encapsulated in shell scripts


## Assessment

- Overhead for updating themes and plugins: High
- Deployment speed: Fast
- Our confidence leve: Moderate


## Goals / lessons learned

- Don't do that
- Find a solution that works with the existing situation



## Overlaying dependency management


## Leave the rocks in the field
<!-- .slide: data-background-image="resources/field-of-rocks.jpg" -->


## Let's try a light touch

- Bring the dependency management to the code
- Don't make work for yourself


## A manifest, not a statement

```json
{
    "name": "lafayette/a-wordpress-site",
    "description": "Our awesome WordPress installation",
    "require": {
        "wordpress/akismet": "~3.3",
        "wordpress/wordpress": "4.7.*",
    }
}
```

How do we get there?


## Composer

- PHP package manager, like npm or bundler
- Use a JSON file to capture metadata
- https://getcomposer.org/


## &nbsp;

```json
{
    "name": "outlandish/wpackagist",
    "description": "Install and manage WordPress plugins with Composer",
    "require": {
        "php": ">=5.3.2",
        "composer/composer": "1.3.*",
        "silex/silex": "~1.1",
        "twig/twig": ">=1.8,<2.0-dev",
        "symfony/console": "*",
        "symfony/filesystem":"*",
        "symfony/twig-bridge": "~2.3",
        "symfony/form": "~2.3",
        "symfony/security-csrf": "~2.3",
        "symfony/locale": "~2.3",
        "symfony/config": "~2.3",
        "symfony/translation": "~2.3",
        "pagerfanta/pagerfanta": "dev-master",
        "franmomu/silex-pagerfanta-provider": "dev-master",
        "doctrine/dbal": "2.5.*",
        "knplabs/console-service-provider": "1.0",
        "rarst/wporg-client": "dev-master",
        "guzzlehttp/guzzle-services": "1.0.*"
    },
    "bin-dir": "bin",
    "autoload": {
        "psr-4": {
            "Outlandish\\Wpackagist\\": "src/"
        }
    }
}
```

Note: Each key/value pair is a separate project, with a defined version constraint. In this example these are all projects on Packagist, the central Composer repository.


## Packagist

- Centralized Composer repository
- Available by default to all composer projects
- [https://packagist.org/](https://packagist.org/)

Note: With Packagist you register a project and tell it where it can find the code. It matches up the key/value pair in any composer.json file it finds with the tags in the version control system (usually git).


## Problem

- Your stuff isn't in Packagist
- You (probably) don't want your private projects there


## You get a repository, and you...

```json
{
  "name": "yourcompany/yourproject",
  "description": "Your sample project",
  "repositories": [
    {
      "type": "vcs",
      "url": "https://github.com/yourcompany/someotherproject"
    }
  ]
}
```

Note: You probably don't want to register your private themes and plugins on Packagist. Happily, Composer lets you define any version control system as a repository, so long as there's a `composer.json` file present.


## A fistful of repositories

```json
{
	"name": "yourcompany/yourproject",
	"description": "Your sample project",
	"repositories": [
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-about-us"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-academics"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-admissions"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-base"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-brandywine"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-campus-life"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-hermione"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-home"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-map"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-media-vault"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-news"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-victoire"
		},
	]
}
```

Looks like bureaucracy.

Note: You might have a dozen or more private plugins, with some used on multiple installations. Maybe the location changes?


## Satis
- Open-source flatfile Composer repository
- Directs traffic to your repositories
- [https://github.com/composer/satis](https://github.com/composer/satis)


## This goes in Satis

```json
{
  "name": "Your school's satis repository",
  "homepage": "https://satis.yourschool.edu",
  "repositories": [
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-about-us"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-academics"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-admissions"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-base"
		},
		{
			"type": "vcs",
			"url": "git@git.lafayette.edu:wordpress/marquis-brandywine"
		},
    ]
}
```


## ...that goes in your project

```json
{
  "name": "yourcompany/yourproject",
  "description": "Your sample project",
  "repositories": [
    {
      "type": "composer",
      "url": "https://satis.yourschool.edu"
    }
  ]
}
```


## Meanwhile, WordPress.org

- Themes and plugins on WordPress.org do not have `composer.json` files
- No, adding `composer.json` files to the svn-to-git sync is not a solution
- Really, it's not


## WordPress Packagist
- Composer mirror of the WordPress.org repositories
- Maintained by [Outlandish](https://outlandish.com/)
- [https://wpackagist.org/](https://wpackagist.org/)


## WordPress Packagist links to ZIP downloads

```json
"wpackagist-plugin/akismet": "3.3.*",
```

==

```json
"url":"https://downloads.wordpress.org/plugin/akismet.3.3.2.zip",
```


## Your site on Composer
```json
{
  "type": "project",
  "repositories": [

    {
      "type": "composer",
      "url": "https://wpackagist.org"
    },
    {
      "type": "composer",
      "url": "https://satis.yourschool.edu"
    }
  ],
  "require": {
    "lafayette/lafayette-utilities": "^1.5",
    "lafayette/wordpress-maintenance": "2.0.0",
    "lafayette/wpmuldap": "4.0.2.*",
    "wpackagist-plugin/akismet": "3.3.*",
    "wpackagist-plugin/multisite-plugin-manager": "3.1.*",
    "wpackagist-plugin/rewrite-rules-inspector": "1.2.1",
    "wpackagist-plugin/site-creation-utilities": "1.0.*",
    "wpackagist-plugin/ssl-insecure-content-fixer": "2.2.*",
    "wpackagist-plugin/wordpress-importer": "0.6.*",
    "wpackagist-plugin/wp-cassify": "^2.0",
    "lafayette/marquis-base": "^2.3",
    "lafayette/library-post-types": "0.2.*",
    "wordpress/advanced-custom-fields-pro": "^5.5",
    "lafayette/marquis-hermione": "^1.1",
    "wordpress/gravityforms": "2.1.3.5",
    "lafayette/ezproxy-converter": "^1.0",
  },
}
```


## Wait, what about WordPress core?

- Is that important?


## Submodules not harmful this one time

```bash
[submodule "public"]
	path = public
	url = https://github.com/WordPress/WordPress
```

Note: Composer doesn't work well with core WordPress if you're keeping the traditional directory structure.


## The project so far
```bash
composer.json
composer.lock
public
```



## Deployment


## Time to roll rocks downhill!
<!-- .slide: data-background-image="resources/falling-rocks.jpg" -->


## Capistrano

- Ruby-based application which manages deployed state
- Tells a story about your deployment
- [http://capistranorb.com/](http://capistranorb.com/)


## Your project on Capistrano

```bash
Capfile
composer.json
composer.lock
config/
	deploy.rb
	deploy/
		staging.rb
		production.rb
Gemfile
Gemfile.lock
public
```


## It's all local trouble

- Install gems on the local client
- Execute commands on the remote over SSH

Note: There's no server-side configuration, and no local-side configuration external to the project repository.


## Calm before the storm

```bash
releases/
repo/
revisions.log
shared/
	public/
		.htaccess
		wp-config.php
			wp-content/
				uploads/
```

Note: With Capistrano you keep your configuration and uploads outside the deployment root.


## Symlinks all the way down

```bash
current -> /var/www/sites/releases/20170424135711
releases/
	20170301101452
	20170303204702
	20170424135711
		public/
			.htaccess -> /var/www/sites/shared/public/.htaccess
			wp-config.php -> /var/www/sites/shared/public/wp-config.php
			wp-content/
				uploads -> /var/www/sites/shared/public/wp-content/uploads
repo/
revisions.log
shared/
	public/
		.htaccess
		wp-config.php
			wp-content/
				uploads/
```

Note: On deployment, the configuration and uploads are symlinked in, and the "current" symlink points at the last good release.


## Tasks: there's a plugin for that

- [Git submodule capistrano module](https://github.com/ekho/capistrano-git-with-submodules)
- [Composer capistrano module](https://github.com/capistrano/composer)
- [Npm capistrano module](https://github.com/capistrano/npm)

Note: in config/deploy.rb you tell Capistrano what to do when it deploys. Init submodules, install Composer dependencies, do arbitrary things with node, run shell commands.


## One node, two node

```ruby
server 'node0.foo.edu', user: fetch(:user), roles: %w{app web}
server 'node1.foo.edu', user: fetch(:user), roles: %w{web}
set :deploy_to, '/var/www/sites'
```

Note: environment files tell Capistrano where to deploy things, and let you vary tasks in a multi-node environment. We fell right in to a solution for high availability just when we needed it.


## Deployment as documentation

- WordPress in the submodule
- Themes and plugins in `composer.json` file
- Nodes in the environment files
- Tasks in the config files

Note: it would be cruel, but we could hand off a project repository to a new developer and tell them to redeploy it.



## The way we deploy now


## Upgrading a plugin

```bash
$ composer update wpackagist-plugin/powerpress
Loading composer repositories with package information
Updating dependencies (including require-dev)
  - Removing wpackagist-plugin/powerpress (7.0.3)
  - Installing wpackagist-plugin/powerpress (7.0.4)
    Downloading: 100%

Writing lock file
Generating autoload files
$ git add composer.* && git commit -m "Update powerpress"
[master 19a7cb0] Update powerpress
 1 file changed, 5 insertions(+), 5 deletions(-)
```


## Deploying an entire site

```bash
$ bundle exec cap production deploy
```


## Running database upgrades

```ruby
namespace :deploy do
	# Run an upgrade
    task :upgrade do
        on roles(:app) do |host|
            execute "cd #{fetch(:deploy_to)}/current/public && wp core update-db --network"
        end
    end
end
```

```bash
$ bundle exec cap production deploy:upgrade
```


## Minifying theme files on deployment

```ruby
namespace :deploy do
    # Minify base theme
    before :updated, :grunt do
        on roles(:web) do |host|
            execute "cd #{fetch(:release_path)}/public/wp-content/themes/marquis-base && grunt deploy --gruntfile Gruntfile-deploy.js"
        end
    end
```

Note: I mentioned node. We had a problem with managing minified CSS and Javascript files for our base theme in git. We moved that to a Grunt task, to be executed on every deployment.


## Assessment

- Overhead for updating themes and plugins: Low
- Deployment speed: Fast (and frequent)
- Our confidence level: High

Note: we're deploying all the time, any time.



## Wrap-up


## Review

- Manage versions and state, not code
- Don't push rocks uphill


## Comparison of methods

| Method              | Overhead | Deployment | Confidence |
| ------------------- | -------- | ---------- | ---------- |
| Zip files           | High     | Glacial    | Shaky      |
| Submodules          | High     | Fast       | Moderate   |
| Composer-Capistrano | Low      | Fast       | High       |


## Extended edition of this talk

- Submodules: [Don't push rocks uphill](http://sites.lafayette.edu/fultonc/2017/07/07/dont-push-rocks-uphill/)
- Composer: [Overlaying dependency management](http://sites.lafayette.edu/fultonc/2017/07/07/overlaying-dependency-management/)
- Capistrano: [Rolling rocks downhill](http://sites.lafayette.edu/fultonc/2017/07/10/rolling-rocks-downhill/)


## Git/developer resources

- [Annette Liskey: Git in the Van](https://www.slideshare.net/AnnetteLiskey/git-in-the-van-highedweb-2013)
- [Git Tutorial on Youtube](https://www.youtube.com/playlist?list=PLwrxhoDq6Kivqmc3jbqZhQnTuuv8odAdy)
- [Tracy Rotton: The Modern WordPress Developer's Toolbox](https://videopress.com/v/rPkUrRWK)
- [An introduction to version control - Beanstalk Guides](http://guides.beanstalkapp.com/version-control/intro-to-version-control.html)
- [Raisa Yang: From Newbie to Front End Developer](http://raiscake.me/talks/newbie-to-frontend)
- [A Visual Git Reference](http://marklodato.github.io/visual-git-guide/index-en.html)


## Thank you

- Please leave feedback at [https://2017.wpcampus.org/session-survey/415](https://2017.wpcampus.org/session-survey/415)
- Find me on Twitter at @mackensen
- This presentation: [https://mackensen.github.io/uphillrocks](https://mackensen.github.io/uphillrocks)
