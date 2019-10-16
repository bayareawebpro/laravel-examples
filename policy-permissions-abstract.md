```
<?php
namespace App\Policies;

use App\User;
use App\Policies\AbstractPolicy;

class DefaultPolicyTemplate extends AbstractPolicy
{
    public static $SCOPE = 'page';

    /**
     * Create a new policy instance.
     * @return void
     */
    public function __construct()
    {
        //
    }
}

```


```
<?php
namespace App\Policies;
use App\User;
use Illuminate\Auth\Access\HandlesAuthorization;
use Illuminate\Database\Eloquent\Model;
abstract class AbstractPolicy
{
    use HandlesAuthorization;

    public static $SCOPE = 'my-scope';

    /**
     * Determine whether the user can view the entity.
     * @param  User $user
     * @return bool
     */
    public function viewAny(User $user)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:view_any")
        );
    }

    /**
     * Determine whether the user can view a specific entity (show).
     * @param  User $user
     * @param  Model $model
     * @return bool
     */
    public function view(User $user, Model $model)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:view:{$model->id}")
        );
    }

    /**
     * Determine whether the user can create a new entity (create).
     * @param  User $user
     * @return bool
     */
    public function create(User $user)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:create")
        );
    }

    /**
     * Determine whether the user can update a specific entity (update).
     * @param  User $user
     * @param  Model $model
     * @return bool
     */
    public function update(User $user, Model $model)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:update_any") ||
            $user->hasPermission("{$this::$SCOPE}:update:{$model->id}")
        );
    }

    /**
     * Determine whether the user can delete a specific entity (delete).
     * @param  User $user
     * @param  Model $model
     * @return bool
     */
    public function delete(User $user, Model $model)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:delete_any") ||
            $user->hasPermission("{$this::$SCOPE}:delete:{$model->id}")
        );
    }

    /**
     * Determine whether the user can force delete a specific entity (forceDelete).
     * @param  User $user
     * @param  Model $model
     * @return bool
     */
    public function forceDelete(User $user, Model $model)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:force_delete_any") ||
            $user->hasPermission("{$this::$SCOPE}:force_delete:{$model->id}")
        );
    }

    /**
     * Determine whether the user can restore a specific entity (restore).
     * @param  User $user
     * @param  Model $model
     * @return bool
     */
    public function restore(User $user, Model $model)
    {
        /** @var $user User */
        return (
            $user->isRole('admin') ||
            $user->hasPermission("{$this::$SCOPE}:all") ||
            $user->hasPermission("{$this::$SCOPE}:restore_any") ||
            $user->hasPermission("{$this::$SCOPE}:restore:{$model->id}")
        );
    }

}

```