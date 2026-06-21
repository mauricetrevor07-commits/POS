# Deployment Guide

This guide will help you deploy the Django Point of Sale system to production.

## Prerequisites

- Python 3.x
- PostgreSQL database (recommended for production)
- A hosting platform (Heroku, Render, Railway, DigitalOcean, etc.)
- Git

## Environment Variables

Create a `.env` file in the project root (copy from `.env.example`):

```bash
cp .env.example .env
```

Required environment variables:

- `DEBUG=False` - Must be False in production
- `SECRET_KEY` - Generate a secure secret key (use: `python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"`)
- `ALLOWED_HOSTS` - Comma-separated list of allowed domains (e.g., `yourdomain.com,www.yourdomain.com`)
- `DATABASE_URL` - PostgreSQL connection string (format: `postgres://user:password@host:port/database_name`)

## Deployment Platforms

### Heroku

1. **Install Heroku CLI** and login:
   ```bash
   heroku login
   ```

2. **Create a new Heroku app**:
   ```bash
   heroku create your-app-name
   ```

3. **Add PostgreSQL addon**:
   ```bash
   heroku addons:create heroku-postgresql:mini
   ```

4. **Set environment variables**:
   ```bash
   heroku config:set DEBUG=False
   heroku config:set SECRET_KEY=your-generated-secret-key
   heroku config:set ALLOWED_HOSTS=your-app-name.herokuapp.com
   ```

5. **Deploy**:
   ```bash
   git push heroku main
   ```

6. **Run migrations**:
   ```bash
   heroku run python manage.py migrate
   ```

7. **Create superuser**:
   ```bash
   heroku run python manage.py createsuperuser
   ```

8. **Collect static files**:
   ```bash
   heroku run python manage.py collectstatic --noinput
   ```

### Render

1. **Create a new Web Service** on Render dashboard
2. **Connect your GitHub repository**
3. **Configure build settings**:
   - Build Command: `cd django_pos && pip install -r ../requirements.txt && python manage.py collectstatic --noinput`
   - Start Command: `cd django_pos && gunicorn django_pos.wsgi:application`
4. **Add PostgreSQL database** (Render will provide DATABASE_URL)
5. **Set environment variables** in Render dashboard:
   - `DEBUG=False`
   - `SECRET_KEY=your-generated-secret-key`
   - `ALLOWED_HOSTS=your-app-name.onrender.com`
6. **Deploy**

### Railway

1. **Install Railway CLI** and login:
   ```bash
   railway login
   ```

2. **Initialize project**:
   ```bash
   railway init
   ```

3. **Add PostgreSQL database**:
   ```bash
   railway add postgresql
   ```

4. **Set environment variables**:
   ```bash
   railway variables set DEBUG=False
   railway variables set SECRET_KEY=your-generated-secret-key
   railway variables set ALLOWED_HOSTS=your-app-name.railway.app
   ```

5. **Deploy**:
   ```bash
   railway up
   ```

### DigitalOcean App Platform

1. **Create a new App** on DigitalOcean dashboard
2. **Connect your GitHub repository**
3. **Configure build settings**:
   - Build Command: `cd django_pos && pip install -r ../requirements.txt && python manage.py collectstatic --noinput`
   - Run Command: `cd django_pos && gunicorn django_pos.wsgi:application`
4. **Add PostgreSQL database** (DigitalOcean will provide DATABASE_URL)
5. **Set environment variables** in App Settings:
   - `DEBUG=False`
   - `SECRET_KEY=your-generated-secret-key`
   - `ALLOWED_HOSTS=your-app-name.ondigitalocean.app`
6. **Deploy**

## Manual Deployment (VPS/Dedicated Server)

If deploying to a VPS (e.g., AWS EC2, DigitalOcean Droplet):

1. **Install dependencies**:
   ```bash
   sudo apt update
   sudo apt install python3-pip python3-venv postgresql postgresql-contrib nginx
   ```

2. **Clone repository**:
   ```bash
   git clone https://github.com/your-username/django_point_of_sale.git
   cd django_point_of_sale
   ```

3. **Create virtual environment**:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

4. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

5. **Configure environment variables**:
   ```bash
   cp .env.example .env
   # Edit .env with your values
   nano .env
   ```

6. **Setup PostgreSQL database**:
   ```bash
   sudo -u postgres psql
   CREATE DATABASE django_pos;
   CREATE USER django_pos_user WITH PASSWORD 'your-password';
   GRANT ALL PRIVILEGES ON DATABASE django_pos TO django_pos_user;
   \q
   ```

7. **Run migrations**:
   ```bash
   cd django_pos
   python manage.py migrate
   ```

8. **Create superuser**:
   ```bash
   python manage.py createsuperuser
   ```

9. **Collect static files**:
   ```bash
   python manage.py collectstatic --noinput
   ```

10. **Configure Gunicorn as a systemd service**:
    Create `/etc/systemd/system/gunicorn.service`:
    ```ini
    [Unit]
    Description=gunicorn daemon for Django POS
    After=network.target

    [Service]
    User=www-data
    Group=www-data
    WorkingDirectory=/path/to/django_point_of_sale/django_pos
    ExecStart=/path/to/django_point_of_sale/venv/bin/gunicorn \
              --workers 3 \
              --bind unix:/path/to/django_point_of_sale/django_pos.sock \
              django_pos.wsgi:application

    [Install]
    WantedBy=multi-user.target
    ```

11. **Start Gunicorn**:
    ```bash
    sudo systemctl start gunicorn
    sudo systemctl enable gunicorn
    ```

12. **Configure Nginx**:
    Create `/etc/nginx/sites-available/django_pos`:
    ```nginx
    server {
        listen 80;
        server_name your-domain.com;

        location /static/ {
            alias /path/to/django_point_of_sale/django_pos/staticfiles/;
        }

        location / {
            proxy_pass http://unix:/path/to/django_point_of_sale/django_pos.sock;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
    ```

13. **Enable site and restart Nginx**:
    ```bash
    sudo ln -s /etc/nginx/sites-available/django_pos /etc/nginx/sites-enabled
    sudo nginx -t
    sudo systemctl restart nginx
    ```

14. **Setup SSL with Let's Encrypt** (recommended):
    ```bash
    sudo apt install certbot python3-certbot-nginx
    sudo certbot --nginx -d your-domain.com
    ```

## Security Notes

- **Never commit `.env` file** to version control
- **Always use strong, unique SECRET_KEY** in production
- **Keep DEBUG=False** in production
- **Use HTTPS** in production (SSL/TLS certificate)
- **Keep dependencies updated** regularly
- **Use a firewall** to restrict access
- **Regular backups** of your database

## Troubleshooting

### Static files not loading
- Run `python manage.py collectstatic --noinput`
- Check STATIC_ROOT and STATIC_URL settings
- Verify whitenoise middleware is properly configured

### Database connection errors
- Verify DATABASE_URL is correct
- Check PostgreSQL is running
- Ensure database user has proper permissions

### 500 Internal Server Error
- Check application logs
- Verify all environment variables are set
- Ensure DEBUG=False in production (check error logs instead)

### Permission denied errors
- Check file permissions
- Ensure proper ownership of files and directories

## Post-Deployment Checklist

- [ ] DEBUG is set to False
- [ ] SECRET_KEY is set and secure
- [ ] ALLOWED_HOSTS includes your domain
- [ ] DATABASE_URL is configured
- [ ] Migrations have been run
- [ ] Superuser has been created
- [ ] Static files have been collected
- [ ] HTTPS/SSL is configured
- [ ] Database backups are set up
- [ ] Monitoring/logging is configured
- [ ] Firewall rules are configured

## Support

For issues or questions, please refer to the main README.md or open an issue on GitHub.
