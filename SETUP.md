# Django Setup Commands (Reference)

## 1. Check Python version
python3 --version

## 2. Create a virtual environment
python3 -m venv venv

## 3. Activate the virtual environment
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

## 4. Upgrade pip (optional)
pip install --upgrade pip

## 5. Install Django
pip install django

## 6. Verify installation
python -m django --version

## 7. Create a Django project
django-admin startproject myproject .

## 8. Run database migrations
python manage.py migrate

## 9. Run the development server
python manage.py runserver
# Visit http://127.0.0.1:8000/

## 10. Create a superuser (optional, for admin panel)
python manage.py createsuperuser

## 11. Create your first app (optional)
python manage.py startapp myapp

## 12. Save dependencies (optional)
pip freeze > requirements.txt

---

## Common day-to-day commands
source venv/bin/activate        # activate venv before working
python manage.py runserver      # start dev server
python manage.py makemigrations # create migration files after model changes
python manage.py migrate        # apply migrations
deactivate                      # exit virtual environment
