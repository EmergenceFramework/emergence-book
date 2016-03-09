# Creating a Dynamic Page

```php
<?php
print("Hello world!");
```

Why the `<?php`? So you can do this:
```php
<html>
  <body>
    <p>
      <?php
        print("Hello, it's " . date('Y-m-d'));
      ?>
    </p>
  </body>
</html>
```

