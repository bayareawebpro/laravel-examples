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
        ->addStep(3)->onStep(3, function (MultiStepForm $form) {
           if($form->request->get('submit') === 'reset'){
                $form->reset();
           }else{
               return response('OK');
           }
        });
})->name('submit');
```

### Test View
```blade
<form method="post" action="{{ route('submit') }}">
    @csrf
    <input
        type="hidden"
        name="form_step"
        value="{{ $form->currentStep() }}">

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
            Name: {{ $form->getValue('name') }}<br>
            Role: {{ $form->getValue('role') }}<br>
        @break
    @endswitch

    @if($form->isStep(3))
   <button type="submit" name="submit">Save</button>
   <button type="submit" name="submit" value="reset">Reset</button>
   @else
   <button type="submit">Continue</button>
   @endif
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
use Illuminate\Validation\Rule;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\View;
use Illuminate\Session\Store as Session;
use Illuminate\Contracts\Support\Responsable;

class MultiStepForm implements Responsable
{
    static string $namespace = 'multistep-form';

    public Request $request;
    public Session $session;
    public Collection $steps;
    public Collection $callbacks;
    protected $view;

    public function __construct(
        Request $request,
        Session $session,
        $view = null
    ){
        $this->callbacks = new Collection;
        $this->steps = new Collection;
        $this->request = $request;
        $this->session = $session;
        $this->view = $view;
    }

    public static function make($view = null): self
    {
        return app(static::class, [
            'view' => $view
        ]);
    }

    public function namespaced(string $namespace): self
    {
        static::$namespace = $namespace;
        return $this;
    }

    public function toResponse($request = null)
    {
        $this->request = $request ?? $this->request;
        if($this->request->isMethod('GET')) {
            return $this->renderRequest();
        }
        return $this->handleRequest();
    }

    protected function renderRequest()
    {
        if(is_string($this->view)){
            return View::make($this->view, ['form' => $this]);
        }
        return new Response($this->toArray());
    }

    protected function handleRequest()
    {
        $this->validate();
        $this->nextStep();
        if ($response = $this->handleCallback()) {
            return $response;
        }
        if (is_string($this->view)) {
            return redirect()->back();
        }
        return new Response($this->toArray());
    }

    public function toArray(): array
    {
        return $this->session->get(static::$namespace, []);
    }

    public function toCollection(): Collection
    {
        return Collection::make($this->toArray());
    }

    public function addStep(int $number, array $config = []): self
    {
        $this->steps->put($number, $config);
        return $this;
    }

    public function onStep(int $number, \Closure $closure): self
    {
        $this->callbacks->put($number, $closure);
        return $this;
    }

    public function reset($data = []): self
    {
        $this->session->put(static::$namespace, array_merge($data, [
            'form_step' => 1
        ]));
        return $this;
    }

    public function currentStep(): int
    {
        return (int)$this->request->get('form_step',
            $this->session->get(static::$namespace . ".form_step", 1)
        );
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
            $this->session->get(static::$namespace, []), $data
        ));
        $this->session->save();
        return $this;
    }

    protected function handleCallback()
    {
        if ($this->callbacks->has($this->currentStep())) {
            $callback = $this->callbacks->get($this->currentStep());
            return $callback($this);
        }
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