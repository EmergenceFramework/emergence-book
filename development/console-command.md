# Development: Build a Console Command

- Create `console-commands/myproject/mycommand.php`
- Example content:

    ```php
    <?php

    echo "Hello ".Site::$title;

    dump($_COMMAND);

    $_COMMAND['LOGGER']->warning('Danger Will Robinson!');
    ```

- Update site:

    ```bash
    update-site
    ```

- Run command:

    ```bash
    console-run myproject:mycommand --foo "bar"
    ```
