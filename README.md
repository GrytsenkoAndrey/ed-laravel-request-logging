# Log requests and responses in Laravel

Today we’ll learn how to log each request and response in Laravel using middleware and then call it on any route that request and response you need to log for further analysis.

you can use this approach to implement this on a new application as well as in any existing application with just a few steps.

## 1. Create a Middleware to handle logs only in order of code separation
## 2. move to the app/Http/Middleware directory and create a file called LoggerMiddleware.php of course you can name it any, but for readability, I'll use the same name as of its responsibility and paste the following contents as below and save it.

```
<?php
// app/Http/Middleware/LoggerMiddleware.php

namespace App\Http\Middleware;

use Closure;
use Carbon\Carbon;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class LoggerMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param Request $request
     * @param Closure $next
     *
     * @return mixed
     */
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        $contents = json_decode($response->getContent(), true, 512);
        
        $headers  = $request->header();

    
        $dt = new Carbon();
        $data = [
            'path'         => $request->getPathInfo(),
            'method'       => $request->getMethod(),
            'ip'           => $request->ip(),
            'http_version' => $_SERVER['SERVER_PROTOCOL'],
            'timestamp'    => $dt->toDateTimeString(),
            'headers'      => [
                // get all the required headers to log
                'user-agent' => $headers['user-agent'],
                'referer'    => $headers['referer'],
                'origin'     => $headers['origin'],
            ], 
        ];

        // if request if authenticated
        if ($request->user()) {
            $data['user_id'] = $request->user()->id;
        }

        // if you want to log all the request body
        if (count($request->all()) > 0) {
             // keys to skip like password or any sensitive information
            $hiddenKeys = ['password'];

            $data['request'] = $request->except($hiddenKeys);
        }

        // to log the message from the response
        if (!empty($contents['message'])) {
            $data['response']['message'] = $contents['message'];
        }

        // to log the errors from the response in case validation fails or other errors get thrown
        if (!empty($contents['errors'])) {
            $data['response']['errors'] = $contents['errors'];
        }

        // to log the data from the response, change the RESULT to your API key that holds data
        if (!empty($contents['result'])) {
            $data['response']['result'] = $contents['result'];
        }

        // a unique message to log, I prefer to save the path of request for easy debug
        $message     = str_replace('/', '_', trim($request->getPathInfo(), '/'));

        // log the gathered information
        Log::info($message, $data);

        // return the response
        return $response;
    }
}
```

## 3. open file app/Http/Kernal.php and add this newly created middleware $routeMiddleware Array

```
protected $routeMiddleware = [
...
'logger' => \App\Http\Middleware\LoggerMiddleware::class,
]
```

## 4. now time to implement this middleware so it can do what it is to

open your route file(s) in the routes directory and apply this middleware to let it perform its functionality

```
Route::middleware(['logger'])->group(function () {
    Route::get('/', function() {
        return 'welcome to logger middleware implementation';
    });

    Route::post('/', function(Request $request) {
        // to do something with your request and return the response
        return 'yes it is working as required';
    });
});
```

and that’s it.

open your log file under storage/logs/* and you'll see the request and a response in a very nicely formatted way.
