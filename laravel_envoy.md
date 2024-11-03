# Detailed Guide: Creating a Laravel Envoy Deployment Script

This guide will walk you through creating an Envoy script for deploying Laravel applications, based on a practical example. We'll explain each component, allowing you to adapt it for your own projects.

## 1. Setting Up Servers

Start by defining your servers:

```php
@servers(['staging' => 'username@ip-here', 'production' => 'username@domain-name-here'])
```

This sets up two server environments: staging and production. Replace the SSH connections with your own server details.

## 2. Environment Setup

Use the `@setup` directive to define variables:

```php
@setup
    $env = isset($env) ? $env : 'staging';
    $repository = ($env == 'production') ? 'git@github.com:YourOrg/your-app.git' : 'git@github.com-repo-2:YourOrg/your-app.git';
    $branch = $env == 'production' ? 'main' : 'staging';
    $app_dir = $env == 'production' ? '/var/www/app' : '/var/www/your-app';
    $release = date('Y_m_d_H_i');
    $releases_dir = $app_dir . '/releases';
    $new_release_dir = $releases_dir .'/'. $release;
@endsetup
```

Adjust these variables to match your project:
- Set your repository URLs
- Define your branch names
- Set your application directories

## 3. Deployment Story

Create a deployment story that groups all tasks:

```php
@story('deploy', ['on' => $env])
    clone_repository
    run_composer
    update_symlinks
    writeable
    migrate
    restart_queues
    cleanup_old_releases
@endstory
```

This story ensures that all necessary steps, including cleaning up old releases, are performed during each deployment. It allows you to run all tasks with a single command: `envoy run deploy`.

## 4. Cloning the Repository

```php
@task('clone_repository')
    echo 'Cloning repository'
    [ -d {{ $releases_dir }} ] || mkdir -p {{ $releases_dir }}
    git clone --depth 1 --branch {{ $branch }} {{ $repository }} {{ $new_release_dir }}
    cd {{ $new_release_dir }}
    git reset --hard {{ $commit }}
@endtask
```

This task:
- Creates the releases directory if it doesn't exist
- Clones the specified branch into a new release directory
- Resets to a specific commit (useful for rolling back)

## 5. Running Composer

```php
@task('run_composer')
    echo 'Linking .env file'
    ln -nfs {{ $app_dir }}/.env {{ $new_release_dir }}/.env

    echo "Starting deployment ({{ $release }})"
    cd {{ $new_release_dir }}
    composer install --prefer-dist --no-scripts -q -o
@endtask
```

This task:
- Links the `.env` file from the main app directory
- Runs `composer install` in the new release directory

## 6. Updating Symlinks

```php
@task('update_symlinks')
    echo "Linking storage directory"
    rm -rf {{ $new_release_dir }}/storage
    ln -nfs {{ $app_dir }}/storage {{ $new_release_dir }}/storage

    echo 'Linking current release'
    ln -nfs {{ $new_release_dir }} {{ $app_dir }}/current

    echo 'Symling storage to public folder'
    cd {{ $new_release_dir }} && php artisan storage:link
@endtask
```

This crucial task:
- Links the shared storage directory
- Updates the 'current' symlink to the new release
- Creates the storage symlink in the public folder

## 7. Setting Proper Permissions

```php
@task('writeable')
    echo 'make bootstrap/cache writeable ...'
    cd {{ $new_release_dir }}
    chgrp -R www-data bootstrap/cache
    chmod -R g+w bootstrap/cache
@endtask
```

Ensures that the `bootstrap/cache` directory is writeable by the web server.

## 8. Running Migrations

```php
@task('migrate')
    echo "migrating database ..."
    cd {{ $new_release_dir }}
    php artisan migrate --force -q
@endtask
```

Runs database migrations. The `--force` flag is used to run migrations in production.

## 9. Restarting Queues

```php
@task('restart_queues')
    cd {{ $new_release_dir }}
    php artisan queue:restart
@endtask
```
Restarts the Laravel queue workers to ensure they're using the new code.

## 10. Cleaning Up Old Releases

```php
@task('cleanup_old_releases')
    echo "Cleaning up old releases..."
    cd {{ $releases_dir }}
    ls -dt */ | tail -n +8 | xargs -d "\n" rm -rf
@endtask
```

This task removes all but the 7 most recent release directories, helping to manage server space.


## Customizing for Your App

To adapt this script for your own application:

1. Update the `@servers` directive with your server details.
2. Modify the `@setup` variables to match your repository, branches, and directory structure.
3. Adjust the tasks or add new ones based on your specific deployment needs.
4. Consider adding tasks for:
   - Running tests before deployment
   - Clearing and recaching config and routes
   - Restarting any other services your app depends on

## Running the Deployment

To deploy using this script:

1. Save it as `Envoy.blade.php` in your project root.
2. Run `envoy run deploy` for staging deployment.
3. Run `envoy run deploy --env=production` for production deployment.

Remember to test thoroughly in a staging environment before deploying to production!