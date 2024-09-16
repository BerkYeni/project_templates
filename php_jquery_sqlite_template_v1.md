# Vanilla PHP Project Template

This project template provides a foundational structure for building a web application using Vanilla PHP for the backend, SQLite for the database, and jQuery, JavaScript, HTML, and CSS for the frontend. It incorporates OAuth for authentication and is configured for deployment on a Cloudflare VPS.

## Table of Contents

- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [File Structure and Code](#file-structure-and-code)
  - [Configuration](#configuration)
  - [Database Connection](#database-connection)
  - [OAuth Authentication](#oauth-authentication)
  - [Frontend Assets](#frontend-assets)
    - [HTML](#html)
    - [CSS](#css)
    - [JavaScript](#javascript)
- [Deployment on Cloudflare VPS](#deployment-on-cloudflare-vps)
- [Conclusion](#conclusion)

## Project Structure

```
my-php-app/
├── config/
│   └── config.php
├── db/
│   └── database.sqlite
├── public/
│   ├── assets/
│   │   ├── css/
│   │   │   └── style.css
│   │   └── js/
│   │       └── app.js
│   ├── index.php
│   └── oauth.php
├── vendor/
│   └── [OAuth Libraries]
└── README.md
```

## Setup Instructions

1. **Clone the Repository**
   ```bash
   git clone https://github.com/yourusername/my-php-app.git
   cd my-php-app
   ```

2. **Initialize the Database**
   - Ensure PHP has SQLite support enabled.
   - Run the following script to create the necessary tables:
     ```bash
     php scripts/init_db.php
     ```

3. **Configure OAuth**
   - Obtain OAuth credentials from your provider.
   - Update the `config/config.php` with your OAuth client ID and secret.

4. **Deploy to Cloudflare VPS**
   - Follow the [Deployment](#deployment-on-cloudflare-vps) section.

## File Structure and Code

### Configuration

#### `config/config.php`

```php:config/config.php
<?php
define('DB_PATH', __DIR__ . '/../db/database.sqlite');

// OAuth Configuration
define('OAUTH_CLIENT_ID', 'your-client-id');
define('OAUTH_CLIENT_SECRET', 'your-client-secret');
define('OAUTH_REDIRECT_URI', 'https://yourdomain.com/oauth.php');

// Other configurations
define('BASE_URL', 'https://yourdomain.com/');
?>
```

### Database Connection

#### `public/index.php`

```php:public/index.php
<?php
require_once __DIR__ . '/../config/config.php';

function getDbConnection() {
    try {
        $pdo = new PDO('sqlite:' . DB_PATH);
        // Set error mode to exceptions
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        return $pdo;
    } catch (PDOException $e) {
        die("DB Connection failed: " . $e->getMessage());
    }
}

// Example usage
$db = getDbConnection();
// Your code here
?>
```

### OAuth Authentication

For OAuth implementation, we'll use the [PHP League's OAuth2 Client](https://oauth2-client.thephpleague.com/). Install it via Composer.

#### Installation

```bash
composer require league/oauth2-client
```

#### `public/oauth.php`

```php:public/oauth.php
<?php
require_once __DIR__ . '/../config/config.php';
require_once __DIR__ . '/../vendor/autoload.php';

use League\OAuth2\Client\Provider\GenericProvider;

function getOAuthProvider() {
    return new GenericProvider([
        'clientId'                => OAUTH_CLIENT_ID,
        'clientSecret'            => OAUTH_CLIENT_SECRET,
        'redirectUri'             => OAUTH_REDIRECT_URI,
        'urlAuthorize'            => 'https://provider.com/oauth2/authorize',
        'urlAccessToken'          => 'https://provider.com/oauth2/token',
        'urlResourceOwnerDetails' => 'https://provider.com/oauth2/resource'
    ]);
}

session_start();

$provider = getOAuthProvider();

if (!isset($_GET['code'])) {
    // If we don't have an authorization code then get one
    $authorizationUrl = $provider->getAuthorizationUrl();
    $_SESSION['oauth2state'] = $provider->getState();
    header('Location: ' . $authorizationUrl);
    exit;
} elseif (empty($_GET['state']) || ($_GET['state'] !== $_SESSION['oauth2state'])) {
    unset($_SESSION['oauth2state']);
    exit('Invalid state');
} else {
    try {
        // Try to get an access token using the authorization code grant
        $accessToken = $provider->getAccessToken('authorization_code', [
            'code' => $_GET['code']
        ]);

        // Use the access token to get user details
        $resourceOwner = $provider->getResourceOwner($accessToken);
        $user = $resourceOwner->toArray();

        // Store user information in session or database
        $_SESSION['user'] = $user;

        // Redirect to the homepage
        header('Location: index.php');
        exit;

    } catch (Exception $e) {
        exit('Failed to get access token: ' . $e->getMessage());
    }
}
?>
```

### Frontend Assets

#### HTML

##### `public/index.php`

```php:public/index.php
<?php
require_once __DIR__ . '/../config/config.php';
session_start();
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My PHP App</title>
    <link rel="stylesheet" href="assets/css/style.css">
</head>
<body>
    <header>
        <h1>Welcome to My PHP App</h1>
        <?php if(isset($_SESSION['user'])): ?>
            <p>Hello, <?= htmlspecialchars($_SESSION['user']['name']) ?>!</p>
            <a href="logout.php">Logout</a>
        <?php else: ?>
            <a href="oauth.php">Login with OAuth</a>
        <?php endif; ?>
    </header>

    <main>
        <div id="content">
            <!-- Your content here -->
        </div>
    </main>

    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="assets/js/app.js"></script>
</body>
</html>
```

#### CSS

##### `public/assets/css/style.css`

```css:public/assets/css/style.css
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 0;
}

header {
    background-color: #4CAF50;
    color: white;
    padding: 1em;
    text-align: center;
}

main {
    padding: 2em;
}

a {
    color: #fff;
    text-decoration: none;
    margin-left: 1em;
}
```

#### JavaScript

##### `public/assets/js/app.js`

```javascript:public/assets/js/app.js
$(document).ready(function() {
    console.log("App initialized");

    // Example AJAX request
    $('#content').load('api/data.php', function(response, status, xhr) {
        if (status == "error") {
            $('#content').html("An error occurred: " + xhr.status + " " + xhr.statusText);
        }
    });
});
```

### API Endpoint Example

#### `public/api/data.php`

```php:public/api/data.php
<?php
require_once __DIR__ . '/../../config/config.php';

header('Content-Type: application/json');

function getDbConnection() {
    try {
        $pdo = new PDO('sqlite:' . DB_PATH);
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        return $pdo;
    } catch (PDOException $e) {
        echo json_encode(['error' => 'DB Connection failed']);
        exit;
    }
}

$db = getDbConnection();

try {
    $stmt = $db->query('SELECT * FROM your_table');
    $data = $stmt->fetchAll(PDO::FETCH_ASSOC);
    echo json_encode($data);
} catch (Exception $e) {
    echo json_encode(['error' => 'Failed to fetch data']);
}
?>
```

## Deployment on Cloudflare VPS

1. **Provision a VPS with Cloudflare**
   - Choose your preferred operating system (e.g., Ubuntu).
   - Ensure PHP and SQLite are installed.

2. **Transfer Project Files**
   - Use `git` to clone the repository or `scp` to transfer files.
   ```bash
   git clone https://github.com/yourusername/my-php-app.git
   cd my-php-app
   ```

3. **Install Dependencies**
   - Install Composer if not already installed.
   ```bash
   curl -sS https://getcomposer.org/installer | php
   sudo mv composer.phar /usr/local/bin/composer
   ```
   - Install PHP dependencies.
   ```bash
   composer install
   ```

4. **Set Up Web Server**
   - Install and configure a web server like Nginx or Apache.
   - Configure the document root to point to the `public/` directory.

   **Example Nginx Configuration:**

   ```nginx
   server {
       listen 80;
       server_name yourdomain.com;

       root /path/to/my-php-app/public;
       index index.php index.html index.htm;

       location / {
           try_files $uri $uri/ /index.php?$query_string;
       }

       location ~ \.php$ {
           include snippets/fastcgi-php.conf;
           fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
       }

       location ~ /\.ht {
           deny all;
       }
   }
   ```

5. **Secure the Application**
   - Obtain an SSL certificate. You can use Cloudflare's SSL or Let's Encrypt.
   - Configure HTTPS in your web server.

6. **Configure Cloudflare DNS**
   - Add your domain to Cloudflare.
   - Update your DNS records to point to your VPS IP address.

7. **Finalize Deployment**
   - Ensure file permissions are set correctly.
   - Restart the web server.
   ```bash
   sudo systemctl restart nginx
   ```
   - Test your application by visiting `https://yourdomain.com`.

## Conclusion

This template sets up a simple yet extensible foundation for building PHP-based web applications with essential features like OAuth authentication and a SQLite database. By following the structure and utilizing the provided code snippets, developers can accelerate the development process and focus on building unique functionalities tailored to their project's needs. Deploying on a Cloudflare VPS ensures scalability and security, leveraging Cloudflare's robust infrastructure.

Feel free to customize and expand upon this template to suit your specific requirements.