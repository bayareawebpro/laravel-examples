### Stripe API Example
- See Unit Tests for Usage
- https://github.com/stripe/stripe-php

```
composer require stripe/stripe-php
```

## API Class
```
<?php declare(strict_types=1);

namespace App;

use Illuminate\Support\Facades\Config;
use Illuminate\Support\Arr;
use Stripe\StripeObject;
use Stripe\Subscription;
use Stripe\Customer;
use Stripe\Balance;
use Stripe\Account;
use Stripe\Stripe;
use Stripe\Payout;
use Stripe\Charge;
use Stripe\OAuth;
use Stripe\Card;
use Stripe\Plan;

class StripeApi
{
    const CURRENCY = 'usd';

    public
        $subscription,
        $transaction,
        $customer,
        $account,
        $balance,
        $payout,
        $charge,
        $stripe,
        $plan,
        $card;

    /**
     * StripeApi constructor.
     * @param Stripe $stripe
     * @param Subscription $subscription
     * @param Customer $customer
     * @param Balance $balance
     * @param Account $account
     * @param Payout $payout
     * @param Charge $charge
     * @param Card $card
     * @param Plan $plan
     */
    public function __construct(
        Subscription $subscription,
        Stripe $stripe,
        Customer $customer,
        Balance $balance,
        Account $account,
        Payout $payout,
        Charge $charge,
        Card $card,
        Plan $plan
    )
    {
        $this->stripe = $stripe;
        $this->stripe->setClientId(Config::get('services.stripe.client'));
        $this->stripe->setApiKey(Config::get('services.stripe.secret'));
        $this->subscription = $subscription;
        $this->customer = $customer;
        $this->balance = $balance;
        $this->account = $account;
        $this->payout = $payout;
        $this->charge = $charge;
        $this->card = $card;
        $this->plan = $plan;
        $this->transaction = [
            "amount"   => 0,
            "currency" => self::CURRENCY,
            "source"   => null,
        ];
    }

    /**
     * Make new instance of self.
     * @return static
     */
    public static function make(): StripeApi
    {
        return app(self::class);
    }

    /**
     * Get Login Url
     * @param string $redirectUri
     * @return string
     */
    public function getLoginUrl(string $redirectUri = 'http://localhost'): string
    {
        return OAuth::authorizeUrl([
            'scope'          => 'read_write',
            'stripe_landing' => 'login',
            'redirect_uri'   => $redirectUri,
        ]);
    }

    /**
     * @param array $params
     * @param array $options
     * @return Plan
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function makePlan(array $params, array $options = []): Plan
    {
        return $this->getPlan($params) ?? $this->plan->create(array_merge([
                "currency" => self::CURRENCY,
            ], $params), $options);
    }

    /**
     * @param array $params
     * @return Plan|null
     */
    public function getPlan(array $params)
    {
        try {
            return $this->plan->retrieve(Arr::get($params, 'id'));
        } catch (\Stripe\Exception\ApiErrorException $exception) {
            return null;
        }
    }

    /**
     * Update Plan
     * @param array $params
     * @return Plan
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function updatePlan(array $params): Plan
    {
        return $this->plan->update(Arr::get($params, 'id'), Arr::except($params, 'id'));
    }

    /**
     * Create Customer From Token
     * @param string $token
     * @param array $attributes
     * @return Customer
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function createCustomerFromToken(string $token, array $attributes = []): Customer
    {
        return $this->customer->create(array_merge([
            "source" => $token,
        ],$attributes));
    }

    /**
     * Subscribe Customer to Plan
     * @param string $customer
     * @param string $planId
     * @return Subscription
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function subscribeToPlan(string $customer, string $planId): Subscription
    {
        return $this->subscription->create([
            "customer" => $customer,
            "items" => [
                [ "plan" => $planId ],
            ]
        ]);
    }

    /**
     * Get Subscription
     * @param string $subscriptionId
     * @return Subscription
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function getSubscription(string $subscriptionId): Subscription
    {
        return $this->subscription->retrieve($subscriptionId);
    }

    /**
     * Update Subscription Meta
     * @param string $subscriptionId
     * @param array $meta
     * @return Subscription
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function updateSubscription(string $subscriptionId, array $meta = []): Subscription
    {
        return $this->subscription->update($subscriptionId, [
            'metadata' => $meta
        ]);
    }

    /**
     * UnSubscribe from Plan
     * @param string $subscriptionId
     * @return Subscription
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function unSubscribeFromPlan(string $subscriptionId): Subscription
    {
        return $this->getSubscription($subscriptionId)->cancel();
    }

    /**
     * Connect Account
     * @param string $code
     * @return StripeObject Object containing the response from the API.
     * @throws \Stripe\Exception\OAuth\OAuthErrorException
     */
    public function connect(string $code): StripeObject
    {
        return OAuth::token([
            'code'       => $code,
            'grant_type' => 'authorization_code',
        ]);
    }

    /**
     * Get Account Data
     * @param $stripeID
     * @param $options array|string|null
     * @return \Stripe\Account
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function getAccount($stripeID, $options = null): Account
    {
        return $this->account->retrieve($stripeID, $options);
    }

    /**
     * Get Account Balance
     * @param $options
     * @return Account|Balance
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function getBalance($options = []): Balance
    {
        return $this->balance->retrieve($options);
    }

    /**
     * Set Transaction Source
     * @param $stripeID
     * @return $this
     */
    public function source($stripeID)
    {
        data_set($this->transaction, 'source', $stripeID);
        return $this;
    }

    /**
     * Set Transaction MetaData
     * @param array $data
     * @return $this
     */
    public function withMeta(array $data)
    {
        data_set($this->transaction, 'metadata', $data);
        return $this;
    }

    /**
     * Set Transaction Destination
     * @param $stripeID
     * @return $this
     */
    public function destination($stripeID)
    {
        data_set($this->transaction, 'destination.account', $stripeID);
        return $this;
    }

    /**
     * Set Transaction Amount
     * @param $dollarsInCents
     * @return $this
     */
    public function amount($dollarsInCents)
    {
        data_set($this->transaction, 'amount', $dollarsInCents);
        return $this;
    }

    /**
     * Add Application Fee
     * @param $dollarsInCents
     * @return $this
     */
    public function fee($dollarsInCents)
    {
        data_set($this->transaction, 'destination.amount',
            ($this->transaction['amount'] - $dollarsInCents)
        );
        return $this;
    }

    /**
     * Charge & Process the Transaction
     * @return $this
     * @throws \Stripe\Exception\ApiErrorException
     */
    public function charge()
    {
        $this->transaction = $this->charge->create($this->transaction);
        return $this;
    }

    /**
     * Transaction Succeeded
     * @return bool
     */
    public function succeeded()
    {
        return (
            data_get($this->transaction, 'status') === 'succeeded' &&
            data_get($this->transaction, 'paid') === true
        );
    }

    /**
     * Get Transaction ID
     * @return mixed
     */
    public function getTransactionId()
    {
        return data_get($this->transaction, 'id');
    }

    /**
     * Get Transaction ID
     * @return mixed
     */
    public function toArray()
    {
        return $this->transaction;
    }
}
```

---

## Unit Tests

- Uses real api, update to use mocks, dump responses to get the schema.
- Create a Test Customer Account to Account & OAuth Steps

```
<?php namespace Tests\Feature;

use App\StripeApi;

use Illuminate\Support\Str;
use Illuminate\Support\Facades\Config;

use Tests\TestCase;

class StripeApiAccountTest extends TestCase
{

    const TEST_KEY = 'ca_XXX';
    const TEST_CLIENT_ID = 'sk_test_XXX';
    const TEST_ACCOUNT_ID = 'acct_XXX';

    protected function setUp(): void{
        parent::setUp();
        Config::set('services.stripe.client', self::TEST_KEY);
        Config::set('services.stripe.secret', self::TEST_CLIENT_ID);
    }
  
    public function test_can_get_own_balance()
    {
        //Get the test Account.
        $stripe = StripeApi::make();

        $balance = $stripe->getBalance()->toArray();
        $this->assertArrayHasKey('available', $balance);
        $this->assertArrayHasKey('pending', $balance);
        dump($balance);
    }

    public function test_can_get_oauth_links()
    {
        //Get the test Account.
        $stripe = StripeApi::make();

        //Open Link and Follow Directions to get response token.
        dump($stripe->getLoginUrl());

        $this->assertTrue(Str::contains($stripe->getLoginUrl(), Config::get('services.stripe.client')));
    }

    public function test_can_connect_with_token()
    {
        $stripe = StripeApi::make();

        //Token from previous test.
        $token = 'ac_G0TNtnezSGNLrufBYVb4u0Ev266OdZAH';
        $response = $stripe->connect($token)->toArray();

        $this->assertSame(self::TEST_ACCOUNT_ID, data_get($response, 'stripe_user_id'));
    }

    public function test_can_get_accounts()
    {
        //Get the test Account.
        $stripe = StripeApi::make();

        $account = $stripe->getAccount(self::TEST_ACCOUNT_ID)->toArray();

        $this->assertSame(self::TEST_ACCOUNT_ID, data_get($account, 'id'));
        $this->assertArrayHasKey('charges_enabled', $account);
        $this->assertArrayHasKey('payouts_enabled', $account);
        dump($account);
    }

    public function test_can_make_transactions()
    {
        //Get the test Account.
        $stripe = StripeApi::make();

        $stripe
            // Customer
            ->source('tok_visa_debit')
            ->amount(5000)
            ->withMeta([
                'user_id' => 1,
            ])
            ->charge();

        $this->assertTrue($stripe->succeeded());
        $this->assertNotEmpty($stripe->getTransactionId());
    }

    public function test_can_make_split_payments()
    {

        //Get the test Account.
        $stripe = StripeApi::make();

        $transaction = $stripe
            // Customer
            ->source('tok_visa_debit')
            ->destination(self::TEST_ACCOUNT_ID)
            ->amount(5000)
            ->fee(2500)
            ->withMeta([
                'user_id' => 1,
            ])
            ->charge();

        $this->assertTrue($transaction->succeeded());
        $this->assertNotEmpty($transaction->getTransactionId());
    }

    public function test_can_get_make_plans()
    {
        //Get the test Account.
        $stripe = StripeApi::make();

        $plan = $stripe->makePlan([
            'id' => 'gold-special',
            "interval" => "month",
            "currency" => "usd",
            "amount" => 5000,
            'metadata' => [],
            "product" => [
                "name" => "Gold special"
            ],
        ]);
        $this->assertArrayNotHasKey('testKey', $plan->metadata->toArray());

        $plan = $stripe->updatePlan([
            'id' => 'gold-special',
            'metadata' => [
                'testKey' => true
            ]
        ]);
        $plan->refresh();
        $this->assertArrayHasKey('testKey', $plan->metadata->toArray());

        $plan->delete();
        $this->assertTrue($plan->isDeleted());
    }

    public function test_can_subscribe_to_plans()
    {
        //Get the test Account.
        $stripe = StripeApi::make();

        $plan = $stripe->makePlan([
            'id' => 'gold-special',
            "interval" => "month",
            "currency" => "usd",
            "amount" => 5000,
            'metadata' => [],
            "product" => [
                "name" => "Gold special"
            ],
        ]);
        $customer = $stripe->createCustomerFromToken('tok_visa_debit', [
            'email' => 'test@tester.local'
        ]);

        $subscription = $stripe->subscribeToPlan($customer->id, $plan->id);

        $this->assertSame($plan->id, data_get($subscription, 'plan.id'));
        $this->assertNotEmpty(data_get($subscription, 'latest_invoice'));
        $this->assertEmpty(data_get($subscription, 'canceled_at'));
        $this->assertEmpty(data_get($subscription, 'ended_at'));

        $this->assertArrayNotHasKey('testKey', $subscription->metadata->toArray());
        $subscription = $stripe->updateSubscription($subscription->id, [
            'testKey' => true
        ]);
        $this->assertArrayHasKey('testKey', $subscription->metadata->toArray());

        $subscription = $stripe->unSubscribeFromPlan($subscription->id);
        $this->assertNotEmpty(data_get($subscription, 'canceled_at'));
    }
}
```