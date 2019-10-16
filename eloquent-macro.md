
Developers who are used to writing SQL statements may have trouble getting the most out of Laravel’s ORM / Query Builder. When Builder methods don’t offer the exact implementation we’re looking for it’s common to write SQL expressions using the select(DB::raw(…)) implementation.

These types of statements can be written in a more eloquent way by using Laravel’s “Macros”. The operation I want to perform will be called “insertOrUpdateMany” which will build and execute a custom SQL statement with a single method call.
Define the Macro:

To keep the code clean and readable I pass the Query Builder instance bound to the Macro definition’s context and the callback’s single argument “$rows” to a new instance of my Macro Class.
```
use Illuminate\Database\Query\Builder;
use App\Macros\InsertOrUpdateMany;Builder::macro('insertOrUpdateMany', function(array $rows){
    return with(new InsertOrUpdateMany($this, $rows))->execute();
});
```
A new service provider is a good place to store these definitions.
Macro Usage:

This macro will not use a chained query thats passed to it. Instead, it uses a single method call just like the “insert” method that’s already available. The expected result will be the number of rows affected by the query.
```
$affectedRowCount = DB::table('users')->insertOrUpdateMany(array(
    array(
        'id' => 1, 
        'name' => 'Test 1', 
        'email' => 'test1@test.local', 
        'password' => 'XXX'
    ),
    array('id' => 2, 
        'name' => 'Test 2', 
        'email' => 'test2@test.local', 
        'password' => 'XXX'
    ),
    array(
        'id' => 3, 
        'name' => 'Test 3', 
        'email' => 'test3@test.local', 
        'password' => 'XXX'
    ),
));

dd($affectedRowCount);
```

Database Query Class

The InsertOrUpdateMany Class will assemble and execute the statement against the database connection instance (bound to the Query Builder) the macro is called from. Using the affectingStatement method allows the number of affected rows to be returned to the caller. Easy as pie.

```
<?php namespace App\Macros;
use Illuminate\Database\Query\Builder;
class InsertOrUpdateMany
{
    /**
     * @param \Illuminate\Database\Query\Builder $builder
     * @var \PDO $pdo
     * @var array $entries
     * @var array $columnsArray
     * @var string $columnsString
     * @var string $valuesString
     * @var string $updateString
     * @var string $sql
     */
    protected $builder, $pdo, $entries;
    private $columnsArray, $columnsString, $valuesString, $updateString, $sql;
    /**
     * InsertOrUpdateMany constructor.
     * @param \Illuminate\Database\Query\Builder $builder
     * @param array $entries
     * @return void
     */
    public function __construct(Builder $builder, array $entries)
    {
        $this->builder = $builder;
        $this->entries = $entries;
        $this->pdo = $this->builder->getConnection()->getPdo();
        $this->prepareValues();
        $this->assembleStatement();
    }
    /**
     * Prepare the values.
     * @return void
     */
    private function prepareValues()
    {
        //Get the Columns Array
        $this->columnsArray = $this->getColumnsArray();
        //Build the Columns String
        $this->columnsString = $this->collapse($this->columnsArray);
        //Build Values String
        $this->valuesString = $this->collapse(array_map(function ($row) {
            return $this->encase($this->collapse($this->escape($row)));
        }, $this->entries));
        //Build Updates String
        $this->updateString = $this->collapse(array_map(function ($value) {
            return "$value = VALUES($value)";
        }, $this->columnsArray));
    }
    /**
     * Assemble the SQL statement.
     * @return void
     */
    private function assembleStatement()
    {
        $this->sql = "INSERT INTO {$this->builder->from} ({$this->columnsString}) VALUES {$this->valuesString} ON DUPLICATE KEY UPDATE {$this->updateString}";
    }
    /**
     * Get the table columns array.
     * @return array
     */
    private function getColumnsArray()
    {
        return array_keys(reset($this->entries));
    }
    /**
     * Escape the values array.
     * @param array $values
     * @return array
     */
    private function escape(array $values)
    {
        return array_map(function ($value) {
            return $value ? $this->pdo->quote($value) : 'NULL';
        }, $values);
    }
    /**
     * Collapse a value array.
     * @param array $values
     * @return string
     */
    private function collapse(array $values)
    {
        return implode(',', $values);
    }
    /**
     * Encase a value string.
     * @param string $value
     * @return string
     */
    private function encase(string $value)
    {
        return '(' . $value . ')';
    }
    /**
     * Execute Query
     * @return int
     */
    public function execute()
    {
        return $this->builder->getConnection()->affectingStatement($this->sql);
    }
}
```