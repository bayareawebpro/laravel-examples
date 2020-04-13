# Multi-step Form Builder

You could easily modify this to return views instead of json.

```php
public function submission()
{
   return MultiStepForm::make()
       ->addStep(1, [
           'rules' => Locations::rules(),
           'messages' => Locations::messages(),
       ])
       ->addStep(2, [
           'rules' => Person::rules(),
           'messages' => Person::messages(),
       ])
       ->addStep(3, [
           'rules' => Vehicle::rules(),
           'messages' => Vehicle::messages(),
       ])
       ->onStep(3, function(MultiStepForm $form){
           ContractResolver::dispatch(
               PrimaryLead::make($form)
           );
       })
       ->addStep(4, [
           'rules' => Household::rules(),
           'messages' => Household::messages(),
       ])
       ->onStep(4, function(MultiStepForm $form){
           if(in_array($form->request->get('up_sell'), ['yes'])){
             ContractResolver::dispatch(
                 UpSellLead::make($form)
             );
           }
       })
   ;
}
```

```php
<?php declare(strict_types=1);

namespace App;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Collection;
use Illuminate\Session\Store as Session;
use Illuminate\Config\Repository as Config;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Validation\Rule;

class MultiStepForm implements Responsable
{
    public Request $request;
    public Session $session;
    public Config $config;
    public Collection $steps;
    public Collection $callbacks;
    static string $namespace = 'multistep-form';

    public function __construct(Request $request, Session $session, Config $config)
    {
        $this->callbacks = new Collection;
        $this->steps = new Collection;
        $this->request = $request;
        $this->session = $session;
        $this->config = $config;
    }

    public static function make(): self
    {
        return app(self::class);
    }

    protected function handle($request = null)
    {
        $this->request = $request ?? $this->request;
        $this
            ->validate()
            ->handleCallbacks();
        //Get data before mutating session state.
        $data = $this->toArray();
        $this->nextStep();
        return $data;
    }

    public function toResponse($request = null): Response
    {
        return response($this->handle($request ?? $this->request));
    }

    public function toArray(): array
    {
        return $this->session->get(static::$namespace, []);
    }

    public function toCollection(): Collection
    {
        return Collection::make($this->toArray());
    }

    public function addStep(int $number, array $attrs): self
    {
        $this->steps->put($number, $attrs);
        return $this;
    }

    public function onStep(int $number, \Closure $closure): self
    {
        $this->callbacks->put($number, $closure);
        return $this;
    }

    public function currentStep(): int
    {
        return (int)$this->request->get('form_step', $this->session->get(static::$namespace . ".form_step", 1));
    }

    public function stepConfig(int $step = 1): Collection
    {
        return Collection::make($this->steps->get($step));
    }

    public function isStep(int $step = 1): bool
    {
        return (bool)($this->currentStep() === $step);
    }

    protected function nextStep(): self
    {
        if (!$this->isStep($this->steps->count())) {
            $this->session->increment(static::$namespace . '.form_step');
        }
        return $this;
    }

    protected function save(array $data): self
    {
        $this->session->put(static::$namespace, array_merge(
            $this->session->get(static::$namespace, []),
            $data
        ));
        return $this;
    }

    protected function handleCallbacks()
    {
        if ($this->callbacks->has($this->currentStep())) {
            $callback = $this->callbacks->get($this->currentStep());
            $callback($this);
        }
        return $this;
    }

    protected function validate(): self
    {
        $step = $this->stepConfig($this->currentStep());
        $this->save($this->request->validate(
            array_merge($step->get('rules', []), [
                'form_step' => ['required', 'numeric', Rule::in(range(1, $this->steps->count()))],
            ]),
            $step->get('messages', [])
        ));
        return $this;
    }
}
```