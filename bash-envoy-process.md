```
<?php
$result = [];
$task = 'deploy';
$live = true;
$process = new \Symfony\Component\Process\Process('/Applications/MAMP/bin/php/php7.1.0/bin/php ~/.composer/vendor/bin/envoy run '. $task);
$process->setTimeout(3600);
$process->setIdleTimeout(300);
$process->setWorkingDirectory(base_path());
$process->run(
    function ($type, $buffer) use ($live, &$result) {
        $buffer = str_replace('[-i ~/.ssh/host_key user@host.com]: ', '', $buffer);
        if ($live) {
            echo $buffer . '</br />';
        }
        $result[] = $buffer;
    }
);
```