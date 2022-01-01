## Magento

### Request flow

1. In `index.php` file:
   1. Create a bootstrap object
   2. Create an application object that implements the `\Magento\Framework\AppInterface` interface e.g. `\Magento\Framework\App\Http` for HTTP request or the `\Magento\Framework\App\Cron` for the Cron application.
   3. Run the application by calling `$bootstrap->run($app)`
2. The `run()` method of the `\Magento\Framework\App\Bootstrap` calls the `launch()` method of the `\Magento\Framework\AppInterface` which returns an object that implements the `\Magento\Framework\App\ResponseInterface` interface as an application's response, then calls the `sendResponse()` method of the response object to send the response.
3. In HTTP request context, the `launch()` method of the `Http` application does the following:
   1. Gets Magento 2 Area code (`adminhtml`, `frontend`, or `base`) from the request
   2. Sets area code in the application's state
   3. Loads configuration for the area code and uses it to configure the application's `ObjectManager` object (Not sure what this is)
   4. Gets the application's `\Magento\Framework\App\FrontController` object from the `ObjectManager` object. The `FrontController` implements the `\Magento\Framework\App\FrontControllerInterface` interface
   5. Calls the `dispatch()` method of the `FrontController` object with the `Magento\Framework\App\Request\Http` as an argument to get the result (how does the object manager instantiates the `FrontController` object?)
   6. Checks if the result object is an implementation of the `\Magento\Framework\Controller\ResultInterface`.
      a. If it is, calls the `renderResult()` method of the result object (I guess the method modifies application's response object in some ways since it returns nothing)
      b. If it implements the `\Magento\Framework\Controller\HttpInterface`, sets the result object as application's response
      c. Otherwise, throws the `\InvalidArgumentException`
   7. Returns the response
4. The `FrontController` object loops throuh each `Router` in the `Router` list given to it (by the object manager?). Within each loop (each router), the router object matches the request with its action objects
5. The matched action execute the request by composing a `Page` and returns it back to the router which returns back to the `FrontController` which returns back to the `Http` application object which send the response back to the HTTP client.

### Website Migration

_1. Backup filesystem_:

1. Skip to step 3. You don't need this since you would dump the database anyway~~Dump Magento 2 configuration from database to files~~:

```bash
cd <magento_root>
bin/magento app:config:dump system
```

2. Remove everything except the `modules` array in the `app/etc/config.php` file
3. Create directory for backup files:

```bash
mkdir -p backup
```

4. Zip all required files:

```bash
zip backup/html.zip -r html -i "html/app/*" "html/pub/*" -x "html/pub/static/*" "html/pub/media/catalog/product/cache/*"
```

5. Dump the database:

```bash
mysqldump -h <host> -u <username> -p --skip-comments <db_name> > backup/dump.sql
```

6. Zip the backup folder

```bash
zip backup.zip -r backup
```

7. Copy the zipped file to your host:

```bash
scp <user>@<host>:/path/to/backup.zip /path/to/paste/backup.zip
```

8. Unzip the backup.zip file:

```bash
unzip backup.zip -d .
unzip backup/html.zip -d backup
```

9. Merge the files:

```bash
rsync -avizP backup/html/app/ magento2/app/
rsync -avizP backup/html/pub/ magento2/pub/
```

10. Find files changes:

```bash
git status
git diff --name-only
git diff --summary
```

11. Clean the working tree e.g. remove unused files, restore mode-change files, merge changed files.
12. Commit the changes
13. Import the database:

```bash
cat backup/dump.sql | docker exec -i <mysql_container_name> mysql -u <root_username> -p<root_password> <db_name>
```

14. Change Magento 2 configuration in the `app/etc/env.php` file such as MySQL host, Redis host, Elasticsearch host, Varnish host, etc.
15. Import the Magento 2 configuration file:

```bash
docker compose exec -u `id -u <username>`:`id -g www-data` php-fpm sh -c "bin/magento app:config:import && bin/magento setup:upgrade && bin/magento setup:di:compile && bin/magento setup:static-content:deploy -f && bin/magento cache:flush"
```

### Make varnish purgable by Magento 2

1. In the `php-fpm` container, run `bin/magento setup:config:set --http-cache-hosts=<varnish_container_name>`
2. Then run `bin/magento setup:upgrade`. Done.

To check if the above steps work:

1. Run varnishlog: `docker compose exec varnish varnishlog`
2. Open another terminal then run: `docker compose exec php-fpm bin/magento cache:flush`

If the varnishlog terminal outputs the `PURGE` method then it is working correctly.

*See:*

- [How cache clearing works with Varnish](https://devdocs.magento.com/guides/v2.4/config-guide/varnish/use-varnish-cache.html)

### Locked configuration

If you've unknowingly run the `bin/magento app:config:dump` command, you might need to remove everything in the `app/etc/config.php` file except the `modules` array and run the following commands in order:

```bash
bin/magento setup:upgrade \
&& bin/magento setup:di:compile \
&& bin/magento setup:static-content:deploy -f \
&& bin/magento cache:flush
```

After that, you'll be able to configure your Magento 2 again in the admin side.

### Useful Links

1. Request Flow in Magento 2 -- [https://www.dckap.com/blog/request-flow-in-magento-2/](https://www.dckap.com/blog/request-flow-in-magento-2/)

## PHP-FPM

### Memory exhausted

You might find an error log in the `var/log/magento.cron.log` indicating that the memory allocation is exhausted like this:

```bash
PHP Fatal error:  Allowed memory size of 2097152 bytes exhausted (tried to allocate 4096 bytes) in /app/vendor/composer/autoload_static.php on line 3079
```

Consider lower your PHP's `memory_limit`. Yes, lower! Like 2G to 1G for example.

### About Argon 2ID3 Encryption

Magento 2 uses Argon 2ID3 as an upgraded hash algorithm. In order to make your PHP support Argon 2ID3 algorithm, you need to install and enable PHP Sodium extension which requires you to install `libsodium` package on your system.

You basically need to run these commands below:

```bash
wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.18.tar.gz \
    && tar xfvz libsodium-1.0.18.tar.gz \
    && cd libsodium-1.0.18 \
    && ./configure \
    && make && make install \
    && pecl install -f libsodium
```

And check if the extension included in PHP: `php -i | grep sodium`
You should see something like:

```bash
sodium
sodium support => enabled
sodium compiled version => 2.0.20
libsodium headers version => 1.0.18
libsodium library version => 1.0.18
```

If the extension is not enabled as expected, you should enable it directly in the `php.ini` file.

If you're using PHP-FPM Docker, you need to restart the container: `docker restart <container_id>`

_See more_:

- [GitHub issue | Magento 2.3.2 Undefined offset Encryptor.php](https://github.com/magento/magento2/issues/23511#issuecomment-520534869)
- [Magento DevBlog | Required libsodium upgrade for latest Magento release](https://community.magento.com/t5/Magento-DevBlog/Required-libsodium-upgrade-for-latest-Magento-release/bc-p/135463/highlight/true#M540)
- [grafzahl.io | Magento warning after updating to 2.3.2](http://blog.grafzahl.io/magento-warning-after-updating-to-2-3-2/)
