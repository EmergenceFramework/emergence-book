# Make a Model Searchable by Tags

To make a model searchable by tags, we'll make use of the `callback` option for `$searchConditions` to specify a function that can generate advanced SQL procedurally.

First, define a new search condition:

```php
<?php

class MyModel extends \ActiveRecord
{
    // ...

    public static $searchConditions = [
        'Tags' => [
            'qualifiers' => ['any', 'tag', 'tags'],
            'points' => 2,
            'callback' => 'getTagsSearchConditions'
        ]
    ];

    // ...
}
```

The `callback` option can take any PHP `callable`, or a simple string referencing a public static method on the model class. This method will be called for each term in the search query and should return an SQL expression to match records that match this term. The ultimate results list gets sorted by how many terms are matched.

While it is possible to implement joins via `$searchConditions`, in this example we are instead going to "unroll" any matching tags ahead of time into a list of tagged record IDs while the search query is being prepared:

```php
    public static function getTagsSearchConditions($term)
    {
        $ids = \DB::allValues(
            'ContextID',
            '
            SELECT TagItem.ContextID
              FROM `%1$s` Tag
              JOIN `%2$s` TagItem
                ON TagItem.TagID = Tag.ID AND TagItem.ContextClass = "%3$s"
             WHERE Tag.Title LIKE "%%%4$s%%" OR Tag.Handle LIKE "%%%4$s%%"
            ',
            [
                \Tag::$tableName, // %1$s
                \TagItem::$tableName, // %2$s
                \DB::escape(static::getStaticRootClass()), // %3$s
                \DB::escape($term) // %4$s
            ]
        );

        return count($ids) ? 'ID IN ('.implode(',',$ids).')' : '0';
    }
```
