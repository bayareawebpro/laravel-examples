```shell script
$user = $request->user();
$user->update([
  'affiliate' => $request->only('affiliate')
]);
$user->save();
```

```shell script
<?php

/**
 * HasOne Affiliate Relationship
 * @return \Illuminate\Database\Eloquent\Relations\hasOne
 */
function affiliate(){
  return $this->hasOne(Affiliate::class);
}

/**
 * Fill Affiliate Relationship
 * @return void
 */
function setAffiliateAttribute(array $values){
  if($this->affiliate()->exists()){
    $this->affiliate()->first()->update($values);
  }else{
    $this->affiliate()->save(new Affiliate($value));
  }

}
```