---
title: Installing Redis on WordPress
description: A walkthrough of how to enable WP-Redis on your Pantheon WordPress site.
tags: [cacheapp, addons]
categories: [wordpress]
---
Redis is an open-source, networked, in-memory, key-value data store that can be used as a drop-in caching backend for your Drupal or WordPress website.

<div class="enablement">
  <h4 class="info" markdown="1">[Agency DevOps Training](https://pantheon.io/agencies/learn-pantheon?docs){.external}</h4>
  <p>Learn industry best practices for WordPress caching, how to take advantage of them on the platform, and troubleshooting common issues with help from the experts at Pantheon.</p>
</div>

For Drupal, see [Installing Redis on Drupal](/docs/drupal-redis/).

## Benefits of Redis
Most website frameworks like Drupal and WordPress use the database to cache internal application "objects" which can be expensive to generate (menu trees, filter results, etc.), and to keep cached page content. Since the database also handles many queries for normal page requests, it is the most common bottleneck causing increase load-times.

Redis provides an alternative caching backend, taking that work off the database, which is vital for scaling to a larger number of logged-in users. It also provides a number of other nice features for developers looking to use it to manage queues, or do custom caching of their own.

Currently, all plans except for Personal can use Redis. Redis is available to Sandbox plans for developmental purposes, but Redis will not be available going live on a Personal plan.



 | Plans        | Supported
 | -------------|:-------------:|
 | Sandbox      | **Yes**       |
 | Personal     | No            |
 | Professional | **Yes**       |
 | Business     | **Yes**       |
 | Elite        | **Yes**       |

---


## Enable Redis on the Pantheon Site Dashboard
First enable Redis from your Pantheon Site Dashboard by going to **Settings** > **Add Ons** > **Add**. It may take a couple minutes for the Redis server to come online.
## Install Drop-in Plugin
[WP Redis](https://wordpress.org/plugins/wp-redis/) is loaded via a drop-in file, so there's no need to activate it on your WordPress sites. The following installation methods are supported:
### Install via Symlink Drop-in
This method will store object cache values persistently in Redis while preserving the standard procedures for applying plugin updates.

1. Install the [WP Redis](https://wordpress.org/plugins/wp-redis/) plugin via SFTP or Git. To install via [Terminus](/docs/terminus), [set the connection mode to SFTP](/docs/sftp) then run:

 ```
 terminus wp <site>.<env> -- plugin install wp-redis
 ```

 For site networks, you will need to specify the site URL by adding that to the command:

  ```
 terminus wp <site>.<env> -- plugin install wp-redis --url=<url>
 ```

2. Create a new file named `wp-content/object-cache.php` that contains the following:

 ```
 <?php
 # This is a Windows-friendly symlink
 require_once WP_CONTENT_DIR . '/plugins/wp-redis/object-cache.php';
 ```
This file is a symlink to the `/plugins/wp-redis/object-cache.php` file. Using SFTP or Git, commit the new file to the Dev environment.

3. In the Dev environment's WordPress Dashboard, verify installation by selecting **Drop-ins** from the Plugins section:

  ![The object-cache Drop-In Plugin](/docs/assets/images/redis-dropin-plugin.png "The object-cache plugin, visible in the Drop-ins section of Plugins.")

  When a new version of the WP Redis plugin is released, you can upgrade by the normal Plugin update mechanism in WordPress or via Terminus:

  ```
  terminus wp <site>.<env> -- plugin update wp-redis
  ```

### Install via Composer

1. Set the Dev environment's connection mode to Git from within the Site Dashboard or via Terminus:

 ```
 terminus connection:set <site>.<env> git
 ```

2. [Clone the site's codebase](/docs/git/#clone-your-site-codebase) if you have not done so already.

3. Use the following within `composer.json` to install the WP Redis plugin as a drop-in via Composer using [koodimonni/composer-dropin-installer](https://github.com/Koodimonni/Composer-Dropin-Installer):

   ```json
   "repositories": {
     "wpackagist": {
       "type": "composer",
       "url": "https://wpackagist.org"
     }
   },
   "require": {
     "composer/installers": "^1.0.21",
     "koodimonni/composer-dropin-installer": "*",
     "wpackagist-plugin/wp-redis": "0.6.0"
     },
     "extra": {
       "installer-paths": {
         "wp-content/plugins/{$name}/": ["type:wordpress-plugin"]
         },
       "dropin-paths": {
          "wp-content": [
          "package:wpackagist-plugin/wp-redis:object-cache.php"
        ]
      }
    }
   ```

4. Run `composer install` to install WP Redis into the `wp-content` directory.
5. Use git status to verify your local state, then commit and push your code to Pantheon:

 ```
 git status
 git commit --all -m "Initiate composer, require custom code"
 git push origin master
 ```

## Use the Redis Command-Line Client

1. [Install Redis locally](https://redis.io/download).
2. From the Site Dashboard, select the desired environment (Dev, Test, or Live).
3. Click the **Connection Info** button, copy the Redis connection string, and run the command in your local terminal.

Execute the following command to return existing Redis keys:
```bash
> keys *
```
If Redis is configured properly, it will show all existing keys. If nothing is returned, proceed to the troubleshooting section below.

To check if a specific key exists, you can pass the exists command:
```bash
> SET key1 "Hello"
OK
> EXISTS key1
(integer) 1
> EXISTS key2
(integer) 0
>
```
### Find a Specific Key

If you need to find a specific key, you can use search patterns that contain globs. For example:

```bash
redis> KEYS *a*
$17
englash bigkahuna
redis> KEYS engl?sh
$23
englosh englash english
redis> KEYS engl[ia]sh
$15
englash english
```
### Clear Cache

Pass the `flushall` command to clear all keys from the cache:

```bash
redis> flushall
OK
```
### Check the Number of Keys in Cache

To check the number of keys in the cache, you can use the `DBSIZE` command. The following is sample output:

```bash
redis> DBSIZE
:0
```

## Troubleshooting

### Cannot Activate the Redis Plugin
WP Redis is a drop-in plugin that should only be loaded using the installation methods above. No activation is required.

### Redis Server is Gone
Enable Redis via the Pantheon Site Dashboard by going to **Settings** > **Add Ons** > **Add** > **Redis**. It may take a few minutes to provision the service.

## Frequently Asked Questions

#### What happens when Redis reaches maxmemory?

The behavior is the same as a standard Redis instance. The overall process is described best in the top four answers of [this thread](https://stackoverflow.com/questions/8652388/how-does-redis-work-when-ram-starts-filling-up), keeping in mind our `maxmemory-policy` is `allkeys-lru`.

#### Is Redis set up as an LRU cache?

We are using [allkeys-lru](https://redis.io/topics/lru-cache). Here is the Redis configuration file for your Live environment:

```nohighlight
cat redis.conf
port 11455
timeout 300
loglevel notice
logfile /srv/bindings/xxxxxxxxx/logs/redis.log
databases 16
save 900 1
save 300 10
save 60 10000
rdbcompression yes
dbfilename dump.rdb
dir /srv/bindings/xxxxxxxxx/data/
requirepass 278801a71e2c4264b7d7b155def62bea
maxclients 1024
maxmemory 964689920
maxmemory-policy allkeys-lru
appendonly no
appendfsync everysec
no-appendfsync-on-rewrite no
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
activerehashing yes
```

#### If Redis hits the upper limit of memory usage, is this logged on Pantheon?

Yes. There is a `redis.log` file that is available on the Redis container for each environment. You can see where the log files and configuration reside:

```nohighlight
$ sftp -o Port=2222 live.81fd3bea-d11b-401a-85e0-07ca0f4ce7cg@cacheserver.live.81fd3bea-d11b-401a-85e0-07ca0f4ce7cg.drush.in Connected to cacheserver.live.81fd3bea-d11b-401a-85e0-07ca0f4ce7cg.drush.in.
sftp> ls -la
-rw-r--r-- 1 11455 11455 18 Oct 06 05:16 .bash_logout -rw-r--r-- 1 11455 11455 193 Oct 06 05:16 .bash_profile -rw-r--r-- 1 11455 11455 231 Oct 06 05:16 .bashrc -rw-r--r-- 1 0 0 0 Mar 10 19:46 .pantheonssh_login drwxr-x--- 2 0 11455 4096 Nov 10 07:55 certs
-rw-r--r-- 1 0 0 42 Mar 10 09:46 chef.stamp drwx------ 2 11455 11455 4096 Mar 10 19:46 data
drwxrwx--- 2 0 11455 4096 Nov 10 07:55 logs
-rw------- 1 0 0 2677 Mar 10 09:46 metadata.json -rw-r----- 1 0 11455 531 Nov 10 07:55 redis.conf drwxrwx--- 2 0 11455 4096 Mar 10 09:46 tmp
sftp> ls -la logs/
-rw-r--r-- 1 11455 11455 40674752 Mar 10 19:46 redis.log sftp>
```
#### I restored my site (or imported a database backup) and now my site won't come back.

When you replace the database with one that doesn't match the redis cache, it can cause database errors on the site, and you may be unable to clear the cache via the dashboard. To resolve the issue, [flush your redis cache from the command line](#clear-cache).
