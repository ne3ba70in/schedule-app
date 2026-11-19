# Schedule App (Backend)


## Стек
- Laravel 11/13 (PHP 8.3)
- MySQL

## Демо
Работающее приложение (Vue + Laravel): http://workt0prog.alwaysdata.net

## Запуск локально
1. `composer install`
2. `cp .env.example .env`
3. Настройте базу данных в `.env`
4. `php artisan key:generate`
5. `php artisan migrate --seed`
6. `php artisan serve`
