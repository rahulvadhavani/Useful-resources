
# Passport 

Use passport auth by using access-token and refresh-token




## Steps

1) After creating the project install the passport package.

```bash
  composer require laravel/passport
```

2) Laravel passport package comes up with some migration files for storing OAuth2 clients and access tokens. so you need to hit this command.

```bash
  php artisan migrate
```

3) Generate encryption keys.

```bash
  php artisan passport:install
```

Note:- The command will create personal access and password grant clients which will be used to generate access tokens.



4) Publish passport config file.

```bash
  php artisan vendor:publish --tag=passport-config
```

5) Open passport.php file and update it like this.

```php

    'personal_access_client' => [
        'id' => env('PASSPORT_PERSONAL_ACCESS_CLIENT_ID'),
        'secret' => env('PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET'),
    ],

    /*
    |--------------------------------------------------------------------------
    | Password Grant Client
    |--------------------------------------------------------------------------
    |
    */

    'password_grant_client' => [
        'id' => env('PASSPORT_PASSWORD_GRANT_CLIENT_ID'),
        'secret' => env('PASSPORT_PASSWORD_GRANT_CLIENT_SECRET'),
    ],

```


6) Open .env file and update these environment variables which are generated in step 3.

```
// Replace with your client_id and secret_key
PASSPORT_PERSONAL_ACCESS_CLIENT_ID=1
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET=xGhkmSlKzthLLmEhvqKGSvCPXcd0ZtujO4MJt58i

PASSPORT_PASSWORD_GRANT_CLIENT_ID=2
PASSPORT_PASSWORD_GRANT_CLIENT_SECRET=u2V6IqBraMA2aUZov3McO6DRPKg0SWDbiZjwcUSX
```


7) Update your User model.

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
```

8) Update your Auth Guard.

```
    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],
    ],
```

9) Update AuthServiceProvider

```php
 protected $policies = [
        'App\Models\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     */
    public function boot(): void
    {
        $this->registerPolicies();
        // if (!$this->app->routesAreCached()) {
        //     Passport::routes();
        // }
        Passport::ignoreRoutes();
        Passport::tokensExpireIn(now()->addDays(1));
        Passport::refreshTokensExpireIn(now()->addMonths(6));
        Passport::personalAccessTokensExpireIn(now()->addMonths(12));
    }
```

Note:- Passport::routes() add few oauth routes in our application. all routes prefix with oauth

Example:- http://example.com/oauth/token 

you can also modify this prefix

```
Passport::routes(null, ['prefix' => 'api/oauth']);
```


10)  Create the AuthController.

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Laravel\Passport\RefreshTokenRepository;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Facades\Http;
use Illuminate\Http\Request;
use App\Models\User;
use Carbon\Carbon;
class AuthController extends Controller
{

    public function register(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name' => 'required',
            'email' => 'required|email|unique:users',
            'password' => 'required'
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 400);
        }

        $user = new User();
        $user->name = $request->name;
        $user->email = $request->email;
        $user->password = bcrypt($request->password);
        $user->save();
        return response()->json(['data' => $user]);
    }

    public function login(Request $request)
    {
        $credentials = $request->only(['email', 'password']);
        $validator = Validator::make($credentials, [
            'email' => 'required|email',
            'password' => 'required'
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 400);
        }

        if (!auth()->attempt($credentials)) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        /* ------------ Create a new personal access token for the user. ------------ */
        $tokenData = auth()->user()->createToken('MyApiToken');
        $token = $tokenData->accessToken;
        $expiration = $tokenData->token->expires_at->diffInSeconds(Carbon::now());

        return response()->json([
            'access_token' => $token,
            'token_type' => 'Bearer',
            'expires_in' => $expiration
        ]);
    }

    public function getUser()
    {
        return response()->json(auth()->user());
    }

    public function logout()
    {
        $token = auth()->user()->token();

        /* --------------------------- revoke access token -------------------------- */
        $token->revoke();
        $token->delete();

        /* -------------------------- revoke refresh token -------------------------- */
        $refreshTokenRepository = app(RefreshTokenRepository::class);
        $refreshTokenRepository->revokeRefreshTokensByAccessTokenId($token->id);

        return response()->json(['message' => 'Logged out successfully']);
    }

    /* ----------------- get both access_token and refresh_token ---------------- */
    public function loginGrant(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'email' => 'required|email',
            'password' => 'required'
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 400);
        }

        $baseUrl = url('/');
        $response = Http::post("{$baseUrl}/oauth/token", [
            'username' => $request->email,
            'password' => $request->password,
            'client_id' => config('passport.password_grant_client.id'),
            'client_secret' => config('passport.password_grant_client.secret'),
            'grant_type' => 'password'
        ]);

        $result = json_decode($response->getBody(), true);
        if (!$response->ok()) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }
        return response()->json($result);
    }

    /* -------------------------- refresh access_token -------------------------- */
    public function refreshToken(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'refresh_token' => 'required'
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 400);
        }

        $baseUrl = url('/');
        $response = Http::post("{$baseUrl}/oauth/token", [
            'refresh_token' => $request->refresh_token,
            'client_id' => config('passport.password_grant_client.id'),
            'client_secret' => config('passport.password_grant_client.secret'),
            'grant_type' => 'refresh_token'
        ]);

        $result = json_decode($response->getBody(), true);
        if (!$response->ok()) {
            return response()->json(['error' => $result['error_description']], 401);
        }
        return response()->json($result);
    }
}

```


11) Go to app/Providers and open RouteServiceProvider.php. uncomment $namespace; (optional)

```
protected $namespace = 'App\\Http\\Controllers';
```



12) Go to app/Http/Middleware and open Authenticate.php. After that update redirectTo()  method.

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Auth\Middleware\Authenticate as Middleware;

class Authenticate extends Middleware
{
    /**
     * Get the path the user should be redirected to when they are not authenticated.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return string|null
     */
    protected function redirectTo($request)
    {
       return $request->expectsJson() ? null : route('login');
    }
}

```


13) Go to routes folder and update api.php

```php

<?php

use App\Http\Controllers\Api\AuthController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider and all of them will
| be assigned to the "api" middleware group. Make something great!
|
*/

Route::group(['prefix' => 'auth', 'namespace' => 'Api'], function () {
    Route::post('register', [AuthController::class, 'register']);

    /* ------------------------ For Personal Access Token ----------------------- */
    Route::post('login',    [AuthController::class, 'login']);
    /* -------------------------------------------------------------------------- */

    Route::group(['middleware' => 'auth:api'], function () {
        Route::get('logout',  [AuthController::class, 'logout']);
        Route::get('user',    [AuthController::class, 'getUser']);
    });

    /* ------------------------ For Password Grant Token ------------------------ */
    Route::post('login_grant', [AuthController::class, 'loginGrant']);
    Route::post('refresh',    [AuthController::class, 'refreshToken']);
    /* -------------------------------------------------------------------------- */
});


```
