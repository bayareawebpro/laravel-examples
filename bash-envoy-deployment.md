```
@php
    $key = '~/.ssh/host_key';
    $host = 'user@host.com';
    $connection = '-i '.$key.' '.$host;
    $home = '/home/myroot';
    $development = $home.'/webapps/dev';
    $releases = $home.'/webapps/deployments/releases';
    $storage = $home.'/webapps/deployments/storage';
    $current = $home.'/webapps/deployments/current';
    $release = $releases.'/ver_'.date('YmdHis');
    $php = '/usr/local/bin/php71';
    $composer = $home.'/composer.phar';
@endphp

@servers(['local' => '127.0.0.1','web' => '-i ~/.ssh/host_key user@host.com'])

@task('makeKey', ['on' => 'local'])
    ssh-keygen -t rsa;
    scp {{$key}}.pub {{$host}}:temp_id_rsa_key.pub;
@endtask

@task('appendKey', ['on' => 'web'])
    cat ~/temp_id_rsa_key.pub >> ~/.ssh/authorized_keys;
    rm temp_id_rsa_key.pub;
    chmod 600 ~/.ssh/authorized_keys;
    chmod 700 ~/.ssh;
    chmod go-w $HOME;
@endtask

@task('destroy', ['on' => 'web'])
rm -rf {{ $home.'/webapps/deployments/*' }}
@endtask

@task('deploy', ['on' => 'web'])

    echo "Checking directories...";
    [ -d {{ $releases }} ] || mkdir {{ $releases }};
    [ -d {{ $storage }} ] || mkdir {{ $storage }}; chmod 777 {{ $storage }};
    [ -d {{ $storage }}/app ] || mkdir {{ $storage }}/app;
    [ -d {{ $storage }}/app/public ] || mkdir {{ $storage }}/app/public;
    [ -d {{ $storage }}/framework ] || mkdir {{ $storage }}/framework;
    [ -d {{ $storage }}/framework/cache ] || mkdir {{ $storage }}/framework/cache;
    [ -d {{ $storage }}/framework/sessions ] || mkdir {{ $storage }}/framework/sessions;
    [ -d {{ $storage }}/framework/views ] || mkdir {{ $storage }}/framework/views;



    echo "Cloning development to new release...";
    cp -r {{ $development }} {{ $release }};

    echo "Cleaning up release...";
    rm -rf {{ $release }}/vendor/;
    rm -rf {{ $release }}/storage/;
    rm -rf {{ $release }}/public/storage;


    echo "Creating symlink for release storage...";
    ln -s {{ $storage }} {{ $release }};

    echo "Installing release dependencies...";
    cd {{ $release }};
    {{ $php }} {{ $composer }} install --no-ansi --quiet --no-dev --no-interaction --no-progress --optimize-autoloader;

    echo "Creating symlink for release public storage...";
    {{ $php }} artisan storage:link;

    echo "Cleaning up caches...";
    {{ $php }} artisan cache:clear --quiet;
    {{ $php }} artisan view:clear --quiet;
    {{ $php }} artisan route:clear --quiet;

    echo "Optimizing release...";
    {{ $php }} artisan route:cache --quiet;
    {{ $php }} artisan optimize --quiet;

    echo "Creating symlink for release...";
    ln -nfs {{ $release }} {{ $current }};

    echo "Release is active...";
@endtask
```