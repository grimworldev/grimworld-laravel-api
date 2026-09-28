# GRIMWORLD Starter Kit: Backend (Laravel API)

The REST API for the **GRIMWORLD Starter Kit**, a Laravel + Next.js starter. It provides token authentication (Laravel Sanctum) with UUID-based users.

> Developed by **FJ Buenaflor**.

The frontend lives in the `frontend/` folder. See its README for setup.

## Features

- REST API under `/api`
- Token authentication with Laravel Sanctum (register, login, logout)
- Login with **username** and password
- UUID primary keys for users
- Protected routes with `auth:sanctum`
- CORS configured for the Next.js frontend

## Requirements

- PHP 8.2 or higher
- Composer
- MySQL 8 or higher (or another database Laravel supports)

## Quick start

```bash
cd backend
composer install
cp .env.example .env
php artisan key:generate
```

Create an empty database (for example `grimworld`), set the `DB_*` values in `.env`, then run:

```bash
php artisan migrate
php artisan serve
```

The API is now running at `http://localhost:8000`. Keep this terminal open while you use the frontend.

## Environment variables

Copy `.env.example` to `.env`. These are the values that matter for this kit:

| Variable | Description | Example |
| --- | --- | --- |
| `APP_NAME` | Application name | `GRIMWORLD` |
| `APP_URL` | URL of the Laravel app | `http://localhost:8000` |
| `FRONTEND_URL` | Origin allowed by CORS (the Next.js app) | `http://localhost:3000` |
| `DB_CONNECTION` | Database driver | `mysql` |
| `DB_HOST` | Database host | `127.0.0.1` |
| `DB_PORT` | Database port | `3306` |
| `DB_DATABASE` | Database name | `grimworld` |
| `DB_USERNAME` | Database user | `root` |
| `DB_PASSWORD` | Database password | *(empty)* |

Never commit `.env`. Only `.env.example` is committed.

## API endpoints

All endpoints are prefixed with `/api`. Send `Accept: application/json` on every request, otherwise Laravel may redirect instead of returning JSON errors.

| Method | Endpoint | Auth | Description |
| --- | --- | --- | --- |
| GET | `/api/hello` | No | Health check |
| POST | `/api/register` | No | Create an account and receive a token |
| POST | `/api/login` | No | Log in with username and password, receive a token |
| POST | `/api/logout` | Yes | Revoke the current token |
| GET | `/api/user` | Yes | Get the authenticated user |
| * | `/api/users` | Yes | Users resource (`UserController`) |

Protected requests must send the header `Authorization: Bearer <token>`.

### Register

`POST /api/register`

```json
{
  "username": "johndoe",
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "password": "password123",
  "password_confirmation": "password123"
}
```

Response `201`:

```json
{
  "user": { "id": "uuid...", "username": "johndoe", "...": "..." },
  "token": "1|abc..."
}
```

### Login

`POST /api/login`

```json
{
  "username": "johndoe",
  "password": "password123"
}
```

Response `200` has the same shape as register. Wrong credentials return `422` with an error under `errors.username`.

### Validation errors

Laravel returns `422` in this shape, and the frontend maps `errors` to the form fields:

```json
{
  "message": "The username has already been taken.",
  "errors": { "username": ["The username has already been taken."] }
}
```

## Testing the API

With curl:

```bash
# Health check
curl http://localhost:8000/api/hello -H "Accept: application/json"

# Register
curl -X POST http://localhost:8000/api/register \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -d '{"username":"johndoe","first_name":"John","last_name":"Doe","email":"john@example.com","password":"password123","password_confirmation":"password123"}'

# Login
curl -X POST http://localhost:8000/api/login \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -d '{"username":"johndoe","password":"password123"}'

# Current user (replace TOKEN)
curl http://localhost:8000/api/user \
  -H "Accept: application/json" -H "Authorization: Bearer TOKEN"
```

You can also use Postman or Insomnia. Also confirm that `GET /api/users` without a token returns `401`.

## Project structure (important files)

```
backend/
├── app/
│   ├── Http/Controllers/
│   │   ├── AuthController.php     # register, login, logout
│   │   └── UserController.php
│   └── Models/User.php            # HasApiTokens, HasUuids, hashed password
├── config/cors.php                # allows FRONTEND_URL
├── database/migrations/
│   ├── ..._create_users_table.php
│   └── ..._create_personal_access_tokens_table.php   # uses uuidMorphs
├── routes/api.php
└── .env.example
```

## Important notes for this kit

- **UUID users and Sanctum:** the `personal_access_tokens` migration must use `$table->uuidMorphs('tokenable')` instead of `$table->morphs('tokenable')`. Otherwise token creation fails with `Data truncated for column 'tokenable_id'`.
- **Password hashing:** the `User` model has `'password' => 'hashed'` in its casts, so do **not** call `Hash::make()` before saving. Doing so hashes the password twice and login will always fail.
- **`HasApiTokens`:** the `User` model must use the `Laravel\Sanctum\HasApiTokens` trait, otherwise `createToken()` throws an error.
- **Sessions table:** if you use the `database` session driver, use `foreignUuid('user_id')` in the `sessions` migration, since user IDs are UUIDs.

## CORS

CORS is configured in `config/cors.php`. If you don't have this file, publish it:

```bash
php artisan config:publish cors
```

```php
'paths' => ['api/*'],
'allowed_methods' => ['*'],
'allowed_origins' => [env('FRONTEND_URL', 'http://localhost:3000')],
'allowed_headers' => ['*'],
```

After changing `.env` or config files, run `php artisan config:clear`.

## Troubleshooting

| Problem | Likely cause and fix |
| --- | --- |
| `Data truncated for column 'tokenable_id'` | Change `morphs` to `uuidMorphs` in the personal access tokens migration, then run `php artisan migrate:fresh` (this wipes dev data). |
| Login always says credentials are incorrect | The password was hashed twice. Remove `Hash::make()` from registration and register the account again. |
| `Call to undefined method createToken()` | Add the `HasApiTokens` trait to the `User` model. |
| Browser shows a CORS error | Check `FRONTEND_URL` in `.env`, then run `php artisan config:clear`. |
| `422` on register | Read the response body. Usually the username or email is taken, the password is under 8 characters, or `password_confirmation` is missing. |
| `401` on protected routes | The `Authorization: Bearer <token>` header is missing or the token was revoked. |
| Redirect to `/login` instead of JSON | The `Accept: application/json` header is missing. |

## Useful commands

```bash
php artisan migrate:fresh      # reset the database (deletes all data)
php artisan route:list         # list all routes
php artisan config:clear       # clear cached config
php artisan tinker             # interactive shell
```

## Credits

Developed by **FJ Buenaflor**.
# grimworld-laravel-api
