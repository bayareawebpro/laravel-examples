## EnforcesPolicies Trait

```
<?php declare(strict_types=1);

namespace App\Http\Requests\Traits;
use Illuminate\Support\Facades\Gate;

trait EnforcesPolicies{
    /**
     * Get the validated data from the request.
     * @return \Illuminate\Contracts\Auth\Access\Gate
     */
    public function getGate()
    {
        return Gate::forUser($this->user());
    }
}
```

## Form Request
```
<?php declare(strict_types=1);
namespace App\Http\Requests;

use App\Post;
use Illuminate\Support\Collection;
use App\Http\Requests\Traits\Filterable;
use App\Http\Requests\Traits\EnforcesPolicies;
use Illuminate\Foundation\Http\FormRequest;

class PostRequest extends FormRequest
{
    use EnforcesPolicies;

    /**
     * Determine if the user is authorized to make this request.
     * @return bool
     */
    public function authorize()
    {
        $gate = $this->getGate();
        switch ($this->route()->getName()) {
            case 'posts.index':
                return $gate->allows('viewAny', [Post::class]);
                break;
            case 'posts.create':
                return $gate->allows('create', [Post::class]);
                break;
            case 'posts.store':
                return $gate->allows('store', [Post::class]);
                break;
            case 'posts.show':
                return $gate->allows('view', [Post::class, $this->route('post')]);
                break;
            case 'posts.edit':
                return $gate->allows('edit', [Post::class, $this->route('post')]);
                break;
            case 'posts.update':
                return $gate->allows('update', [Post::class, $this->route('post')]);
                break;
            case 'posts.destroy':
                return $gate->allows('destroy', [Post::class, $this->route('post')]);
                break;
            default:
                return false;
                break;
        }
    }
    
    ...
}
```

## Policy
```
<?php declare(strict_types=1);
namespace App\Policies;

use App\User;
use App\Post;
use Illuminate\Auth\Access\HandlesAuthorization;

class PostPolicy
{
    use HandlesAuthorization;

    /**
     * Determine whether the user can view any posts.
     * @param  \App\User  $user|null
     * @return mixed
     */
    public function viewAny(?User $user)
    {
        return true;
    }

    /**
     * Determine whether the user can view the post.
     * @param  \App\User  $user|null
     * @param  \App\Post  $post
     * @return mixed
     */
    public function view(?User $user, Post $post)
    {
        return true;
    }

    /**
     * Determine whether the user can create posts.
     * @param  \App\User  $user
     * @return mixed
     */
    public function create(User $user)
    {
        return true;
    }

    /**
     * Determine whether the user can store posts.
     * @param  \App\User  $user
     * @return mixed
     */
    public function store(User $user)
    {
        return true;
    }

    /**
     * Determine whether the user can edit the post.
     * @param  \App\User  $user
     * @param  \App\Post  $post
     * @return mixed
     */
    public function edit(User $user, Post $post)
    {
        return $user->getAttribute('id') === $post->getAttribute('user_id');
    }

    /**
     * Determine whether the user can update the post.
     * @param  \App\User  $user
     * @param  \App\Post  $post
     * @return mixed
     */
    public function update(User $user, Post $post)
    {
        return $user->getAttribute('id') === $post->getAttribute('user_id');
    }

    /**
     * Determine whether the user can delete the post.
     * @param  \App\User  $user
     * @param  \App\Post  $post
     * @return mixed
     */
    public function delete(User $user, Post $post)
    {
        return $user->getAttribute('id') === $post->getAttribute('user_id');
    }

    /**
     * Determine whether the user can restore the post.
     * @param  \App\User  $user
     * @param  \App\Post  $post
     * @return mixed
     */
    public function restore(User $user, Post $post)
    {
        return $user->getAttribute('id') === $post->getAttribute('user_id');
    }

    /**
     * Determine whether the user can permanently delete the post.
     * @param  \App\User  $user
     * @param  \App\Post  $post
     * @return mixed
     */
    public function forceDelete(User $user, Post $post)
    {
        return $user->getAttribute('id') === $post->getAttribute('user_id');
    }
}
```
