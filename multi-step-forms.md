# Multi-step Form Builder

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
           if(in_array($form->getValue('up_sell'), ['yes'])){
             ContractResolver::dispatch(
                 UpSellLead::make($form)
             );
           }
       })
   ;
}
```


### Test Route


```php
Route::any('/', function(){
    return MultiStepForm::make('form') //View
        ->addStep(1, [
            'rules' => ['name' => 'required']
        ])
        ->addStep(2, [
            'rules' => ['role' => 'required']
        ])
        ->addStep(3, []) //No Rules, just reset form state.
        ->onStep(3, function (MultiStepForm $form) {
            $form->setValue('form_step',1);
            $form->setValue('name',null);
            $form->setValue('role',null);
        });
})->name('submit');
```

### Test View
```blade

<form method="post" action="{{ route('submit') }}">
    <input
        type="hidden"
        name="form_step"
        value="{{ session('multistep-form.form_step', 1) }}">
    @csrf

    @switch($form->currentStep())

        @case(1)
        <label>Name</label>
        <input
            type="text"
            name="name"
            value="{{ $form->getValue('name') }}">
            {{ $errors->first('name') }}
        @break

        @case(2)
        <label>Role</label>
        <input
            type="text"
            name="role"
            value="{{ $form->getValue('role') }}">
            {{ $errors->first('role') }}
        @break

        @case(3)
            Done
        @break
    @endswitch

    <button type="submit">Continue</button>
    <hr>

    {{ $form->toCollection() }}
</form>

```

### Builder Class
```php
<?php declare(strict_types=1);

namespace App;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Collection;
use Illuminate\Session\Store as Session;
use Illuminate\Config\Repository as Config;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Support\Facades\View;
use Illuminate\Validation\Rule;

class MultiStepForm implements Responsable
{
    public Request $request;
    public Session $session;
    public Config $config;
    public Collection $steps;
    public Collection $callbacks;
    static string $namespace = 'multistep-form';
    protected $view;

    public function __construct(
        Request $request,
        Session $session,
        Config $config,
        string $view = null
    ){
        $this->callbacks = new Collection;
        $this->steps = new Collection;
        $this->request = $request;
        $this->session = $session;
        $this->config = $config;
        $this->view = $view;
    }

    public static function make(?string $view = null): self
    {
        return app(self::class, compact('view'));
    }

    protected function handle($request = null)
    {
        $this->request = $request ?? $this->request;
            $this->validate()
                ->handleCallbacks()
                ->nextStep();
        return $this->toArray();
    }

    public function toResponse($request = null)
    {
        if(!$this->request->isMethod('GET')) {
            $this->handle($request ?? $this->request);
            if ($this->request->wantsJson() || !$this->view) {
                return new Response($this->toArray());
            }
            return redirect()->back();
        }
        return View::make($this->view, [
            'form' => $this
        ]);
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

    public function getValue(string $key, $fallback = null)
    {
        return $this->session->get(static::$namespace . ".$key", $fallback);
    }

    public function setValue(string $key, $value): self
    {
        $this->session->put(static::$namespace . ".$key", $value);
        return $this;
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
        $this->session->save();
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