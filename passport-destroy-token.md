```
<?php
/** OAuth Destroy Token **/
Route::delete('oauth/destroy', function(){
    //Get the Token from the Request.
    $token = request()->header('X-Auth-Token');
    $token = "XXXXX";
    /**
     * Get the Token Parser
     * @var $parser \Lcobucci\JWT\Parser
     */
    $parser = app('Lcobucci\JWT\Parser');
    /**
     * Get The Parsed Token
     * @var $parsedToken \Lcobucci\JWT\Token
     */
    $parsedToken = $parser->parse($token);
    /**
     * Get the JSON Token Identifier (Model ID)
     * @var $jsonTokenIdentifier string
     */
    $jsonTokenIdentifier = $parsedToken->getClaim('jti');
    /**
     * Get the Token Model (Model)
     * @var $tokenModel \Laravel\Passport\Token
     */
    $tokenModel = \Laravel\Passport\Token::find($jsonTokenIdentifier);
    /** Delete The Token */
    $tokenModel->delete();
});
```