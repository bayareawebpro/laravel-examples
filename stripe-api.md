
## Split Payment

```
$stripeApi = app()->make('stripeApi');

$transaction = $stripeApi
    ->source($request->get('token'))
    ->destination($seller->stripe_id)
    ->amount($invoice->transaction_total_cents)
    ->fee($invoice->fee_cents)
    ->charge();

if($transaction->succeeded()){

    //Update your model...
    $invoice->markAsPaid($transaction->getId());
}
```

## Login Url
```
$stripeApi
    ->getAccount($request->user()->stripe_id)
    ->login_links
    ->create();
```

## Connect Account
```
$request->user()->update([
    'stripe_id' => $stripeApi->connect($request->get('code'))->stripe_user_id,
]);
```

## Get Account
```
$stripeApi->getAccount($user->stripe_id);
```

## API Class
```
<?php namespace App;
use Stripe\Stripe;
use Stripe\Account;
use Stripe\Payout;
use Stripe\Charge;
use Stripe\Card;
use Stripe\OAuth;
use Stripe\Balance;
class StripeApi
{
	protected $stripe, $account, $payout, $charge, $card, $auth, $apiKey, $balance;
	/**
	 * @var array
	 */
	public $transaction = array(
		"amount" => 0,
		"currency" => "usd",
		"source" => null,
		"destination" => array(
			"amount" => 0,
			"account" => null,
		),
	);
	/**
	 * StripeApi constructor.
	 * @param Stripe $stripe
	 * @param Account $account
	 * @param Payout $payout
	 * @param Charge $charge
	 * @param Card $card
	 * @param Balance $balance
	 */
	public function __construct(
		Stripe $stripe,
		Account $account,
		Payout $payout,
		Charge $charge,
		Card $card,
		Balance $balance
	) {
		$this->stripe = $stripe;
		$this->account = $account;
		$this->payout = $payout;
		$this->charge = $charge;
		$this->card = $card;
		$this->balance = $balance;
		$this->stripe->setClientId(config()->get('services.stripe.client'));
		$this->stripe->setApiKey(config()->get('services.stripe.secret'));
	}
	/**
	 * Connect Account
	 * @param $code
	 * @return \Stripe\StripeObject
	 */
	public function connect($code){
		return OAuth::token(array(
			'grant_type' => 'authorization_code',
			'code' => $code,
		));
	}
	/**
	 * Get Account Data
	 * @param $stripeID
	 * @param $options array|string|null
	 * @return Account|Charge|Payout
	 */
	public function getAccount($stripeID, $options = null){
		return $this->account->retrieve($stripeID, $options);
	}
	/**
	 * Get Account Balance
	 * @param $options
	 * @return Account|Balance
	 */
	public function getBalance($options){
		return $this->balance->retrieve($options);
	}
	/**
	 * Set Transaction Source
	 * @param $stripeID
	 * @return $this
	 */
	public function source($stripeID){
		$this->transaction['source'] = $stripeID;
		return $this;
	}
	/**
	 * Set Transaction Destination
	 * @param $stripeID
	 * @return $this
	 */
	public function destination($stripeID){
		$this->transaction['destination']['account'] = $stripeID;
		return $this;
	}
	/**
	 * Set Transaction Amount
	 * @param $dollarsInCents
	 * @return $this
	 */
	public function amount($dollarsInCents){
		$this->transaction['amount'] = $dollarsInCents;
		return $this;
	}
	/**
	 * Add Application Fee
	 * @param $dollarsInCents
	 * @return $this
	 */
	public function fee($dollarsInCents){
		$this->transaction['destination']['amount'] = ($this->transaction['amount'] - $dollarsInCents);
		return $this;
	}
	/**
	 * Charge & Process the Transaction
	 * @return $this
	 */
	public function charge(){
		$this->transaction = $this->charge->create($this->transaction);
		return $this;
	}
	/**
	 * Transaction Succeeded
	 * @return bool
	 */
	public function succeeded(){
		return $this->transaction['status'] == 'succeeded' && $this->transaction['paid'] == true;
	}
	/**
	 * Get Transaction ID
	 * @return mixed
	 */
	public function getId(){
		return $this->transaction['id'];
	}
}
```