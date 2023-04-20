# Deploy React - Django - DRF - Gunicorn - NGINX application

## Local Development

1.  Create a DRF app with a model, the admin panel, and a JSON endpoint
2.  Generate a Django superuser, add some content in the admin portal
3.  Create a React app fetching data from backend endpoint. The fetch url is
    http://localhost:8000
4.  To allow React to access data delivered by the backend, set up CORS in
    Django with `pip install django-cors-headers` and add corsheaders to
    INSTALLED_APPS and MIDDLEWARE

        Allow requests from the local React app:

        ```
        CORS_ALLOWED_ORIGINS = [
            "http://localhost:3000",
        ]

        MIDDLEWARE = [
            "corsheaders.middleware.CorsMiddleware",
            '...',
        ]
        ```

5.  Test with locally running React and Django. React app should load data
    coming from Django.

## Prepare for deployment

### Backend

1. Install gunicorn for serving the Django app more effectively (with either pip
   or apt)
2. Test serving the app with `gunicorn app.wsgi`
3. Generate requirements `pip freeze > requirements.txt`
4. Run `python manage.py check --deploy`. A lot of points will be missing.

## Deploy to VPS

### initial setup

1. set up VPS with SSH
2. Do a [server initial
   setup](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)
3. Allow WWW in UFW
4. install nginx
5. verify nginx is working by opening the VPS IP address in the browser.

### deploy frontend [(docs)](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-react-application-with-nginx-on-ubuntu-20-04)

1. Do a production build of the React app with `yarn build`
2. on the VPS, open `/etc/nginx/sites-available`
3. replace the `default` file content with the simplest nginx config:

   ```
   server {
           listen 80;
           listen [::]:80;

           root /var/www//html;
           index index.html index.htm index.nginx-debian.html;

           server_name your_domain www.your_domain;

           location / {
                   try_files $uri $uri/ =404;
           }
   }
   ```

4. Copy the built react app to the VPS html location `scp -r ./build/* <vps_address>:/var/www/html`
5. Check the VPS again, the react app should load, but it won't be able to fetch
   the data from the backend

### deploy backend

1. In Django, run `python manage.py collectstatic`, we'll need the static files
   for the admin portal styling
2. Remove the db.sqlite3 or don't copy it to the VPS
3. Copy the Django app to the VPS using `scp` to the home folder of the user that you created in the server initial setup.
4. Make sure Python3 and pip is installed on the VPS, and pip targets Python3
   and not 2 (it can happen!)
5. Install gunicorn
6. (temporarily) enable port 8000 in ufw `sudo ufw allow 8000`
7. Test running the backend with `gunicorn app.wsgi --bind 0.0.0.0`
8. The app will fail because of HTTP_ALLOWED_HOSTS
9. Not it's time for splitting the settings file to different envs:

   - local settings

   ```
   from app.base_settings import *

   SECRET_KEY = 'django-insecure-t#d=(qmc9c6h!1kw$(80-$%k9wz71603jo*&l^o0k#v2(zuht7'
   DEBUG = True
   ALLOWED_HOSTS = []
   CORS_ALLOWED_ORIGINS = [
       "http://localhost:3000",
   ]
   ```

   - prod settings:

   ```
   from app.base_settings import *

   SECRET_KEY = '<something long and random>'
   DEBUG = False
   ALLOWED_HOSTS = ["<vps_address>"]
   CORS_ALLOWED_ORIGINS = [
       "http://<vps_address>",
   ]
   ```

   - base setttings (everything else that was left in `settings.py`)

10. Push updated app to the VPS
11. Now running the dev server or gunicorn will require specifying the settings
    file per enviroment: run `export DJANGO_SETTINGS_MODULE=app.local` or `export
DJANGO_SETTINGS_MODULE=app.prod` in the django app folder. This will only work as long as the terminal session is opened, so later we will need something more stable.
12. Run the migrations on the VPS `python3 manage.py migrate`, this will also
    recreate the db file
13. Run again with gunicorn
14. Backend app should now run, but with broken static files and no posts
15. create superuser `python3 manage.py createsuperuser`
16. go to the admin portal [http://<vps_address>:8000/admin](http://<vps_address>:8000/admin)
17. Add the first post, verify on the index page if it was added.
18. Static files won't work, as nginx will be serving them, not backend
19. copy backend static files to the same place as frontend static files `sudo
cp -r /<local_django_app_path>/ /var/www/html/static/`
20. [Set up nginx reverse-proxy for
    gunicorn](https://docs.gunicorn.org/en/stable/deploy.html)

    ```
    server {
            listen 80;
            listen [::]:80;

            root /var/www/html;
            index index.html;

            server_name your_domain www.your_domain;

            location / {
                    try_files $uri $uri/ @proxy_to_app;
            }

            location @proxy_to_app {
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header Host $http_host;
                    proxy_redirect off;
                    proxy_pass http://<vps_address>:8000;
            }

    }
    ```

21. Check config with nginx-t
22. restart nginx `sudo systemctl restart nginx`
23. backend should now reply at http://<vps_address>:8000/api and
    http://<vps_address>:8000/admin with nicely working static files in the admin
24. Remove access to port 8000 in the VPS, we won't need it anymore `sudo ufw
deny 8000`
25. in React, use different base urls based on the environment. React
    automatically discerns between local dev and production:

    ```
    const development = "http://localhost:8000/api";
    const production = "http://<vps_address>/api";

    const baseUrl =
    process.env.NODE_ENV === "development" ? development : production;
    ```

26. If everything works, time to [daemonize
    backend](https://docs.gunicorn.org/en/stable/deploy.html#supervisor) to let
    it work without running gunicorn via ssh: `pip3 install supervisor`
27. how to run supervisor: http://supervisord.org/running.html#running

    ```
    [program:gunicorn]
    command=/home/stefan/.local/bin/gunicorn app.wsgi --bind 0.0.0.0
    directory=/home/stefan/app
    user=stefan
    autostart=true
    autorestart=true
    redirect_stderr=true
    environment=DJANGO_SETTINGS_MODULE="app.prod"
    ```

    Run `supervisor` `supervisorctl status` to confirm supervisor is working.

    DONE!

## Next steps
- Postgres instead of sqlite
- JWT Auth
- Domain and SSL