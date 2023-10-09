Project overview:

A real estate management system accessible through a website and mobile app using APIs,, allowing users to interact with real estate properties, make reservations, leave reviews, and manage their bookings.

The application supports multiple user roles, including guests, hosts, and administrators. Each role has specific privileges and access levels, ensuring a secure and personalized experience for all users involved in the real estate ecosystem.

The application streamlines the management of real estate properties, allowing property owners to easily list their properties, manage reservations, and track availability. This simplifies the process of handling multiple properties and enables efficient management of bookings.

The project offers a reservation system, enabling guests to search for available properties, book accommodations. Reservation details including start and end dates, costs, discounts, and additional services, enhancing the overall booking experience.

The application includes a robust review and rating system, empowering users to share their experiences and feedback about the properties they have stayed in.

The project incorporates a sophisticated coupon and discount management system. Property owners can create enticing promotional offers to attract more bookings and boost revenue. This feature not only benefits property owners but also adds value for guests, making their stays more affordable and enjoyable.



> :warning: **Warning:** This contents below ↓ contains just parts of my code.
>                        You can access my full project files by clone it from my GitLab repository
>                        (requires asking for my permissions  to grant you access to it):
>                        https://gitlab.com/skandar.s1998/neoma 

## Contents
(contains descriptive parts of my code)

[Tables and relations](#tables-and-relations)

[Project actions and progress(graph)](#project-graph)

[Authentication (register & login)](#authentication)

- [Swagger(API documentation)](#Swagger)
    - [l5-Swagger Definition](#l5-Swagger-definition)
    - [l5-Swagger Usage](#l5-swagger-usage)
    - [l5-Swagger Configuration](#l5-swagger-configuration)
    - [l5-Swagger Example](#l5-swagger-example)
[Show specific realestate function](#show-real-estates)

[Store offer function](#store-offer)

[Offer resource](#offer-resource)

[Add real estate to favorite](#favorite)

[Store reservation function](#reservation)

### **tables-and-relations**

![Logo](/images/tables.png)

For more details about the content of the tables <a href="/bouyout.pdf" target="_blank">Click here</a>

[🔝 Back to contents](#contents)

### **project-graph**

This graph diagram represents the actions and progress for the project.

![App Logo](/images/graph(1).png)
![App Logo](/images/graph(2).png)
![App Logo](/images/graph(3).png)

[🔝 Back to contents](#contents)

### **authentication**

`routes\api.php`

```php
Route::post('register', [AuthController::class, 'register']);
Route::post('login', [AuthController::class, 'login']);
```

`app\Http\Controllers\AuthController.php`

### constructor method

```php
public function __construct()
{
    $this->middleware('auth:api')->only(['logout']);
}
```

The constructor of this controller class is responsible for setting up the middleware for authentication using Laravel Sanctum. This middleware ensures that the user is authenticated using the Sanctum authentication guard before accessing the methods. The line `$this->middleware('auth:api')->only(['logout']);` specifies that the `auth:api` middleware should be applied only to the `logout` method.

### register method

```php
public function register(Request $request)
{
    $request->validate([
        'name'           => ['required', 'string'],
        'email'          => ['required', 'string', 'email'],
        'phone_code'     => ['required', 'exists:countries,phone_code'],
        'phone'          => ['required', 'numeric'],
        'code'           => ['required', 'numeric', 'digits:4'],
    ]);

    $verification_code = VerificationCode::where([
        ['phone', $request->phone],
        ['phone_code', $request->phone_code],
    ])->first();

    if ($verification_code) {
        if ($verification_code->created_at > Carbon::now()->addMinutes(env('CODE_EXPIRATION'))) {
            $verification_code->delete();
            throw ValidationException::withMessages(['code' => __('The verification code is expired.', ['attribute' => 'code'])]);
        } elseif ($verification_code->code != $request->code) {
            throw ValidationException::withMessages(['code' => __('The Code is incorrect.', ['attribute' => 'code'])]);
        }
    }

    $user = User::where([
        ['phone', $request->phone],
        ['phone_code', $request->phone_code],
    ])->first();

    if ($user) {
        throw ValidationException::withMessages(['phone' => __('The User already exist.', ['attribute' => 'phone'])]);
    }

    $user = User::create([
        'name'             => $request->name,
        'email'            => $request->email,
        'phone_code'       => $request->phone_code,
        'phone'            => $request->phone,
        'balance'          => 0,
        'role'             => 'user',
    ]);

    $token = $user->createToken('Sanctum', [])->plainTextToken;
    return response()->json([
        'user' => new UserResource($user),
        'token' => $token,
    ], 200);
}
```

The register method in the AuthController class handles the registration of a new user with additional features like phone verification. Here is a detailed description of the code:

1. The incoming request data is validated using the `validate` method. The validation rules are defined for the `name`, `email`, `phone_code`, `phone`, and `code` fields. These rules ensure that the fields are required, have the correct data types, and meet certain conditions (e.g., existence in the `countries` table, numeric values, specific digits for the code).
3. The code checks if a verification code exists for the provided phone and phone code by querying the `VerificationCode` model.
4. If a verification code is found, the code checks if it has expired by comparing its creation time with the current time. If it has expired, the code is deleted, and a validation exception is thrown with an appropriate error message.
5. If the verification code is not expired, the code compares it with the provided code. If they don't match, a validation exception is thrown with an error message.
6. Next, the code checks if a user with the provided phone and phone code already exists by querying the `User` model.
7. If a user with the provided phone and phone code is found, a validation exception is thrown with an error message indicating that the user already exists.
8. After passing the validation checks, a new user record is created using the `create` method on the `User` model. The provided name, email, phone code, phone number, balance, and role are stored in the respective columns of the `users` table.
9. Finally, a token is generated for API authentication using Laravel Sanctum's `createToken` method. A JSON response is returned with the user resource, formatted using the `UserResource` class, and the token.

### login method

```php
public function login(Request $request)
{
    $request->validate([
        'phone_code'     => ['required', 'exists:countries,phone_code'],
        'phone'          => ['required', 'numeric'],
        'code'           => ['required', 'numeric', 'digits:4'],
    ]);

    $verification_code = VerificationCode::where([
        ['phone', $request->phone],
        ['phone_code', $request->phone_code],
    ])->first();

    if ($verification_code) {
        if ($verification_code->created_at > Carbon::now()->addMinutes(env('CODE_EXPIRATION'))) {
            $verification_code->delete();
            throw ValidationException::withMessages(['code' => __('The verification code is expired.', ['attribute' => 'code'])]);
        } elseif ($verification_code->code != $request->code) {
            throw ValidationException::withMessages(['code' => __('The Code is incorrect.', ['attribute' => 'code'])]);
        }
    }

    $user = User::where([
        ['phone', $request->phone],
        ['phone_code', $request->phone_code],
    ])->first();
    if ($user) {
        Auth::login($user);
        $token = $user->createToken('Sanctum', [])->plainTextToken;
        if ($request->remember && !$user->remember_token) {
            $user->remember_token = Str::random(40);
            $user->save();
        }
        return response()->json([
            'user' => new UserResource($user),
            'remember_token' => $user->remember_token,
            'token' => $token,
        ], 200);
    }
    throw ValidationException::withMessages(['phone' => __('User phone is incorrect.', ['attribute' => 'phone'])]);
}
```

The `login` method in the given code is responsible for authenticating a user based on their phone number and verification code. Here's a breakdown of how the code works:

1. The incoming request data is validated using the `validate` method. The validation rules are defined for the `phone_code`, `phone`, and `code` fields. These rules ensure that the fields are required, have the correct data types, and meet certain conditions (e.g., existence in the `countries` table, numeric values, specific digits for the code).
3. The code checks if a verification code exists for the provided phone and phone code by querying the `VerificationCode` model.
4. If a verification code is found, the code performs additional checks:
   - It verifies if the verification code's `created_at` timestamp is within the allowed time frame, specified by the `CODE_EXPIRATION` environment variable. If the code is expired, it is deleted, and a validation exception is thrown with an error message.
   - It compares the verification code with the one provided in the request. If they don't match, a validation exception is thrown with an error message.
5. The code then finds a user with the provided phone and phone code by querying the `User` model.
6. If a user is found, the code performs the following tasks:
   - It logs in the user using Laravel's `Auth::login` method.
   - It generates a token for API authentication using Laravel Sanctum's `createToken` method. The token is associated with the user and named 'Sanctum'. An empty array is passed as the second argument to specify that no specific abilities are required for the token.
   - It checks if the "remember me" option is enabled (`$request->remember`) and if the user doesn't already have a remember token. If both conditions are met, a remember token is generated and saved for the user.
   - It returns a JSON response with the user resource represented by the `UserResource` class, the remember token, and the token.
7. If there is no user with the provided phone and phone code, a validation exception is thrown with an error message indicating that the user's phone is incorrect.

[🔝 Back to contents](#contents)

### **l5-swagger-definition**

This project integrates Swagger documentation using the L5 Swagger package. Swagger is an open-source framework that allows to design, build, document, and consume RESTful APIs. With it, I easily generated interactive API documentation for my project.

![Swagger](/images/swagger.png)

l5-swagger benefits:
- Documentation: Generate comprehensive API documentation automatically, saving time and effort in writing and maintaining
  documentation manually.
- API Testing: The Swagger UI provides a user-friendly interface for testing API endpoints, allowing to send requests,
  view responses, and verify the functionality of APIs.

[🔝 Back to contents](#contents)

### **l5-swagger-usage**

- Install the package by running the following command:
  ```cmd
  composer require darkaonline/l5-swagger
  ```
- Configure the package by modifying the [config/l5-swagger.php](#l5-swagger-configuration).
  This configuration file allows to specify various settings such as the API title, routes, file paths, security definitions.

- Generate OpenAPI documentation for RESTful APIs that defined, and add Swagger-PHP annotations
  depending on https://zircote.github.io/swagger-php/ website that provides documentation and resources for the Swagger-PHP library.
   
- Run the following command to generate the Swagger documentation: 
  ```cmd
  php artisan l5-swagger:generate
  ```
- Access the Swagger UI by visiting the /api/documentation route in web browser. As the photo illustrates [Here](#swagger),
  you can explore and  interact with your API endpoints, view documentation, and test API requests.

- l5-swagger example in my code:
  [Update profile functionality](#swagger-update-profile)

[🔝 Back to contents](#contents)

### **l5-swagger-configuration**

`config\l5-swagger.php`

This file is a configuration file for the L5 Swagger package, which is used to generate API documentation. It provides settings and options to customize the behavior and appearance of the Swagger UI, as well as specify the location of the API annotations and generated documentation files.

```php
<?php
return [
    'default' => 'default',
    'documentations' => [
        'default' => [
            'api' => [
                'title' => 'L5 Swagger UI',
            ],

            'routes' => [
                /*
                 * Route for accessing api documentation interface
                */
                'api' => 'api/documentation',
            ],
            .
            .
            .
            .
        ],

        /*
         * API security definitions. Will be generated into documentation file.
        */
        'securityDefinitions' => [
            'securitySchemes' => [
                /*
                 * Examples of Security schemes
                */
                'bearer_token' => [ // Unique name of security
                    'type' => 'apiKey', // Valid values are "basic", "apiKey" or "oauth2".
                    'description' => 'Enter token in format (Bearer <token>)',
                    'name' => 'Authorization', // The name of the header or query parameter to be used.
                    'in' => 'header', // The location of the API key. Valid values are "query" or "header".
                ],

            ],
          .
          .
          .
    ],
    ]
];
```
The 'api' key under the 'default' group specifies the API title to be displayed in the Swagger UI.

The 'routes' key under the 'default' group defines the routes used to access the API documentation interface.
The 'api' key under the 'routes' key specifies the route for accessing the API documentation interface.

The 'annotations' key specifies the absolute paths to directories containing the Swagger annotations.


Under the 'paths' key within 'defaults', you can specify the absolute paths for storing parsed annotations and exporting views.
The 'base' key defines the API's base path.

The 'securityDefinitions' key defines security schemes and securities for the API documentation.
The 'securitySchemes' key under 'securityDefinitions' specifies security schemes, such as 'bearer_token'.
The 'security' key under 'securityDefinitions' allows you to define security examples.

this part:
```php
        'bearer_token' => [ // Unique name of security
            'type' => 'apiKey', // Valid values are "basic", "apiKey" or "oauth2".
            'description' => 'Enter token in format (Bearer <token>)',
            'name' => 'Authorization', // The name of the header or query parameter to be used.
            'in' => 'header', // The location of the API key. Valid values are "query" or "header".
        ],
```
Is show at the top right of swagger UI interface. It responsable to authenticate API requests using a bearer token.

'type' => 'apiKey': This specifies the type of security scheme, which in this case is an API key. Other valid values for the type are 'basic' for basic authentication and 'oauth2' for OAuth2 authentication.

'description' => 'Enter token in format (Bearer <token>)': This provides a description of how the token should be provided. It instructs the user to enter the token in the format "Bearer <token>", indicating that the word "Bearer" should be included before the actual token value.

'name' => 'Authorization': This specifies the name of the header or query parameter that should be used to send the API key. In this case, the key is expected to be sent in the 'Authorization' header.

'in' => 'header': This indicates the location of the API key, which is in the header of the API request.

### **l5-swagger-annotations-in-Controller**

`app\Http\Controllers\Controller.php:`

```php
/**
 * @OA\Info(title="API TICKETS", version="1.0")
 *
 *  @OA\Server(
 *      url="https://biut.rewardszone.net/api",
 *  )
 *  @OA\Server(
 *      url="http://127.0.0.1:8000/api",
 *  )
 *
 * @OAS\SecurityScheme(
 *      securityScheme="bearer_token",
 *      type="http",
 *      scheme="bearer"
 * )
*/

class Controller extends BaseController
{
    use AuthorizesRequests, ValidatesRequests;
}
```
Swagger annotations provided using the @OA and @OAS tags. These annotations are used by the L5 Swagger package to generate API documentation.
The annotations are used to document the API endpoints and provide additional information such as request and response schemas, parameters, and authentication requirements.

- `@OA\Info`: This annotation is used to provide general information about the API. It includes the title and version of the API.

- `@OA\Server`: This annotation defines the server URLs for the API.
   There are two server URLs specified in this case:
   - The first server URL is "https://biut.rewardszone.net/api".
   - The second server URL is "http://127.0.0.1:8000/api". This is a local development server URL.

- `@OAS\SecurityScheme`: This annotation defines a security scheme for the API. In this case, it specifies a bearer token authentication scheme. The `securityScheme` attribute is set to "bearer_token", and the type is set to "http" with the scheme "bearer".

[🔝 Back to contents](#contents)
### **l5-swagger-example**

![Logo](/images/swagger-example.png)

`app\Http\Controllers\AuthController.php`

```php
/**
 * @OA\Post(
    * path="/user",
    * description="Edit your profile",
    *  tags={"Auth"},
    *  security={{"bearer_token": {} }},
    *   @OA\RequestBody(
    *       required=true,
    *       @OA\MediaType(
    *           mediaType="multipart/form-data",
    *           @OA\Schema(
    *              required={"name","email","birth","id_number","type"},
    *              @OA\Property(property="name", type="string"),
    *              @OA\Property(property="email", type="email"),
    *              @OA\Property(property="birth", type="string"),
    *              @OA\Property(property="id_number", type="string"),
    *              @OA\Property(property="type", type="string",enum={"citizen","resident","tourist"}),
    *              @OA\Property(property="nationality_id", type="integer"),
    *              @OA\Property(property="image", type="file"),
    *              @OA\Property(property="delete_image", type="boolean",enum={"1","0"}),
    *              @OA\Property(property="_method", type="string", format="string", example="PUT"),
    *           )
    *       )
    *   ),
    *     @OA\Response(
    *         response="200",
    *    description="Success"
    *     ),
    * )
*/

public function update_my_profile(Request $request)
{
    $request->validate([
        'name'            => ['required','string'],
        'email'           => ['required','email',Rule::unique('users')->ignore(to_user(Auth::user())->id)],
        'birth'           => ['required','string'],
        'id_number'       => ['required','string'],
        'type'            => ['required','in:citizen,resident,tourist'],
        'nationality_id'  => ['required_if:type,resident','required_if:type,tourist','exists:countries,id'],
        'image'           => ['image'],
        'delete_image'    => ['boolean'],

    ]);

    $user = to_user(Auth::user());
    if($request->type == 'citizen')
        $nationality_id = 193;
    else
        $nationality_id = $request->nationality_id;

    $image = $user->image;
    if($request->delete_image)
    {
        delete_file_if_exist($image);
        $image = null;
    }
    else if($request->hasFile('image'))
    {
        delete_file_if_exist($image);
        $image = upload_file($request->image,'profile_image','profile_images');
    }

    $user->update([
        'name'            => $request->name,
        'email'           => $request->email,
        'code'            => $request->code,
        'birth'           => $request->birth,
        'id_number'       => $request->id_number,
        'type'            => $request->type,
        'nationality_id'  => $nationality_id,
        'image'           => $image,
    ]);

    $_user = (new UserResource($user))->toArray($request);
    $_user['reservations_count'] = $user->reservations_count();
    $_user['user_rate'] = $user->rate;
    $_user['count_of_blockers'] = $user->count_of_blockers();
    return response()->json($_user,200);

}
```
The code between /** */ describes a specific POST API endpoint for editing the user's profile depending on l5-swagger documentation.

Here is a breakdown of the annotations used in this code:

- @OA\Post: This annotation indicates that this API endpoint is an HTTP POST request.

- path="/user": This specifies the URL path for this API endpoint, which is /user.
  The full URL for this endpoint would depend on the base URL that I defined in [app\Http\Controllers\Controller.php](#l5-swagger-annotations-in-Controller) file.

- description="Edit your profile": This provides a brief description of the purpose of this API endpoint,
  which is to edit the user's profile.

- tags={"Auth"}: This assigns the API endpoint to a specific tag or category. In this case,
  it is associated with the "Auth" tag, which typically indicates authentication-related endpoints.

- security={{"bearer_token": {} }}: This specifies the security requirement for this API endpoint.
  It indicates that the bearer_token security scheme, which was defined earlier in [configuration file](#l5-swagger.php), should be applied to this endpoint. This means that the user needs to provide a valid bearer token in the request header for authentication.

- @OA\RequestBody: This annotation indicates that the API endpoint expects a request body containing data.

- required=true: This specifies that the request body is required for this API endpoint.

- @OA\MediaType: This annotation specifies the media type of the request body, which is multipart/form-data.
  This indicates that the request body may contain form data.

- @OA\Schema: This defines the schema or structure of the request body.

- required={"name","email","birth","id_number","type"}: This specifies that the properties name, email, birth, id_number, and type
  are required in the request body.

- @OA\Property: These annotations define the properties of the request body schema. Each property is specified with its name, type, 
  and additional constraints. For example, name is of type string, email is of type email, and type is of type string with an enum constraint allowing values of "citizen", "resident", or "tourist".

- @OA\Response: This annotation describes the response that the API endpoint will return.

- response="200": This indicates that the response has a status code of 200, which typically represents a successful request.

- description="Success": This provides a brief description of the response, indicating that it represents a successful operation.

This function called from this route (with PUT method) that I defined into *routes\api.php* file:
```php
Route::put('/user', [AuthController::class, 'update_my_profile']);
```

[🔝 Back to contents](#contents)

### **show-real-estates**

`app\Http\Controllers\RealEstateController.php`

```php
public function index(Request $request)
{
    $request->validate([
        'cities_ids'             => ['array'],
        'cities_ids.*'           => ['exists:cities,id'],
        'avenue_ids'             => ['array'],
        'avenue_ids.*'           => ['exists:avenues,id'],
        'categories_ids'         => ['array'],
        'categories_ids.*'       => ['exists:categories,id'],
        'customers_types_ids'    => ['array'],
        'customers_types_ids.*'  => ['exists:customer_types,id'],
        'min_price'              => ['numeric'],
        'max_price'              => ['numeric'],
        'direction'              => ['array'],
        'reservation_start_date' => ['date'],
        'reservation_end_date'   => ['date'],
        'min_review'             => ['integer', 'min:6'],
        'without_insurance_only' => ['boolean'],
        'available_only'         => ['boolean'],
        'offers_only'            => ['boolean'],
        'order_by'               => ['string', 'in:nearest,most_viewed,top_reviewed,min_price,max_price'],
        'per_page'               => ['integer', 'min:1'],
    ]);

    $per_page = $request->per_page ?? 10;
    $q = RealEstate::with(['avenue', 'avenue.city', 'category', 'media', 'customer_type', 'host', 'facilities', 'offer'])->where('available', 1);
    if ($request->order_by == "nearest" && $request->user_lat && $request->user_lng)
        $q->select(DB::raw("*, (`lat`-{$request->user_lat})*(`lat`-{$request->user_lat})+(`lng`-{$request->user_lng})*(`lng`-{$request->user_lng}) AS `distance`"));
    if ($request->cities_ids)
        $q->whereIn('city_id', $request->cities_ids);
    if ($request->avenue_ids)
        $q->whereIn('avenue_id', $request->avenue_ids);
    if ($request->categories_ids)
        $q->whereIn('category_id', $request->categories_ids);
    if ($request->customers_types_ids)
        $q->whereIn('customer_type_id', $request->customers_types_ids);
    if ($request->direction)
        $q->where('direction', $request->direction);
    if ($request->min_review)
        $q->where('reviews_avg', '>=', $request->min_review);

    if ($request->order_by == "nearest" && $request->user_lat && $request->user_lng)
        $q->orderBy('distance', 'asc');
    elseif ($request->order_by == 'most_viewed')
        $q->orderBy('views_count', 'desc');
    elseif ($request->order_by == 'top_reviewed')
        $q->orderBy('reviews_avg', 'desc');
    $_realEstates = $q->paginate($per_page);
    $realEstates = $_realEstates;

    if ($request->offers_only) {
        $_realEstates = [];
        foreach ($realEstates as $real_estate) {
            if ($real_estate->offer != null)
                $_realEstates[] = $real_estate;
        }
        $realEstates = $_realEstates;
    }

    return RealEstateResource::collection($realEstates);
}
```

The `index()` function is responsible for retrieving a list of real estate properties based on the provided search parameters.

The function starts by validating the search parameters using various validation rules.

The base query, `$q`, is used to fetch real estate properties along with their related data, including avenue, city, category, media, customer type, host, facilities, and offer. The query includes a condition to retrieve only available properties (`where('available', 1)`).

### Search Criteria
The function checks the provided search parameters and applies additional conditions to the query accordingly, using if conditional statements. For example, if the `order_by` parameter is set to "nearest" and user latitude and longitude are provided, the query calculates the distance between the real estate's location and the user's location and assigns it to a distance alias.

### Sorting
The query applies sorting based on the `order_by` parameter. For example, if it is "nearest" and user latitude and longitude are provided, the results are sorted by the distance in ascending order.

### Pagination
The query results are paginated using the `paginate` method with the specified number of items per page. The paginated results are stored in `$_realEstates` and assigned to `$realEstates`.

### Filtering by Offers
If the `offers_only` parameter is set to true, the function filters the `$realEstates` collection to include only the real estate properties that have an associated offer.


This function is called from the route `real_estates` with the GET method, which is defined in the `routes\api.php` file as follows:
```php
Route::apiResource('real_estates', RealEstateController::class);
```

[🔝 Back to contents](#contents)

### **store-offer**

`app\Http\Controllers\OfferController.php`

```php
public function store(Request $request)
{
    $request->validate([
        'name'           => ['array', translation_rule()],
        'real_estate_id' => ['required','exists:real_estates,id'],
        'percent'        => ['required','integer','between:1,100'],
        'start_date'     => ['required','date'],
        'end_date'       => ['required','date','required_with:start_date'],
        'week_days'      => ['required','array'],
        'week_days.*'    => ['integer','between:0,6'],
        'enabled'        => ['required','boolean'],
    ]);

    $user = to_user(Auth::user());
    $real_estate = RealEstate::find($request->real_estate_id);
    if ($real_estate->host_id != $user->id)
        throw new BadRequestException('This real estate dose not belongs to you!');
    if(Offer::where('real_estate_id', $request->real_estate_id)->where('enabled', 1)->exists())
        throw new BadRequestException('Cannot to enable two offers in the same time');

    $week_days = array_unique($request->week_days);
    sort($week_days);

    $offer = new Offer([
        'name'           => $request->name,
        'real_estate_id' => $request->real_estate_id,
        'percent'        => $request->percent,
        'start_date'     => $request->start_date,
        'end_date'       => $request->end_date,
        'week_days'      => implode(',', $week_days),
        'enabled'        => $request->enabled,
    ]);
    $offer->save();
    return response()->json(new OfferResource($offer), 201);
}
```

The `store()` method is responsible for creating and saving a new offer for a real estate. It performs validation, ownership checks, and prepares the week days for the offer.

### Ownership Check
It checks whether the authenticated user is the owner of the real estate by comparing their IDs. If the owner check fails, a `BadRequestException` is thrown with the message "This real estate does not belong to you!".

### Existing Offer Check
It checks whether there is already an enabled offer for the same real estate. This is done by querying the Offer model with conditions on `real_estate_id` and `enabled` fields. If such an offer exists, a `BadRequestException` is thrown with the message "Cannot enable two offers at the same time".

### Preparing Week Days
Duplicate values are removed from the `week_days` array using `array_unique()`. The array is then sorted in ascending order using `sort()`.

### Creating and Saving Offer
A new Offer object is created with the provided data, including the name, real_estate_id, percent, start_date, end_date, week_days, and enabled values. The offer is then saved to the database.

This function is called from the route `offers` with the POST method, which is defined in the `routes\api.php` file as follows:
```php
Route::apiResource('offers', RealEstateController::class);
```

[🔝 Back to contents](#contents)

### **offer-resource**

`app\Http\Resources\OfferResource.php`

```php
class OfferResource extends JsonResource
{
    private $with_real_estate;

    public function __construct($resource, $with_real_estate=true)
    {
        parent::__construct($resource);
        $this->with_real_estate = $with_real_estate;
    }

    public function toArray(Request $request): array
    {
        return [
            'id'              => $this->id,
            'name'            => $this->name,
            'real_estate_id'  => $this->real_estate_id,
            'percent'         => $this->percent,
            'start_date'      => $this->start_date,
            'end_date'        => $this->end_date,
            'week_days'       => explode(',', $this->week_days),
            'enabled'         => $this->enabled,
            'real_estate'     => $this->with_real_estate ? new LiteRealEstateResource($this->real_estate) : null,
        ];
    }
}
```

The `OfferResource` class is responsible for formatting offer data into an array structure using the `toArray` method.

- ID: The unique identifier of the offer.
- Name: The name of the offer.
- Real Estate ID: The ID of the associated real estate.
- Percent: The percentage value of the offer.
- Dates: The start and end dates of the offer.
- Week Days: The days of the week when the offer is valid.
- Enabled Status: Indicates whether the offer is currently enabled or not.

Additionally, the class provides the option to include the associated real estate information based on the value of the `$with_real_estate` parameter.

[🔝 Back to contents](#contents)

### **favorite**

`app\Http\Controllers\FavoriteController.php`

```php
public function store(Request $request)
{
    $request->validate([
        'real_estate_id' => ['exists:real_estates,id'],
    ]);

    $user = Auth::user();
    to_user($user)->favorites()->attach($request->real_estate_id);
}
```
The `store` method validates the request parameter to ensure the existence of the *real_estate_id* in the real_estates table. It then retrieves the authenticated user and adds the corresponding real estate to the user's favorites by attaching it to the favorites relationship.

### Adding Real Estate to Favorites
The [favorites() method](#favorites) is then called on the *user* model to access the favorites relationship.
The `attach()` method is used to attach the provided real_estate_id to the user's favorites.

This function called from this route (with POST method) that I defined into *routes\api.php* file:
```php
Route::post('favorites', [FavoriteController::class, 'store'])->name('favorites.store');
```
### **favorite**

`app\Models\User.php`

```php
.
.
public function favorites()
{
    return $this->belongsToMany(RealEstate::class,'favorites' ,'user_id','real_estate_id');
}
.
.

```
The `favorites()` method defines a many-to-many relationship between the current model (presumably a User model) and the RealEstate model. It specifies that a user can have multiple favorite real estates, and a real estate can be favorited by multiple users.

The `belongsToMany()` method is used to define this relationship and takes four parameters:

1. The first parameter, `RealEstate::class`, specifies the related model, indicating that the relationship is with the RealEstate model.

2. The second parameter, `'favorites'`, specifies the name of the intermediate table that stores the relationship between users and real estates. This table serves as a bridge between the User and RealEstate models in the many-to-many relationship.

3. The third parameter, `'user_id'`, specifies the foreign key column name in the intermediate table that references the users table. This column represents the foreign key of the User model.

4. The fourth parameter, `'real_estate_id'`, specifies the foreign key column name in the intermediate table that references the real_estates table. This column represents the foreign key of the RealEstate model.

By defining this relationship, I can access a user's favorite real estates using the `favorites` method, which returns a collection of RealEstate models associated with the user.

[🔝 Back to contents](#contents)

### **reservation**

`app\Http\Controllers\ReservationController.php`

```php
public function store(Request $request)
{
    $request->validate([
        'real_estate_id'   => ['required', 'exists:real_estates,id'],
        'start_date'       => ['required', 'date', 'after:today'],
        'end_date'         => ['required', 'date', 'after:start_date'],
        'coupon'           => ['string'],
        'pay_all'          => ['required', 'boolean'],
    ]);

    $status = $this->possibilityCheck($request);
    if (!$status)
        throw new BadRequestException('Cannot reservation right now,the real estate is reserved!');

    $real_estate = RealEstate::find($request->real_estate_id);

    $user = to_user(Auth::user());
    if($user->id != $real_estate->host_id){
        $guest  = $user->id;
        $source = 'G';
    } else {
        $guest = null;
        $source = 'R';
    }

    // Calculate Days
    $start_date = new DateTime($request->start_date);
    $end_date = new DateTime($request->end_date);

    $days = $start_date->diff($end_date)->format('%a');
    $start_date_ind = 0;
    $start_date_dow = $start_date->format('w');
    $end_date_ind = $days;

    $offer = Offer::where('real_estate_id',$real_estate->id)->where('enabled',true)->first();
    if($offer)
    {
        $offer_start_date = new DateTime($offer->start_date);
        $offer_end_date = new DateTime($offer->end_date);
        $offer_start_date_ind = (int)$start_date->diff($offer_start_date)->format('%a');
        if($start_date > $offer_start_date)
            $offer_start_date_ind *= -1;
        $offer_end_date_ind = (int)$start_date->diff($offer_end_date)->format('%a');
        if($start_date > $offer_end_date)
            $offer_end_date_ind *= -1;
    }

    if ($days < 7)
        $cost_per_dow = [
            $real_estate->price_per_day,
            $real_estate->price_per_day,
            $real_estate->price_per_day,
            $real_estate->price_per_day,
            $real_estate->price_thursday,
            $real_estate->price_friday,
            $real_estate->price_saturday,
        ];
    elseif ($days < 30)
        $cost_per_dow = [
            $real_estate->price_per_week/7,
            $real_estate->price_per_week/7,
            $real_estate->price_per_week/7,
            $real_estate->price_per_week/7,
            $real_estate->price_per_week/7,
            $real_estate->price_per_week/7,
            $real_estate->price_per_week/7,
        ];
    else
        $cost_per_dow = [
            $real_estate->price_per_month/30,
            $real_estate->price_per_month/30,
            $real_estate->price_per_month/30,
            $real_estate->price_per_month/30,
            $real_estate->price_per_month/30,
            $real_estate->price_per_month/30,
            $real_estate->price_per_month/30,
        ];

    $cost = 0;
    $offer_discount = 0;
    for ($i=$start_date_ind; $i < $end_date_ind; $i++) {
        $dow = ($i + $start_date_dow) % 7;
        $day_cost = $cost_per_dow[$dow];
        $cost += $day_cost;
        if($offer && $i >= $offer_start_date_ind && $i <= $offer_end_date_ind && in_array($dow, explode(',', $offer->week_days)))
            $offer_discount += $day_cost * $offer->percent / 100.0;
    }

    // Calculate Coupon Discount
    $coupon_id = null;
    $coupon_percent = 0;
    if($request->coupon){
        $valid_coupon = true;

        $coupon = Coupon::where('code',$request->coupon)->first();
        if($coupon)
            $real_estate_ids = RealEstateCoupon::where('coupon_id' , $coupon->id)->pluck('real_estate_id');

        if(!$coupon || !in_array($request->real_estate_id,$real_estate_ids))
            $valid_coupon = false;
        elseif ($request->start_date > $coupon->expire_date)
            $valid_coupon = false;
        elseif ($coupon->enabled != 1)
            $valid_coupon = false;
        elseif ($coupon->uses >= $coupon->max_uses)
            $valid_coupon = false;
        else
            $coupon_percent = $coupon->percent;

        if(!$valid_coupon)
            throw new ValidationException(['coupon' => 'The coupon is invalid']);
        $coupon_id = $coupon->id;
    }
    $coupon_discount = ($cost - $offer_discount) * $coupon_percent / 100;
    // Calculate Subtotal
    $subtotal_cost = $cost - $offer_discount - $coupon_discount ;
    // Calculate Tax
    $tax = $cost * 0.15;
    // Calculate Fee
    $fee = $subtotal_cost * 0.15;
    // Calculate Total Cost
    $total_cost = $subtotal_cost + $tax + $fee;
    // Calculate Pre Paid
    $pre_paid = 0 ;
    if($request->pay_all)
        $pre_paid = $total_cost;
    else
        $pre_paid = $total_cost * $real_estate->pre_paid_percent / 100 ;

    $reservation = new Reservation([
        'guest_id'         => $guest,
        'host_id'          => $real_estate->host_id,
        'real_estate_id'   => $request->real_estate_id,
        'coupon_id'        => $coupon_id,
        'start_date'       => $request->start_date,
        'end_date'         => $request->end_date,
        'status'           => 'in_progress',
        'cost'             => $cost,
        'coupon_discount'  => $coupon_discount,
        'offer_discount'   => $offer_discount,
        'insurance'        => $real_estate->insurance,
        'tax'              => $tax,
        'total_cost'       => $total_cost,
        'fee'              => $fee,
        'source'           => $source,
        'pre_paid'         => $pre_paid,
    ]);
    $reservation->save();

    return response()->json(new ReservationResource($reservation), 201);
}
```
### **possibilityCheck**

```php
public function possibilityCheck(Request $request){
    $status = true ;

    $reserved_real_estates = Reservation::where('real_estate_id',$request->real_estate_id)
        ->orderBy('start_date' , 'asc')->get();

    $start_real_estate = Reservation::where('real_estate_id',$request->real_estate_id)->where('start_date','<',$request->start_date)
        ->orderBy('start_date' , 'desc')->first();

    $end_real_estate = Reservation::where('real_estate_id',$request->real_estate_id)->where('start_date','>',$request->start_date)
        ->orderBy('start_date' , 'asc')->first();


    if( !$reserved_real_estates ){
        $status = true ;
    } elseif (count($reserved_real_estates) == 1){
        if ($request->start_date > $reserved_real_estates[0]->end_date or $request->end_date < $reserved_real_estates[0]->start_date)
            $status = true ;
        else
            $status = false ;
    } elseif(count($reserved_real_estates) > 1){
        if (($request->end_date > $end_real_estate->start_date) or ($request->start_date < $start_real_estate->end_date))
            return false ;
    }

    return $status;
}
```

The `store` function is responsible for handling the creation of a reservation based on the provided data in the `$request` object.

- The function starts by validating the data in the `$request` object using the `validate` method. It checks if the required fields (`real_estate_id`, `start_date`, `end_date`, and `pay_all`) are present and meet the specified validation rules.

- Possibility Check: The function calls the `possibilityCheck` method, passing the `$request` object. This method is defined outside the `store` function and is responsible for checking if it's possible to make a reservation for the given real estate based on the provided start and end dates. It returns a boolean value indicating the possibility.

- The code retrieves the `RealEstate` object corresponding to the `real_estate_id` provided in the `$request` object.
### User Identification
The code determines whether the logged-in user is a guest or the host based on the `host_id` of the `RealEstate` object and the authenticated user's ID. If the user is not the host, they are considered a guest, and the `guest` variable is set to their user ID. Otherwise, if the user is the host, the `guest` variable is set to `null`, and the `source` is set to `'R'` (indicating the reservation is made by the host).

### Calculation of Reservation Days
The code calculates the number of days between the start and end dates provided in the `$request` object using the `DateTime` class.

### Offer Calculation
It checks if there is an active offer (`Offer`) for the given real estate. If an offer exists, it calculates the number of days between the start date of the reservation and the offer's start and end dates.

### Cost Calculation
Based on the number of days (`$days`), the code determines the cost per day of the reservation. The cost depends on the duration of the reservation: less than 7 days, less than 30 days, or 30 days or more.

### Total Cost Calculation
The code calculates the total cost of the reservation by iterating over each day of the reservation and calculating the cost per day based on the day of the week (`$dow`). It also applies any offer discounts if applicable.

### Coupon Validation and Discount Calculation
The code then checks if a coupon is provided in the `$request` object. If a coupon is provided, it validates the coupon and checks if it is applicable to the selected real estate. If the coupon is invalid or not applicable, a `ValidationException` is thrown. If a valid coupon is provided, the code calculates the coupon discount based on the total cost (after applying the offer discount).

### Subtotal, Tax, Fee, and Total Cost Calculation
The code calculates the subtotal cost, tax, fee, and total cost of the reservation based on the cost, offer discount, coupon discount, and predefined percentages.

### Pre-Paid Calculation
WThe code determines the amount to be pre-paid based on whether the user has chosen to pay the entire cost (`$request->pay_all`) or a percentage defined in the real estate object.

### Reservation Creation
Finally, a new `Reservation` object is created with all the relevant information and saved in the database. The function returns a JSON response with the created reservation data.


[🔝 Back to contents](#contents)