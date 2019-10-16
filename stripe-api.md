```
$stripeApi = app()->make('stripeApi');

$transaction = $stripeApi
    ->source($request->get('token'))
    ->destination($seller->stripe_id)
    ->amount($invoice->transaction_total_cents)
    ->fee($invoice->fee_cents)
    ->charge();

if($transaction->succeeded()){
    $invoice->pay($transaction->getId());
}
```

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


```
<?php namespace App\Http\Controllers\API\Stripe;

use App\StripeApi;
use Illuminate\Http\Request;
use App\Http\Controllers\Controller;

/**
 * Manages the Stripe Connect and fetches account data for the connected user.
 */
class AccountController extends Controller
{
	protected $stripe_api;

    /**
     * Create a new controller instance.
     * @param StripeApi $stripe_api
     * @return void
     */
    public function __construct(StripeApi $stripe)
    {
		$this->api = $stripe;
    }

	/**
	 * Get a connected Stripe account's temporary login URL
	 * @param Request $request
	 * @return \Illuminate\Http\Response
	 */
	public function login(Request $request){
		try {
			$user = $request->user();
			if(!is_null($user->stripe_id)){
				$stripe_login_links = $this->api->getAccount($user->stripe_id)->login_links->create();
                return redirect($stripe_login_links->url);
			}
		}catch(\Stripe\Error\Base $exception) {
			return response('Forbidden', 503);
		}
	}

	/**
	 * Connect a new Stripe account
	 * @param Request $request
	 * @return \Illuminate\Http\Response
	 */
	public function connect(Request $request){

		$this->validate($request, array(
			'code' => 'required|string'
		));

		if(!is_null($request->user()->stripe_id)){
            abort(401);
		}

        try {
            $stripeResponse = $this->api->connect($request->get('code'));
            $user->update([
                'stripe_id' => $stripeResponse->stripe_user_id
            ]);
            return response('Stripe Account Connected.');
        }catch(\Stripe\Error\Base $e) {
            return response(array('message' => $e->getMessage()),$e->getHttpStatus());
        }
	}

	/**
	 * Get a connected Stripe account's Details
	 * @param Request $request
	 * @return \Illuminate\Http\Response
	 */
	public function details(Request $request)
    {
		try {
            return response()->json(array(
                'message' => 'Stripe Account Synchronized.',
                'stripe_account' => $this->api->getAccount($request->user()->stripe_id),
            );
		}catch(\Stripe\Error\Base $e) {
		    abort(500);
		}
	}
}
```