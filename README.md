# Create virtual environment (recommended)
python -m venv venv
source venv/bin/activate   # on macOS/Linux
venv\Scripts\activate      # on Windows

# Install Django + DRF + GraphQL
pip install django djangorestframework djangorestframework-simplejwt graphene-django

# Create project
django-admin startproject taskmanager

cd taskmanager

# Create app for tasks
python manage.py startapp tasks
🛠 Step 2. Update settings.py
Inside taskmanager/settings.py, add required apps:

python
Copy code
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party
    'rest_framework',
    'rest_framework.authtoken',
    'graphene_django',

    # Local
    'tasks',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}

GRAPHENE = {
    "SCHEMA": "taskmanager.schema.schema"
}
Run migration to create tables:

bash
Copy code
python manage.py migrate
🛠 Step 3. Create Task Model (tasks/models.py)
python
Copy code
from django.db import models
from django.contrib.auth.models import User

class Task(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('in_progress', 'In Progress'),
        ('done', 'Done'),
    ]

    title = models.CharField(max_length=255)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    created_at = models.DateTimeField(auto_now_add=True)
    assigned_to = models.ForeignKey(User, on_delete=models.CASCADE, related_name="tasks")

    def __str__(self):
        return self.title
Run migration:

bash
Copy code
python manage.py makemigrations
python manage.py migrate
🛠 Step 4. Create Serializer (tasks/serializers.py)
python
Copy code
from rest_framework import serializers
from .models import Task

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ['id', 'title', 'status', 'created_at', 'assigned_to']
        read_only_fields = ['id', 'created_at', 'assigned_to']
🛠 Step 5. REST API Views (tasks/views.py)
python
Copy code
from rest_framework import generics, permissions
from .models import Task
from .serializers import TaskSerializer

class TaskListCreateView(generics.ListCreateAPIView):
    serializer_class = TaskSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Task.objects.filter(assigned_to=self.request.user)

    def perform_create(self, serializer):
        serializer.save(assigned_to=self.request.user)


class TaskDetailView(generics.RetrieveUpdateDestroyAPIView):
    serializer_class = TaskSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Task.objects.filter(assigned_to=self.request.user)
🛠 Step 6. REST API URLs (tasks/urls.py)
python
Copy code
from django.urls import path
from rest_framework.authtoken.views import obtain_auth_token
from .views import TaskListCreateView, TaskDetailView

urlpatterns = [
    path('auth/', obtain_auth_token),  # get token
    path('tasks/', TaskListCreateView.as_view(), name='task-list-create'),
    path('tasks/<int:pk>/', TaskDetailView.as_view(), name='task-detail'),
]
Include it in main taskmanager/urls.py:

python
Copy code
from django.contrib import admin
from django.urls import path, include
from graphene_django.views import GraphQLView
from django.views.decorators.csrf import csrf_exempt

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('tasks.urls')),
    path("graphql/", csrf_exempt(GraphQLView.as_view(graphiql=True))),
]
🛠 Step 7. GraphQL Schema (taskmanager/schema.py)
python
Copy code
import graphene
from graphene_django.types import DjangoObjectType
from tasks.models import Task

class TaskType(DjangoObjectType):
    class Meta:
        model = Task
        fields = ("id", "title", "status", "created_at", "assigned_to")

class Query(graphene.ObjectType):
    all_tasks = graphene.List(TaskType)

    def resolve_all_tasks(self, info):
        user = info.context.user
        if user.is_anonymous:
            raise Exception("Authentication required!")
        return Task.objects.filter(assigned_to=user)

schema = graphene.Schema(query=Query)
🛠 Step 8. Create Superuser & Test
bash
Copy code
python manage.py createsuperuser
Then run the server:

bash
Copy code
python manage.py runserver
✅ Endpoints
Get Token:
POST /api/auth/ with { "username": "yourname", "password": "yourpass" }

List/Create Tasks:
GET/POST /api/tasks/

Update/Delete Task:
PUT/DELETE /api/tasks/<id>/

GraphQL:
POST /graphql/
Query example:

graphql
Copy code
{
  allTasks {
    id
    title
    status
    createdAt
  }
}
