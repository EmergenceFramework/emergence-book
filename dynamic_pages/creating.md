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
        print("Hello, it&rsquo;s " . date('Y-m-d'));
      ?>
    </p>
  </body>
</html>
```

There are syntax shortcuts available for something so simple though:
```php
<html>
  <body>
    <p>Hello, it&rsquo;s <?=date('Y-m-d')?></p>
  </body>
</html>
```