# Eloquent Temporary Tags

### Auto-expiring tags with additional payload data on any eloquent model.

<br>
<br>
<br>
<br>
<p align="center"><a target="_blank" rel="noopener noreferrer" href="https://user-images.githubusercontent.com/6961695/93660285-6a935180-fa62-11ea-98ca-5a7675c6bd6a.png"><img src="https://user-images.githubusercontent.com/6961695/93660285-6a935180-fa62-11ea-98ca-5a7675c6bd6a.png" alt="image" style="max-width:100%;"></a></p>

<br>
<h4 align="center">
Built with :heart: for every smart laravel developer
</h4>


### Installation:

```bash
composer require mjraipro/laravel-temp-tag

```

Then you can publish and migrate to create the needed tags table

```bash
php artisan vendor:publish
php artisan migrate
```




### Use cases:

- You wanna ban a user, but only for a week.

- You wanna give someone VIP access, but only for a month.

- Comments approval by admin.

- You wanna put a product in a slider until the weekend.

- Storing each user preferences, can be done by attaching a 'settings' tag and the preferences as payload.

- Storing users likes and dislikes on any model.

- Scheduling models to be published in the future.

- Coupon code for specific products on specific periods.


Then you put a temporary tag on them and check to see if the model has the tag.

### Tag Payload:

You can also store some additional json data, for example why the user got banned, or who banned the user, or a slug or a translation of the title.

This is done by passing the third argument as an array to the ```->tagIt(...)``` method

### Keynotes:

- Tags can also be permanent.

- We do not touch your existing tables in migrations.

- You can put tag on any eloquent model or any other object with `getKey` and `getTable` methods on it.

--------------

### Example:

1- Tag a user until tomorrow

```php

  $user = User::find(1);
  $tomorrow = Carbon::now()->addDay();
  $note = ['reason' => 'You were nasty!']; // You can optionally store additional data in a json column.

  tempTags($user)->tagIt('banned', $tomorrow, $note);

  tempTags($user)->tagIt('banned');  // will never expire
```

2- After an hour the tag is still active, so:

```php
  $user = User::find(1);
 
  $tagObj = tempTags($user)->getActiveTag('banned');  
  
  if (! is_null($tagObj)) {
    $tagObj->isActive();         // true
    $tagObj->isPermanent();      // false
    $tagObj->title === 'banned'; // true
    $tagObj->payload;            // ['reason' => 'You were nasty!']
    $tagObj->expiresAt();        // Carbon instance
  }
```


3- After a week, the tag gets expired so:

```php
  $user = User::find(1);
  $tagObj = tempTags($user)->getTag('banned');  // <--- fetches the tag regardless of its expire date.
  
  if (! is_null($tagObj)) {
    $tagObj->isActive();         // false
    $tagObj->isPermanent();      // false
    $tagObj->title === 'banned'; // true
    $tagObj->expiresAt();        // Carbon instance
  }
```
--------------

Getting payload data:

There are 2 ways to get payload data :

You can use `getPayload` method 
```php
  $tagObj->getPayload('reason');       //  'You were nasty!'      
  $tagObj->getPayload();               //  ['reason' => 'You were nasty!'] 
  $tagObj->getPayload('missing_key');  //  null
```

or You can act like eloquent attributes :
```php
  $tagObj->reason;       //  'You were nasty!'      
  $tagObj->payload;      //  ['reason' => 'You were nasty!'] 
  $tagObj->missing_key;  //  null
```


--------------

#### Deleting tags:

```php
  $user = User::find(1);
  
  tempTags($user)->unTag('banned');           // single string

  tempTags($user)->unTag('bann*');            // using a wildcard 

  tempTags($user)->unTag(['banned', 'man']);  // an array of tags to delete

  tempTags($user)->deleteExpiredTags();       // all the expired tags, bye bye.

```

**Note:** You can also use * wildcard which matcher 0 or more characters (just like % in sql)

**Note:** These fire "deleting" and "deleted" eloquent events for each one of them.


Expire the tag with title of "banned" right now:

```php

 tempTags($user)->expireNow('banned');  // updates the value of expire_at to now() 

```

-------------

These methods just do what they say:

```php

  $actives = tempTags($user)->getAllActiveTags();  // A collect of "TempTag" model objects.

  $expired = tempTags($user)->getAllExpiredTags();

  $all = tempTags($user)->getAllTags();

```

-------------

### Fetch only tagged models:

Let's say you have a slider for your `Product` model and you want to show only those records which are tagged with 'slider'.

First you have to put `Imanghafoori\Tags\Traits\hasTempTags` trait on the `Product` model.

```php

class Product extends Model 
{
  use hasTempTags;
  
  ...
}
```

Now you can perform these queries:

```php
Product::hasActiveTags('slider')->where(...)->get();

// Only if the tag of model is expired and it has the specified title.
Product::hasExpiredTags('slider')->where(...)->get();

// To fetch regardless of expiration date of tags, only the title matters.
Product::hasTags('slider')->where(...)->get();
```

**Note:** If you pass an array of tags it acts like a `whereIn()`, so if the row has one of tags if will be selected.

If you want to find products which will be active until tomorrow, you can use `hasActiveTagsAt`, or `hasNotActiveTagsAt` as illustrated below:
```php
Product::hasActiveTagsAt('slider', now()->addDay())->where(...)->get();

Product::hasNotActiveTagsAt('slider', now()->addDay())->where(...)->get();
```

#### Example:

When you want to send a notification to users, just 24 hours before their VIP account gets finished, so that they can charge. 
```php
User::hasActiveTag('VIP')
    ->hasNotActiveTagsAt('VIP', now()->addDay())
    ->get();
```
-------------

### Absence of tags:

```php
Product::hasNotActiveTags('slider')->where(...)->get();

Product::hasNotExpiredTags('slider')->where(...)->get();

Product::hasNotTags('slider')->where(...)->get();
```

### Example:
When you want your article to be published in the future, go can tag your articles with a custom tag title like: "hidden"
And fetch them like this:
```php
$articles = Article::hasNotActiveTags('hidden')->where(...)->get();
```
So, as long as the 'hidden' tag is active the article does not show on the list and when the tag gets expired the article goes live.


-------------

### Filtering based on payload:

```php
$product1 = Product::find(1);
$product2 = Product::find(1);

tempTags($product1)->tagIt('status', $tomorrow, ['value' => 'sold_out']);
tempTags($product2)->tagIt('status', $tomorrow, ['value' => 'sold_out']);

// now we have tagged these two rows we can fetch them later on with the below query

...

Product::hasActiveTags('status', ['value' => 'sold_out'])->where(...)->get();
```

The above example gives you the products with active status tag which have also payload data with the specified key and value.

**Note:** You can also use * wildcard which matcher 0 or more characters (just like % in sql):
```php

Product::hasActiveTags('stat*', ['value' => 'sold_out'])->where(...)->get();
```

-------------

### Advanced Usage:

Each and every `has...Tag` method also has a twin `orHas...Tag` so you can put multiple conditions in the query.

Remember that they both should reside in a `->where(function ($q) {` beside one another.

```php
Post::where('user_id', 12)->where(function ($q) {
    $q->hasActiveTags('status', ['value' => 'active'])
    ->orHasActiveTags('status', ['value' => 'promoted']);
})->get();
```

--------------------
