# Django REST Framework – O'quv Markazi Loyihasi (To'liq Qo'llanma)

---

## 1. Loyiha Haqida

Bu loyiha **O'quv Markazi** ni boshqarish uchun yaratilgan to'liq REST API hisoblanadi.  
Quyidagi modellar mavjud:

- **Master** — Fanlar
- **Mentor** — O'qituvchilar (Mentorlar)
- **Group** — Guruhlar
- **Student** — O'quvchilar

Har bir fayl (`models.py`, `serializers.py`, `views.py`) to'liq va izohli tarzda berilgan.

---

## 2. models.py

```python
# models.py
from django.db import models

class Master(models.Model):
    """Fanlar modeli"""
    subject = models.CharField(max_length=100, unique=True)

    def __str__(self):
        return self.subject

    class Meta:
        verbose_name = "Fan"
        verbose_name_plural = "Fanlar"
        ordering = ['subject']


class Mentor(models.Model):
    """Mentor (O'qituvchi) modeli"""
    firstname = models.CharField(max_length=100)
    lastname = models.CharField(max_length=100)
    master = models.ForeignKey(
        Master,
        on_delete=models.CASCADE,
        related_name='mentors'
    )

    def __str__(self):
        return f"{self.firstname} {self.lastname}"

    class Meta:
        verbose_name = "Mentor"
        verbose_name_plural = "Mentorlar"
        ordering = ['firstname']


class Group(models.Model):
    """Guruh modeli"""
    title = models.CharField(max_length=100, unique=True)
    mentor = models.ForeignKey(
        Mentor,
        on_delete=models.CASCADE,
        related_name='groups'
    )

    def __str__(self):
        return self.title

    class Meta:
        verbose_name = "Guruh"
        verbose_name_plural = "Guruhlar"
        ordering = ['title']


class Student(models.Model):
    """O'quvchi modeli"""
    firstname = models.CharField(max_length=100)
    lastname = models.CharField(max_length=100, blank=True, null=True)
    grade = models.IntegerField()

    def __str__(self):
        return f"{self.firstname} {self.lastname or ''}".strip()

    class Meta:
        verbose_name = "O'quvchi"
        verbose_name_plural = "O'quvchilar"
        ordering = ['firstname']
```

---

## 3. serializers.py

```python
# serializers.py
from rest_framework import serializers
from .models import Master, Mentor, Group, Student


# ========================= MASTER SERIALIZER =========================
class MasterSerializer(serializers.ModelSerializer):
    class Meta:
        model = Master
        fields = ['id', 'subject']
        read_only_fields = ['id']

    def validate_subject(self, value):
        value = value.strip()
        if len(value) < 3:
            raise serializers.ValidationError("Fan nomi kamida 3 ta belgi bo'lishi kerak!")

        if Master.objects.filter(subject__iexact=value).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError("Bu fan nomi allaqachon mavjud!")

        return value.title()

    def to_representation(self, instance):
        data = super().to_representation(instance)
        data['subject'] = instance.subject.title()
        return data


# ========================= MENTOR SERIALIZER =========================
class MentorSerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()
    master_subject = serializers.CharField(source='master.subject', read_only=True)
    group_count = serializers.SerializerMethodField()

    class Meta:
        model = Mentor
        fields = [
            'id', 'firstname', 'lastname', 'full_name',
            'master_subject', 'master', 'group_count'
        ]
        read_only_fields = ['id', 'master_subject', 'group_count']
        extra_kwargs = {
            'firstname': {'required': True, 'min_length': 2},
            'lastname': {'required': True, 'min_length': 2},
        }

    def get_full_name(self, obj):
        return f"{obj.firstname} {obj.lastname}".strip()

    def get_group_count(self, obj):
        return obj.groups.count()

    def validate(self, attrs):
        firstname = attrs.get('firstname', '').strip()
        lastname = attrs.get('lastname', '').strip()
        if firstname.lower() == lastname.lower():
            raise serializers.ValidationError({"lastname": "Familiya ism bilan bir xil bo'lishi mumkin emas!"})
        return attrs

    def create(self, validated_data):
        validated_data['firstname'] = validated_data['firstname'].strip().title()
        validated_data['lastname'] = validated_data['lastname'].strip().title()
        return super().create(validated_data)

    def update(self, instance, validated_data):
        if 'firstname' in validated_data:
            validated_data['firstname'] = validated_data['firstname'].strip().title()
        if 'lastname' in validated_data:
            validated_data['lastname'] = validated_data['lastname'].strip().title()
        return super().update(instance, validated_data)

    def to_representation(self, instance):
        data = super().to_representation(instance)
        data['meta'] = {"groups_count": instance.groups.count()}
        return data


# ========================= GROUP SERIALIZER =========================
class GroupSerializer(serializers.ModelSerializer):
    mentor_fullname = serializers.StringRelatedField(source='mentor', read_only=True)
    mentor_firstname = serializers.CharField(source='mentor.firstname', read_only=True)
    student_count = serializers.SerializerMethodField()

    class Meta:
        model = Group
        fields = [
            'id', 'title', 'mentor_fullname', 'mentor_firstname',
            'mentor', 'student_count'
        ]
        read_only_fields = ['id', 'mentor_fullname', 'mentor_firstname', 'student_count']
        extra_kwargs = {'title': {'required': True, 'min_length': 3}}

    def get_student_count(self, obj):
        return 0   # Kelajakda Student bilan bog'lash uchun

    def validate_title(self, value):
        value = value.strip()
        if len(value) < 3:
            raise serializers.ValidationError("Guruh nomi kamida 3 ta belgi bo'lishi kerak!")

        if Group.objects.filter(title__iexact=value).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError("Bu nomdagi guruh allaqachon mavjud!")

        return value.title()

    def to_representation(self, instance):
        data = super().to_representation(instance)
        data['title'] = instance.title.title()
        return data


# ========================= STUDENT SERIALIZER =========================
class StudentSerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()

    class Meta:
        model = Student
        fields = ['id', 'firstname', 'lastname', 'full_name', 'grade']
        read_only_fields = ['id']
        extra_kwargs = {
            'firstname': {'required': True, 'min_length': 2},
            'grade': {'min_value': 1, 'max_value': 11},
        }

    def get_full_name(self, obj):
        lastname = obj.lastname or ''
        return f"{obj.firstname} {lastname}".strip()

    def validate_grade(self, value):
        if not (1 <= value <= 11):
            raise serializers.ValidationError("Sinf (grade) 1 dan 11 gacha bo'lishi kerak!")
        return value

    def validate(self, attrs):
        firstname = attrs.get('firstname', '').strip()
        lastname = attrs.get('lastname', '').strip()
        if firstname and lastname and firstname.lower() == lastname.lower():
            raise serializers.ValidationError({"lastname": "Familiya ism bilan bir xil bo'lishi mumkin emas!"})
        return attrs

    def create(self, validated_data):
        validated_data['firstname'] = validated_data['firstname'].strip().title()
        if validated_data.get('lastname'):
            validated_data['lastname'] = validated_data['lastname'].strip().title()
        return super().create(validated_data)

    def to_representation(self, instance):
        data = super().to_representation(instance)
        data['full_name'] = self.get_full_name(instance)
        return data
```

---

## 4. views.py

```python
# views.py
from rest_framework import viewsets, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticatedOrReadOnly, IsAdminUser
from django_filters.rest_framework import DjangoFilterBackend
from .models import Master, Mentor, Group, Student
from .serializers import MasterSerializer, MentorSerializer, GroupSerializer, StudentSerializer


class MasterViewSet(viewsets.ModelViewSet):
    queryset = Master.objects.all()
    serializer_class = MasterSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    
    search_fields = ['subject']
    ordering_fields = ['subject']
    ordering = ['subject']

    def get_permissions(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            return [IsAdminUser()]
        return super().get_permissions()

    @action(detail=True, methods=['get'])
    def mentors(self, request, pk=None):
        master = self.get_object()
        serializer = MentorSerializer(master.mentors.all(), many=True)
        return Response(serializer.data)


class MentorViewSet(viewsets.ModelViewSet):
    queryset = Mentor.objects.all()
    serializer_class = MentorSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    
    filterset_fields = ['master', 'master__subject']
    search_fields = ['firstname', 'lastname']
    ordering_fields = ['firstname', 'lastname']
    ordering = ['firstname']

    def get_permissions(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            return [IsAdminUser()]
        return super().get_permissions()

    @action(detail=True, methods=['get'])
    def groups(self, request, pk=None):
        mentor = self.get_object()
        serializer = GroupSerializer(mentor.groups.all(), many=True)
        return Response(serializer.data)


class GroupViewSet(viewsets.ModelViewSet):
    queryset = Group.objects.all()
    serializer_class = GroupSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    
    filterset_fields = ['mentor']
    search_fields = ['title']
    ordering_fields = ['title']
    ordering = ['title']

    def get_permissions(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            return [IsAdminUser()]
        return super().get_permissions()


class StudentViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    
    filterset_fields = ['grade']
    search_fields = ['firstname', 'lastname']
    ordering_fields = ['firstname', 'grade']
    ordering = ['firstname']

    def get_permissions(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            return [IsAdminUser()]
        return super().get_permissions()

    @action(detail=False, methods=['get'])
    def by_grade(self, request):
        """Sinf bo'yicha o'quvchilarni ko'rsatish yoki statistika"""
        grade = request.query_params.get('grade')
        if grade:
            students = self.queryset.filter(grade=grade)
            serializer = self.get_serializer(students, many=True)
            return Response(serializer.data)

        from django.db.models import Count
        stats = Student.objects.values('grade').annotate(count=Count('id')).order_by('grade')
        return Response({"stats": list(stats)})
```

---

## 5. urls.py (Router bilan)

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import MasterViewSet, MentorViewSet, GroupViewSet, StudentViewSet

router = DefaultRouter()
router.register(r'masters', MasterViewSet, basename='master')
router.register(r'mentors', MentorViewSet, basename='mentor')
router.register(r'groups', GroupViewSet, basename='group')
router.register(r'students', StudentViewSet, basename='student')

urlpatterns = [
    path('', include(router.urls)),
]
```

---
