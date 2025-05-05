
### Start sail
```console
./vendor/bin/sail start
```
or

start your application in Docker desktop by clicking on **'start'** action button.

### Compile and build frontend assets
<!-- Shell:      console, shell
Bash:       bash, sh, zsh
PowerShell: powershell, ps
DOS:        dos, bat, cmd -->

<!-- <span style="color:blue">some *blue* text</span>. -->
<!-- <p>Some Markdown text with <span style="color:blue">some <em>blue</em> text</span>.</p> -->
<!-- ```properties
 ./vendor/bin/sail npm run dev
``` -->
```console
./vendor/bin/sail npm run dev
```

# One to One

## Define `UserBio` Model and Migration

1. Create the UserBio Model
   
   > ./vendor/bin/sail artisan make:model UserBio -m

   This command creates a model file at app/Models/UserBio.php and a migration file in the database/migrations directory.

2. Open `UserBio.php` and define the properties
    ```php
    class UserBio extends Model
    {
        protected $table = 'user_bios';

        protected $fillable = [
            'user_id',
            'bio',
        ];

        protected $casts = [
            'created_at' => 'datetime',
            'updated_at' => 'datetime',
        ];
    }
    ```

    Note: The protected $casts property on a model is used to specify how attributes should be cast to native types when you access them. This can be useful for ensuring that you get the expected data type when you interact with the model attributes.
    

3. Go to migration file for the `create_user_bios_table.php` and define the schema 
    ```php
    public function up(): void
    {
        Schema::create('user_bios', function (Blueprint $table) {
            $table->id(); // Auto-incrementing primary key
            $table->unsignedBigInteger('user_id')->unique(); // Foreign key to the users table where unsignedBigInteger is used to store only positive integers
            $table->text('bio')->nullable(); 
            $table->timestamps(); // Adds created_at and updated_at columns
            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
        });
    }
    ```

4. Run a migration file
    > ./vendor/bin/sail artisan migrate

    or use this if you would like to run a specific file

    > ./vendor/bin/sail artisan migrate --path=/database/migrations/YOUR_file.php

    Tip: To view actual SQL queries before you run them: `./vendor/bin/sail artisan migrate --pretend`

5. The result should be:

    <img src="./pics/21.png" width="700" border="1"/>

## Define Laravel One to One Relationship

Since this is a one-to-one relationship, each User can have at most one UserBio. Similarly, each UserBio is associated with exactly one User.

### Method 1: Explicitly define

This method is slightly more explicit by indicating **the return type** of the relationship method:

   1. Go to `User` model and add this line

        > use Illuminate\Database\Eloquent\Relations\HasOne;

   2. Define **HasOne** relationship
   
        ```php
        public function bio(): HasOne
        {
            return $this->hasOne(UserBio::class, 'user_id');
        }
        ```

   3. Go to `UserBio` model and add this line
        > use Illuminate\Database\Eloquent\Relations\BelongsTo;


        <!-- > use Illuminate\Database\Eloquent\Factories\HasFactory; -->

   4. Define **BelongsTo** relationship (the inverse of the relationship)
   
        ```php
        public function user(): BelongsTo
        {
            return $this->belongsTo(User::class, 'user_id');
        }
        ```

### Method 2: Without Type Hinting

This is the most common and straightforward way to define a 1-to-1 relationship:
   1. Define **HasOne** relationship: Go to `User` model
   
        ```php
        public function bio()
        {
            return $this->hasOne(UserBio::class, 'user_id');
        }
        ```

   2. Define **BelongsTo** relationship: Go to `UserBio` model
   
        ```php
        public function user()
        {
            return $this->belongsTo(User::class, 'user_id');
        }
        ```

## Modify `UserController` Controller

Modify a controller to handle the user bio actions

When work with the currently authenticated user, you can use **Auth::user()** to retrieve the 'user instance'. This is particularly useful if the actions you're performing (e.g., showing, creating, editing, or deleting a user's bio) should be scoped to the currently logged-in user.

1. Open the controller file `(app/Http/Controllers/UserController.php)`
2. Check if you have include this `use Illuminate\Support\Facades\Auth;`
<!-- 3. Add this line (Make sure that required App\Models are included)
    > use App\Models\User;

    > use App\Models\UserBio; -->

3. Add the following methods:
   
   * Show the form for edit the user's bio
   
        ```php
        public function showBio()
        {
            $user = Auth::user(); // Retrieve the currently authenticated user
            $bio = $user->bio; // Access the related bio for the user
            return view('profile.show-bio', compact('user', 'bio'));
        }
        ```

        Explaination:
        * The hasOne relationship implies that each User has one corresponding UserBio. When you call `$user->bio`, Laravel automatically queries the user_bios table for the record that matches the user's id because of the `hasOne` relationship you defined in the User model (`public function bio()`)
        * The `compact('user', 'bio')` function creates an associative array with user and bio as keys and their respective values. This array is passed to the `profile.show-bio` Blade view, making the user and bio data available in the view.

   * Update the user's profile
  
        This method is designed to update the user's bio if it already exists or create a new bio if it doesn't.

        ```php
        public function updateBio(Request $request)
        {
            $user = Auth::user();
            $bio = $user->bio;
        
            $request->validate([
                'bio' => 'required|string',
            ]);
            
            if ($bio) {
                $bio->update([
                    'bio' => $request->input('bio'),
                ]); 
            } else {
                $user->bio()->create([
                    'bio' => $request->input('bio'),
                ]);
            }
        
            return redirect()->route('profile.show-bio')
                            ->with('status', 'Bio updated successfully!');
        }
        ```
        Explaination:
        * In if-else statement: If the user already has a bio, the `update` method is called to update the existing bio with the new input. If the user does not have a bio, a new UserBio record is created using the `create` method $user->bio()->create([...]), which associates the new bio with the user.
        * After updating or creating the bio, the method redirects the user back to the profile.show-bio route with a session flash message indicating that the bio was updated successfully.
  
## Define Routes

Also add this line `use App\Http\Controllers\UserController;`

```php
Route::middleware('auth')->group(function () {

    ...

    // Route to show the bio page
    Route::get('/profile/bio', [UserController::class, 'showBio'])->name('profile.show-bio');
    // Route to handle updating the bio
    Route::patch('/profile/bio', [UserController::class, 'updateBio'])->name('profile.update-bio');
});
```

Explaination:
* Route Name: `profile.show-bio`: This is a named route. In Laravel, naming routes allows you to refer to them by name rather than hardcoding URLs. You can use this name in your code to generate URLs or redirect users.

## Create or Modify Views

### `resources/views/profile/partials/update-profile-information-form.blade.php`

1. Show current bio

    ```php
    <h4 class="font-medium text-blue-900">
        {{ __('Bio Information') }}
    </h4>
    <p class="mt-2">
        {{$user->bio->bio ?? 'No bio available'}}
    </p>
    ```

   ![pic](./pics/1.png)

2. Add Manage Bio Button in `update-profile-information-form.blade.php`

    ````html
    <a href="{{ route('profile.show-bio') }}" class="inline-flex items-center px-4 py-2 bg-blue-600 border border-transparent rounded-md font-semibold text-xs text-white uppercase tracking-widest hover:bg-blue-500 active:bg-blue-700 focus:outline-none focus:border-blue-700 focus:ring ring-blue-300 disabled:opacity-25 transition ease-in-out duration-150">
        {{ __('Click to Manage Bio') }}
    </a>
    ````

### `resources/views/profile/show-bio.blade.php`

```php
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 dark:text-gray-200 leading-tight">
            {{ __('Edit Bio') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white dark:bg-gray-800 overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900 dark:text-gray-100">

                    <!-- Form to edit bio -->
                    <form method="post" action="{{ route('profile.update-bio') }}" class="mt-6 space-y-6">
                        @csrf
                        @method('patch')

                        <div>
                            <x-input-label for="bio" />
                            <textarea id="bio" name="bio" rows="5" class="mt-1 block w-full border-gray-300 rounded-md shadow-sm dark:bg-gray-700 dark:text-gray-100" required>{{ old('bio', $bio->bio ?? '') }}</textarea>
                            <x-input-error class="mt-2" :messages="$errors->get('bio')" />
                        </div>

                        <div class="flex items-center gap-4">
                            <x-primary-button>{{ __('Update Bio') }}</x-primary-button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>

    <!-- SweetAlert2 JavaScript -->
    <script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
    <script>
        document.addEventListener('DOMContentLoaded', function () {
            @if (session('status'))
                Swal.fire({
                    icon: 'success',
                    title: 'Success',
                    text: '{{ session('status') }}',
                    confirmButtonText: 'OK'
                });
            @endif
        });
    </script>
</x-app-layout>
```

![pic](./pics/2.png)

Explaination:

When you click the "Update Bio" button in the form, the form submission is handled by the action attribute of the `<form>` element, which points to a `route('profile.update-bio')`. This route is associated with a controller method that processes the form data.

```php
<form method="post" action="{{ route('profile.update-bio') }}" class="mt-6 space-y-6">
@csrf
@method('patch')
<!-- Bio textarea and submit button -->
</form>
```

<img src="./pics/15.png" alt="drawing" width="600" border="1"/>

And it will show you a success message in a dialog box.

<img src="./pics/14.png" alt="drawing" width="600" border="1"/>

## Tip: Returning to Page

If you would like the page returning to profile.edit.blade.php, you can modify function `updateBio()` in UserController. This ensures that after the bio is updated, the user is redirected back to the profile edit page, and a success message is flashed into the session.

<img src="./pics/16.png" alt="drawing" width="600" border="1"/>

Next, in the `profile.edit.blade.php` file, you can handle the session message and display the SweetAlert dialog box by adding the following script:

```html
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
<script>
    document.addEventListener('DOMContentLoaded', function() {
        @if (session('status'))
            Swal.fire({
                icon: 'success',
                title: 'Success',
                text: '{{ session('status') }}',
                confirmButtonText: 'OK'
            });
        @endif
    });
</script>
```

The result will be...

<img src="./pics/17.png" alt="drawing" width="600" border="1"/>

----
update: Jan-2025

# Lab Assignment: Route

Create two buttons:
1. **BACK**: Navigates the user back to their profile edit page upon clicking.
2. **GO TO DASHBOARD**: Redirects the user to their dashboard page when clicked.

<img src="./pics/32.png" alt="drawing" width="600" border="1"/>