
## Article
- https://medium.com/@danielalvidrez/a-custom-places-autocomplete-service-with-laravel-elasticsearch-6563898d80d4

## Docs
- https://www.elastic.co/guide/en/elasticsearch/guide/current/configuring-analyzers.html
- https://www.elastic.co/guide/en/elasticsearch/guide/current/asciifolding-token-filter.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-edgengram-tokenizer.html
- https://www.elastic.co/guide/en/elasticsearch/reference/6.7/search-aggregations-bucket-geohashgrid-aggregation.html
- https://hackernoon.com/elasticsearch-building-autocomplete-functionality-494fcf81a7cf


## Data
- https://osmnames.org
- https://openaddresses.io

## Package
- https://github.com/babenkoivan/scout-elasticsearch-driver
```
composer require babenkoivan/scout-elasticsearch-driver
```

## Migration
```
Schema::create('locations', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->string('name', 256)->nullable();
    $table->string('display_name', 500)->nullable();
    $table->text('alternative_names')->nullable();
    $table->string('street', 256)->nullable()->index();
    $table->string('city', 256)->nullable()->index();
    $table->string('county', 256)->nullable()->index();
    $table->string('state', 256)->nullable()->index();
    $table->string('country', 256)->nullable()->index();
    $table->string('country_code', 2)->nullable()->index();
    $table->string('zipcode', 20)->nullable()->index();
    $table->string('wikidata', 256)->nullable();
    $table->string('wikipedia', 256)->nullable();
    $table->string('osm_type', 128)->nullable()->index();
    $table->string('osm_id', 128)->nullable()->index();
    $table->string('class', 128)->nullable()->index();
    $table->string('type', 128)->nullable()->index();
    $table->decimal('lat', 11, 8)->nullable()->index();
    $table->decimal('lon', 11, 8)->nullable()->index();
    $table->decimal('west', 11, 8)->nullable()->index();
    $table->decimal('south', 11, 8)->nullable()->index();
    $table->decimal('east', 11, 8)->nullable()->index();
    $table->decimal('north', 11, 8)->nullable()->index();
    $table->decimal('importance',11, 8)->nullable()->index();
    $table->tinyInteger('place_rank')->nullable()->index();
    $table->index(['west', 'south', 'east', 'north']);
    $table->index(['lat', 'lon']);
});
```

## Notes: 

Insure All Fields All Filled or Null to prevent indexing errors.
```
/**
 * Get the indexable data array for the model.
 * @return array
 */
public function toSearchableArray()
{
    return [
        'field' => $this->field ?? null
    ];
}
```

## Laravel Model
```
<?php
namespace App;
use ScoutElastic\Searchable;
use Illuminate\Database\Eloquent\Model;
class Location extends Model
{
    use Searchable;

    public $timestamps = false;

    /**
     * Fillable Attributes
     * @var array
     */
    public $fillable = [
        "osm_type",
        "osm_id",
        "class",
        "type",
        "name",
        "display_name",
        "alternate_names",
        "street",
        "city",
        "county",
        "state",
        "country",
        "country_code",
        "lat",
        "lon",
        "north",
        "south",
        "east",
        "west",
        "place_rank",
        "importance",
    ];

    /**
     * Attribute Casts
     * @var array
     */
    public $casts = [
        "lat" => "double",
        "lon" => "double",
        "north" => "double",
        "south" => "double",
        "east" => "double",
        "west" => "double",
        "importance" => "double",
    ];

    /**
     * Appendable Attributes
     * @var array
     */
    public $appends = [
        "geo_point",
    ];

    /**
     * LocationsIndexConfigurator
     * @var string $indexConfigurator
     */
    protected $indexConfigurator = LocationIndex::class;

    /**
     * Search Rules
     * @var array $searchRules
     */
    protected $searchRules = [];

    /**
     * Get the indexable data array for the model.
     * @return array
     */
    public function toSearchableArray()
    {
        return $this->toArray();
    }

    /**
     * Get GeoPoint Attribute
     * @return array
     */
    public function getGeoPointAttribute(){
        return $this->only(["lat", "lon"]);
    }

    /**
     * Model Search Mapping
     * @var array
     */
    protected $mapping = [
        "properties" => [
            "name" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "display_name" => [
                "type" => "text",
                "analyzer" =>  "autocomplete",
                "search_analyzer" =>  "autocomplete_search",
                "fields" => [
                    "raw" => [
                        "type" => "completion",
                    ]
                ]
            ],
            "alternate_names" => [
                "type" => "text",
                "analyzer" =>  "autocomplete",
                "search_analyzer" =>  "autocomplete_search",
                "fields" => [
                    "raw" => [
                        "type" => "completion",
                    ]
                ]
            ],
            "street" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "city" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "county" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "state" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "country" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "country_code" => [
                "type" => "text",
                "analyzer" => "keyword",
                "fields" => [
                    "raw" => [
                        "type" => "keyword",
                    ]
                ],
            ],
            "lat" => [
                "type" => "double",
            ],
            "lon" => [
                "type" => "double",
            ],
            "north" => [
                "type" => "double",
            ],
            "south" => [
                "type" => "double",
            ],
            "east" => [
                "type" => "double",
            ],
            "west" => [
                "type" => "double",
            ],
            "importance" => [
                "type" => "float",
            ],
            "place_rank" => [
                "type" => "float",
            ],
            "geo_point" => [
                "type" => "geo_point"
            ],
        ]
    ];
}
```

## Index Configurator
```
<?php
namespace App;
use ScoutElastic\Migratable;
use ScoutElastic\IndexConfigurator;

class LocationIndex extends IndexConfigurator
{
    use Migratable;

    protected $name = "locations";

    /**
     * Configure Analyzers
     * @var array
     */
    protected $settings = [
        "analysis" => [
            "analyzer" => [
                "keyword" => [
                    "type"=> "custom",
                    "tokenizer"=> "keyword",
                    "filter"=> [
                        "lowercase",
                        "asciifolding"
                    ],
                ],
                "autocomplete" => [
                    "tokenizer" => "edge_ngram",
                    "filter" =>[
                        "lowercase",
                        "asciifolding"
                    ],
                ],
                "autocomplete_search" => [
                    "tokenizer" => "lowercase",
                    "filter"=> [
                        "lowercase",
                        "asciifolding"
                    ],
                ]
            ],
            "tokenizer" => [
                "edge_ngram"=> [
                    "type"=> "edge_ngram",
                    "min_gram"=> 2,
                    "max_gram"=> 10,
                    "token_chars" => [
                        "letter",
                        "digit"
                    ]
                ],
            ],
            "aggregations" => [
                "large-grid" => [
                    "geohash_grid" => [
                        "field" => "geo_point",
                        "precision" => 3,
                    ]
                ]
            ],
        ]
    ];
}
```
