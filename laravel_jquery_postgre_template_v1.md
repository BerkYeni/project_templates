# Laravel Project Template with PostgreSQL, jQuery, Tailwind, Shadcn, OAuth, and Cloudflare VPS Deployment

This project template provides a robust starting point for developers to quickly build and deploy Laravel-based applications with PostgreSQL, jQuery, Tailwind CSS, Shadcn components, OAuth authentication, and deployment on a Cloudflare VPS.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Setup Instructions](#setup-instructions)
4. [Configuration Files](#configuration-files)
5. [Frontend Integration](#frontend-integration)
6. [OAuth Authentication](#oauth-authentication)
7. [Deployment on Cloudflare VPS](#deployment-on-cloudflare-vps)

## Prerequisites

- **PHP** >= 8.0
- **Composer**
- **Node.js** and **npm**
- **PostgreSQL**
- **Cloudflare Account** with VPS setup
- **Git**

## Project Structure

```
laravel-project/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   └── Middleware/
│   └── Models/
├── bootstrap/
├── config/
├── database/
│   ├── migrations/
│   └── seeders/
├── public/
│   ├── css/
│   ├── js/
│   └── index.php
├── resources/
│   ├── js/
│   ├── sass/
│   └── views/
├── routes/
│   ├── api.php
│   └── web.php
├── storage/
├── tests/
└── webpack.mix.js
```

## Setup Instructions

1. **Clone the Repository**

   ```bash
   git clone https://github.com/your-repo/laravel-project.git
   cd laravel-project
   ```

2. **Install PHP Dependencies**

   ```bash
   composer install
   ```

3. **Install Node Dependencies**

   ```bash
   npm install
   ```

4. **Configure Environment**

   Copy `.env.example` to `.env` and set your environment variables.

   ```bash
   cp .env.example .env
   ```

   Update the following in `.env`:

   ```env
   APP_NAME=Laravel
   APP_ENV=local
   APP_KEY=base64:...
   APP_DEBUG=true
   APP_URL=http://localhost

   LOG_CHANNEL=stack

   DB_CONNECTION=pgsql
   DB_HOST=127.0.0.1
   DB_PORT=5432
   DB_DATABASE=your_database
   DB_USERNAME=your_username
   DB_PASSWORD=your_password

   # OAuth Settings
   OAUTH_CLIENT_ID=your_client_id
   OAUTH_CLIENT_SECRET=your_client_secret
   OAUTH_REDIRECT_URI=http://localhost/oauth/callback
   ```

5. **Generate Application Key**

   ```bash
   php artisan key:generate
   ```

6. **Run Migrations and Seeders**

   ```bash
   php artisan migrate --seed
   ```

7. **Compile Assets**

   ```bash
   npm run dev
   ```

8. **Start the Development Server**

   ```bash
   php artisan serve
   ```

## Configuration Files

### `.env`

```env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:...
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=your_database
DB_USERNAME=your_username
DB_PASSWORD=your_password

# OAuth Settings
OAUTH_CLIENT_ID=your_client_id
OAUTH_CLIENT_SECRET=your_client_secret
OAUTH_REDIRECT_URI=http://localhost/oauth/callback
```

### `config/database.php`

```php:config/database.php
'pgsql' => [
    'driver' => 'pgsql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '5432'),
    'database' => env('DB_DATABASE', 'forge'),
    'username' => env('DB_USERNAME', 'forge'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => 'utf8',
    'prefix' => '',
    'schema' => 'public',
    'sslmode' => 'prefer',
],
```

### `webpack.mix.js`

```javascript:webpack.mix.js
const mix = require('laravel-mix');

mix.js('resources/js/app.js', 'public/js')
    .sass('resources/sass/app.scss', 'public/css')
    .postCss('resources/css/tailwind.css', 'public/css', [
        require('tailwindcss'),
    ])
    .version();
```

## Frontend Integration

### Tailwind CSS Setup

1. **Install Tailwind CSS**

   ```bash
   npm install tailwindcss postcss autoprefixer
   npx tailwindcss init
   ```

2. **Configure `tailwind.config.js`**

   ```javascript:tailwind.config.js
   module.exports = {
     content: [
       './resources/**/*.blade.php',
       './resources/**/*.js',
       './resources/**/*.vue',
     ],
     theme: {
       extend: {},
     },
     plugins: [],
   }
   ```

3. **Import Tailwind in CSS**

   Create `resources/css/tailwind.css`:

   ```css:resources/css/tailwind.css
   @tailwind base;
   @tailwind components;
   @tailwind utilities;
   ```

### Shadcn Components

Integrate Shadcn components into your Blade templates or JavaScript components as needed. Refer to [Shadcn's documentation](https://shadcn.com/docs) for detailed usage.

### jQuery Integration

1. **Install jQuery**

   ```bash
   npm install jquery
   ```

2. **Include jQuery in `resources/js/app.js`**

   ```javascript:resources/js/app.js
   window.$ = window.jQuery = require('jquery');

   // Your existing JavaScript code
   ```

## OAuth Authentication

Implement OAuth using Laravel Socialite or Passport.

### Using Laravel Socialite

1. **Install Socialite**

   ```bash
   composer require laravel/socialite
   ```

2. **Configure Socialite Providers**

   In `config/services.php`:

   ```php:config/services.php
   'github' => [
       'client_id' => env('GITHUB_CLIENT_ID'),
       'client_secret' => env('GITHUB_CLIENT_SECRET'),
       'redirect' => env('GITHUB_REDIRECT_URI'),
   ],
   ```

3. **Create Authentication Routes**

   ```php:routes/web.php
   use App\Http\Controllers\Auth\OAuthController;

   Route::get('oauth/redirect', [OAuthController::class, 'redirect']);
   Route::get('oauth/callback', [OAuthController::class, 'callback']);
   ```

4. **Create OAuthController**

   ```php:app/Http/Controllers/Auth/OAuthController.php
   namespace App\Http\Controllers\Auth;

   use App\Http\Controllers\Controller;
   use Socialite;

   class OAuthController extends Controller
   {
       public function redirect()
       {
           return Socialite::driver('github')->redirect();
       }

       public function callback()
       {
           $user = Socialite::driver('github')->user();
           // Handle user information
       }
   }
   ```

## Deployment on Cloudflare VPS

1. **Provision a VPS on Cloudflare**

   Set up your VPS instance with the required specifications.

2. **Configure the Server Environment**

   - Install **PHP**, **Nginx** or **Apache**, **PostgreSQL**, **Composer**, **Node.js**, and **npm**.
   - Secure your server with SSL using Cloudflare's SSL services.

3. **Clone the Repository on Server**

   ```bash
   git clone https://github.com/your-repo/laravel-project.git
   cd laravel-project
   ```

4. **Install Dependencies**

   ```bash
   composer install --optimize-autoloader --no-dev
   npm install
   npm run production
   ```

5. **Set Up Environment Variables**

   Ensure the `.env` file is correctly configured for the production environment.

6. **Run Migrations**

   ```bash
   php artisan migrate --force
   ```

7. **Configure the Web Server**

   Set up Nginx or Apache to serve the Laravel application. Example for Nginx:

   ```nginx:server/nginx.conf
   server {
       listen 80;
       server_name your-domain.com;
       root /path-to-your-project/public;

       add_header X-Frame-Options "SAMEORIGIN";
       add_header X-Content-Type-Options "nosniff";

       index index.php;

       location / {
           try_files $uri $uri/ /index.php?$query_string;
       }

       location ~ \.php$ {
           include snippets/fastcgi-php.conf;
           fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
           fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
           include fastcgi_params;
       }

       location ~ /\.ht {
           deny all;
       }
   }
   ```

8. **Start the Services**

   ```bash
   sudo systemctl restart nginx
   sudo systemctl restart php8.0-fpm
   ```

9. **Point Your Domain to Cloudflare**

   Update your domain's DNS settings to point to your Cloudflare DNS.

10. **Enable SSL**

    Use Cloudflare's SSL settings to secure your application.

## Conclusion

This template provides a comprehensive foundation for building modern Laravel applications with a rich frontend and secure authentication mechanisms. Customize the components as per your project requirements and leverage Cloudflare's robust infrastructure for deployment.

Feel free to contribute or raise issues on the [GitHub repository](https://github.com/your-repo/laravel-project).

# Happy Coding!