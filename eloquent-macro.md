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