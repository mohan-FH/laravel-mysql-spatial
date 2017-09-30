# Laravel MySQL Spatial extension

[![Build Status](https://img.shields.io/travis/grimzy/laravel-mysql-spatial.svg?style=flat-square)](https://travis-ci.org/grimzy/laravel-mysql-spatial)
[![Code Climate](https://img.shields.io/codeclimate/github/grimzy/laravel-mysql-spatial/badges/gpa.svg?style=flat-square)](https://codeclimate.com/github/grimzy/laravel-mysql-spatial)
[![Code Climate](https://img.shields.io/codeclimate/coverage/github/grimzy/laravel-mysql-spatial.svg?style=flat-square)](https://codeclimate.com/github/grimzy/laravel-mysql-spatial/coverage)
[![Packagist](https://img.shields.io/packagist/v/grimzy/laravel-mysql-spatial.svg?style=flat-square)](https://packagist.org/packages/grimzy/laravel-mysql-spatial)
[![Packagist](https://img.shields.io/packagist/dt/grimzy/laravel-mysql-spatial.svg?style=flat-square)](https://packagist.org/packages/grimzy/laravel-mysql-spatial)
[![license](https://img.shields.io/github/license/mashape/apistatus.svg?style=flat-square)](LICENSE)

Laravel package to easily work with [MySQL Spatial Data Types](https://dev.mysql.com/doc/refman/5.7/en/spatial-datatypes.html) and [MySQL Spatial Functions](https://dev.mysql.com/doc/refman/5.7/en/spatial-function-reference.html).

Please check the documentation for your MySQL version. MySQL's Extension for Spatial Data was added in MySQL 5.5 but many Spatial Functions were changed in 5.6 and 5.7.

**Versions**

- `1.x.x`: MySQL 5.6 (also supports MySQL 5.5 but not all spatial analysis functions)
- `2.x.x`: MySQL 5.7 and 8.0

[TOC]

## Installation

Add the package using composer:

```shell
composer require grimzy/laravel-mysql-spatial
```

For MySQL 5.6 and 5.5:

```shell
composer require grimzy/laravel-mysql-spatial:^1.0
```

For Laravel versions before 5.5 or if not using auto-discovery, register the service provider in `config/app.php`:

```php
'providers' => [
  /*
   * Package Service Providers...
   */
  Grimzy\LaravelMysqlSpatial\SpatialServiceProvider::class,
],
```



## Quickstart

### Create a migration

From the command line:

```shell
php artisan make:migration create_places_table
```

Then edit the migration you just created by adding at least one spatial data field:

```php
use Illuminate\Database\Migrations\Migration;
use Grimzy\LaravelMysqlSpatial\Schema\Blueprint;

class CreatePlacesTable extends Migration {

    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('places', function(Blueprint $table)
        {
            $table->increments('id');
            $table->string('name')->unique();
            // Add a Point spatial data field named location
            $table->point('location')->nullable();
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::drop('places');
    }
}
```

Run the migration:

```shell
php artisan migrate
```

### Create a model

From the command line:

```shell
php artisan make:model Place
```

Then edit the model you just created. It must use the `SpatialTrait` and define an array called `$spatialFields` with the name of the MySQL Spatial Data field(s) created in the migration:

```php
namespace App;

use Illuminate\Database\Eloquent\Model;
use Grimzy\LaravelMysqlSpatial\Eloquent\SpatialTrait;

/**
 * @property \Grimzy\LaravelMysqlSpatial\Types\Point $location
 */
class Place extends Model
{
    use SpatialTrait;

    protected $fillable = [
        'name',
    ];

    protected $spatialFields = [
        'location',
    ];
}
```

### Saving a model

```php
$place1 = new Place();
$place1->name = 'Empire State Building';
$place1->location = new Point(40.7484404, -73.9878441);
$place1->save();
```

### Retrieving a model

```php
$place2 = Place::first();
$lat = $place2->location->getLat();	// 40.7484404
$lng = $place2->location->getLng();	// -73.9878441
```



## Migrations

### Columns

Available [MySQL Spatial Types](https://dev.mysql.com/doc/refman/5.7/en/spatial-datatypes.html) migration blueprints:

- 
   `$table->geometry('column_name');`

- `$table->point('column_name');`
- `$table->lineString('column_name');`
- `$table->polygon('column_name');`
- `$table->multiPoint('column_name');`
- `$table->multiLineString('column_name');`
- `$table->multiPolygon('column_name');`
- `$table->geometryCollection('column_name');`

### Spatial indexes

You can add or drop spatial indexes in your migrations with the `spatialIndex` and `dropSpatialIndex` blueprints.

- `$table->spatialIndex('column_name');`
- `$table->dropSpatialIndex(['column_name']);` or `$table->dropSpatialIndex('index_name')`

Note about spatial indexes from the [MySQL documentation](https://dev.mysql.com/doc/refman/5.7/en/creating-spatial-indexes.html):

> For [`MyISAM`](https://dev.mysql.com/doc/refman/5.7/en/myisam-storage-engine.html) and (as of MySQL 5.7.5) `InnoDB` tables, MySQL can create spatial indexes using syntax similar to that for creating regular indexes, but using the `SPATIAL` keyword. Columns in spatial indexes must be declared `NOT NULL`.

Following the example in the [Quickstart](#user-content-create-a-migration); from the command line:

```shell
php artisan make:migration update_places_table
```

Then edit the migration file that you just created:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class UpdatePlacesTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        // MySQL < 5.7.5: table has to be MyISAM
        // \DB::statement('ALTER TABLE places ENGINE = MyISAM');

        Schema::table('places', function (Blueprint $table) {
            // Make sure point is not nullable
            $table->point('location')->change();
          
            // Add a spatial index on the location field
            $table->spatialIndex('location');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('places', function (Blueprint $table) {
            $table->dropSpatialIndex(['location']); // either an array of column names or the index name
        });

        // \DB::statement('ALTER TABLE places ENGINE = InnoDB');

        Schema::table('places', function (Blueprint $table) {
            $table->point('location')->nullable()->change();
        });
    }
}
```



## Geometry classes

### Available Geometry classes

| Grimzy\LaravelMysqlSpatial\Types         | OpenGIS Class                            |
| ---------------------------------------- | ---------------------------------------- |
| `Point($lat, $lng)`                      | [Point](https://dev.mysql.com/doc/refman/5.7/en/gis-class-point.html) |
| `MultiPoint(Point[])`                    | [MultiPoint](https://dev.mysql.com/doc/refman/5.7/en/gis-class-multipoint.html) |
| `LineString(Point[])`                    | [LineString](https://dev.mysql.com/doc/refman/5.7/en/gis-class-linestring.html) |
| `MultiLineString(LineString[])`          | [MultiLineString](https://dev.mysql.com/doc/refman/5.7/en/gis-class-multilinestring.html) |
| `Polygon(LineString[])` *([exterior and interior boundaries](https://dev.mysql.com/doc/refman/5.7/en/gis-class-polygon.html))* | [Polygon](https://dev.mysql.com/doc/refman/5.7/en/gis-class-polygon.html) |
| `MultiPolygon(Polygon[])`                | [MultiPolygon](https://dev.mysql.com/doc/refman/5.7/en/gis-class-multipolygon.html) |
| `GeometryCollection(Geometry[])`         | [GeometryCollection](https://dev.mysql.com/doc/refman/5.7/en/gis-class-geometrycollection.html) |

### Using Geometry classes

In order for your Eloquent Model to handle the Geometry classes, it must use the `Grimzy\LaravelMysqlSpatial\Eloquent\SpatialTrait` trait and define a `protected` property `$spatialFields`  as an array of MySQL Spatial Data Type column names (example in [Quickstart](#user-content-create-a-model)).

#### IteratorAggregate and ArrayAccess

The "composite" Geometries (`LineString`, `Polygon`, `MultiPoint`, `MultiLineString`, and `GeometryCollection`) implement [`IteratorAggregate`](http://php.net/manual/en/class.iteratoraggregate.php) and [`ArrayAccess`](http://php.net/manual/en/class.arrayaccess.php); making it easy to perform Iterator and Array operations. For example:

```php
$polygon = $multipolygon[10];	// ArrayAccess

// IteratorAggregate
for($polygon as $i => $linestring) {
  echo (string) $linestring;
}

```

#### Helpers

##### From/To Well Kown Text ([WKT](https://dev.mysql.com/doc/refman/5.7/en/gis-data-formats.html#gis-wkt-format))

```php
$polygon = Polygon::fromWKT('POLYGON((0 0,4 0,4 4,0 4,0 0),(1 1, 2 1, 2 2, 1 2,1 1))');

$polygon->toWKT();	// POLYGON((0 0,4 0,4 4,0 4,0 0),(1 1, 2 1, 2 2, 1 2,1 1))
```

##### From/To String

```php
// fromString($wkt)
$polygon = Polygon::fromString('(0 0,4 0,4 4,0 4,0 0),(1 1, 2 1, 2 2, 1 2,1 1)');

(string)$polygon;	// (0 0,4 0,4 4,0 4,0 0),(1 1, 2 1, 2 2, 1 2,1 1)
```

##### From/To JSON ([GeoJSON](http://geojson.org/))

The Geometry classes implement [`JsonSerializable`](http://php.net/manual/en/class.jsonserializable.php) and `Illuminate\Contracts\Support\Jsonable` and with the help of [jmikola/geojson](https://github.com/jmikola/geojson), we can easily serialize/deserialize GeoJSON:

```php
$point = new Point(10, 20);

json_encode($point);
// or
$point->toJson();
```

Returns:

```javascript
{
  "type": "Feature",
  "properties": {},
  "geometry": {
    "type": "Point",
    "coordinates": [
      -73.9878441,
      40.7484404
    ]
  }
}
```

##### 

## Scopes: Spatial analysis functions

Spatial analysis functions are implemented using [Eloquent Local Scopes](https://laravel.com/docs/5.4/eloquent#local-scopes).

Available scopes:

- `distance($geometryColumn, $geometry, $distance)`
- `distanceExcludingSelf($geometryColumn, $geometry, $distance)`
- `distanceSphere($geometryColumn, $geometry, $distance)`
- `distanceSphereExcludingSelf($geometryColumn, $geometry, $distance)`
- `comparison($geometryColumn, $geometry, $relationship)`
- `within($geometryColumn, $polygon)`
- `crosses($geometryColumn, $geometry)`
- `contains($geometryColumn, $geometry)`
- `disjoint($geometryColumn, $geometry)`
- `equals($geometryColumn, $geometry)`
- `intersects($geometryColumn, $geometry)`
- `overlaps($geometryColumn, $geometry)`
- `doesTouch($geometryColumn, $geometry)`

*Note that behavior and availability of MySQL spatial analysis functions differs in each MySQL version (cf. [documentation](https://dev.mysql.com/doc/refman/5.7/en/spatial-function-reference.html)).*



## Credits

Originally inspired from [njbarrett's Laravel postgis package](https://github.com/njbarrett/laravel-postgis).
