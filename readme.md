# Nova Select Plus

## Installation

```
composer require ziffmedia/nova-select-plus
```

## Versions &amp; Compatibility

- Version `^1.0` supports Nova 2-3
- Version `^2.0` supports Nova 4 (with Vue 3, dark mode support, etc)
- Version `^2.1` supports Nova 5

## Description &amp; Use Cases

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/0-intro.gif "Intro Gif")

This Nova component was built to satisfy the use cases just beyond Nova's built-in `<select>` component. Here are
some scenarios where you might want `SelectPlus` (which uses `vue-select`) over the simple `Select`:

#### Select For BelongsToMany and MorphsToMany On the Form Screen

The default Nova experience for `BelongsToMany` and `MorphsToMany` is to have a separate UI screen for
attaching/detaching and syncing relationships through a "Pivot" model. For simple relationships (relationships that do
not have addition pivot values or the only value in the pivot table is there for ordering), it is benefitial to move
this Selection to the Form workflow instead of a separate workflow.

#### Ajax For Options

For Select's that have between a handful to several 1000 options, it is more peformant to load the full list of options
only on the screen that needs it: the Form screen.

There are 2 options for Ajax Options, the default is to load them all on the Form load. The second is to allow for full
option searching (in this case you can write you own ajax search resolver).

### Reordering Simple Pivot/BelongsToMany Relations

Through `->reorderable()`, you can enable a `SelectPlus` field to be reorderable. This allows, at `BelongsToMany->sync()`
time, to populate a pivot value useful for ordering relations.

## Usage

```php
use ZiffMedia\NovaSelectPlus\SelectPlus;
```

```php
    // setup model like normal:
    public function statesLivedIn()
    {
        return $this->belongsToMany(State::class, 'state_user_lived_in')->withTimestamps();
    }

    // add Nova Resource Field
    SelectPlus::make('States Lived In', 'statesLivedIn', State::class),
```

### Options & Examples

#### `->label(string|closure $attribute)` Pick a different attribute to use as the label

`Default: 'name'`

```php
SelectPlus::make('States Lived In', 'statesLivedIn', State::class)
  ->label('code')
```

If a closure is provided, it will be called and the value can be utilized. Additionally, the output may be HTML as the component will `v-html` the output on the frontend:

```php
// Using php 7.4 short functions:
SelectPlus::make('States Visited', 'statesVisited', State::class)
    ->label(fn ($state) => $state->name . " <span class=\"text-xs\">({$state->code})</span>")
```

#### `->usingIndexLabel()` & `->usingDetailLabel()`

Default is to produce a count of the number of items on the index and detail screen

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/1-default.png "Default Index")

If a *string* name is provided, the name attribute is plucked and comma joined:

```php
SelectPlus::make('States Lived In', 'statesLivedIn', State::class)
  ->usingIndexLabel('name'),
```

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/2-usingIndexLabel-string.png "string and comma separated")

If a closure is provided, it will be called, and the value will be utilized.  If the value is a string, it will be placed:

```php
SelectPlus::make('States Lived In', 'statesLivedIn', State::class)
  ->usingIndexLabel(fn($models) => $models->first()->name ?? ''),
```

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/3-usingIndexLabel-callback.png "return just the first name")

If an array is returned, the Index and Detail screens will produce a `<ol>` or `<ul>` list:

```php
SelectPlus::make('States Lived In', 'statesLivedIn', State::class)
  ->usingIndexLabel(fn($models) => $models->pluck('name')),
```

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/4-usingDetailLabel-array.png "array of values")

#### `->reorderable(string $pivotOrderAttribute)` - Ability to reorder multiple selects

```php
    // assuming in the User model:
    public function statesVisited()
    {
        return $this->belongsToMany(State::class, 'state_user_visited')
            ->withPivot('order')
            ->orderBy('order')
            ->withTimestamps();
    }

    // inside the Nova resource:
    SelectPlus::make('States Lived In', 'statesLivedIn', State::class)
        ->reorderable('order'),
```

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/5-reorderable.gif "reorder a list")

#### `->optionsQuery(closure)` - Ability to apply changes to options query object

```php
    // inside the Nova resource (exclude all states that start with C)
    SelectPlus::make('States Lived In', 'statesLivedIn', State::class)
        ->optionsQuery(function (Builder $query) {
            $query->where('name', 'NOT LIKE', 'C%');
        })
```

* Note: this will apply before any `ajaxSearchable()` functionality, it will not replace it but be applied along with `ajaxSearchable()` if it exists

#### `->ajaxSearchable(string|closure|true)` Ajax search for values

Given a string, models will be search the resources via the provided attribute using WHERE LIKE. Given a callback,
returning a Collection will populate the dropdown:

```php
    SelectPlus::make('States Visited', 'statesVisited', State::class)
        ->ajaxSearchable(function ($search) {
            return StateModel::where('name', 'LIKE', "%{$search}%")->limit(5);
        })
```

![alt text](https://github.com/ziffmedia/nova-select-plus/raw/master/docs/6-ajaxSearchable.gif "reorder a list")

## Validation

To make sure that the user has selected at least one option, you can utilize the `required` rule in combination with a custom validation rule. Since the data is sent back as form data, the value of your field will always be a JSON string of an empty array. Therefore, the traditional `required` rule will not actually validate the field as containing an entry. It will however add the red asterisk to the field, which tells the user it is required.

First create the custom validation rule:

```bash
php artisan make:rule RequiredSelectField
```

Then in your resource, add the following to your field:

```php
SelectPlus::make('States Visited', 'statesVisited', State::class)
    ->rules(['required', new RequiredSelectField()])

// This will also add the asterisk to the field
SelectPlus::make('States Visited', 'statesVisited', State::class)
    ->required()
    ->rules([new RequiredSelectField()])
```

Your rule should contain the following logic:

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    if (empty(json_decode($value))) {
        $fail("The {$attribute} field is required.");
    }
}
```

This way, if the field doesn't contain any selections, the validation will fail.
