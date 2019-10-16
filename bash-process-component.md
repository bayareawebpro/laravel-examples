# SSH Usage with Process Component

The Symphony Process Component is included with Laravel, here's an example of executing a command with artisan.

```

$process = new Process("ssh -C forge@hostname \"mysqldump staging\" | /Applications/MAMP/Library/bin/mysql staging");
$process->setTimeout(60*10);
$process->setTty(Process::isTtySupported());
$process->setEnv(array(
    "PATH" => implode(':', array(
        "/usr/local/bin",
        "/usr/bin",
        "/usr/sbin",
        "/sbin",
    ))
));
$process->run(function ($type, $buffer) use (&$processOutput, &$process) {
    if ($process::OUT === $type) {
        $this->info($buffer);
    } else {
        $this->error($buffer);
    }
});

```