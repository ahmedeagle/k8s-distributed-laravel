
fix database issue due to minikube deletion 

1- access to mysql-0   
 kubectl exec -it mysql-0  -- bash 

 2- access mysql terminal as root 
   mysql -u root -p

 3-   

CREATE USER 'laravel'@'%' IDENTIFIED BY 'supersecretpassword';


GRANT ALL PRIVILEGES ON app_db.* TO 'laravel'@'%';

FLUSH PRIVILEGES;


4- from app container run php artisan migrate 

  i.e  kubectl exec -it laravel-app-79d4846f57-4dtjl -c laravel -- php artisan migrate