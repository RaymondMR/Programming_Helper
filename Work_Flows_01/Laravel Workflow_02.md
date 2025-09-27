# Laravel Workflow 02

## 0) Requisitos locales
- PHP 8.2+ (con ext: OpenSSL, PDO, Mbstring, Tokenizer, XML, Ctype, JSON, BCMath, Fileinfo).
- Composer.
- Node.js 18+ y npm (para Vite).
- Servidor y DB (MySQL/MariaDB o PostgreSQL) — o usa Docker/Sail (opcional).

## 1) Crear el proyecto
**Opción A (carpeta ya creada):**
```bash
cd ruta\de\mi\carpeta
composer create-project laravel/laravel .
php artisan serve
```
**Opción B (nueva carpeta con nombre del proyecto):**
```bash
composer create-project laravel/laravel blog-app
cd blog-app
php artisan serve
```

## 2) Configurar entorno y base de datos
1. Duplica `.env.example` → `.env`:
   ```bash
   cp .env.example .env
   php artisan key:generate
   ```
2. Crea usuario/esquema en tu SGBD.
3. Edita `.env` (MySQL ejemplo):
   ```
   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=mi_db
   DB_USERNAME=mi_usuario
   DB_PASSWORD=mi_password
   ```

## 3) Migraciones, modelos y seeders
```bash
php artisan migrate
php artisan make:model Post -mfs
php artisan migrate:fresh --seed
```

## 4) Rutas y controlador
```php
// routes/web.php
use App\Http\Controllers\PostController;
Route::resource('posts', PostController::class);
```
```bash
php artisan make:controller PostController --resource
```

## 5) Vistas Blade + layout
Crear vistas en `resources/views/posts/` y un layout base en `resources/views/layouts/app.blade.php`.

## 6) Assets con Vite
```bash
npm install
npm run dev
```

## 7) Autenticación rápida (Breeze)
```bash
composer require laravel/breeze --dev
php artisan breeze:install blade
npm install && npm run dev
php artisan migrate
```

## 8) API (opcional con Sanctum)
```bash
composer require laravel/sanctum
php artisan migrate
```

## 9) Políticas y autorización
```bash
php artisan make:policy PostPolicy --model=Post
```

## 10) Form Requests
```bash
php artisan make:request StorePostRequest
```

## 11) Subida de archivos
```bash
php artisan storage:link
```

## 12) Optimización (producción)
```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan optimize
```

## 13) Pruebas
```bash
php artisan test
```

## 14) Jobs y Queues
```bash
php artisan queue:table
php artisan migrate
php artisan queue:work
```

## 15) Git
```bash
git init
echo "/vendor" >> .gitignore
echo "/node_modules" >> .gitignore
git add .
git commit -m "Bootstrap Laravel app"
```

## 16) Despliegue
Apuntar DocumentRoot a `public/`, configurar `.env` productivo y permisos en `storage` y `bootstrap/cache`.

## 17) Alternativa con Docker Sail
```bash
composer require laravel/sail --dev
php artisan sail:install
./vendor/bin/sail up -d
```

---
**Checklist rápido:**
1. Crear proyecto → `php artisan serve`
2. Configurar `.env` y DB
3. Migraciones y seeds
4. Controlador y rutas
5. Vistas Blade
6. `npm install && npm run dev`
7. Breeze para auth
