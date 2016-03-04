---
layout : post
title : 转载 - Django admin 定制案例
category : django
date : 2016-01-05 20:00
tags : [django,]
---


总体思路：重写``admin.ModelAdmin``中的某些方法。

{%highlight python linenos %}

from django.db import models
 
class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()
  
    # On Python 3: def __str__(self):
    def __unicode__(self):
        return self.name
  
class Author(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField()
  
    # On Python 3: def __str__(self):
    def __unicode__(self):
        return self.name
  
class Entry(models.Model):
    blog = models.ForeignKey(Blog)
    headline = models.CharField(max_length=255)
    body_text = models.TextField()
    pub_date = models.DateField()
    mod_date = models.DateField()
    authors = models.ManyToManyField(Author)
    n_comments = models.IntegerField()
    n_pingbacks = models.IntegerField()
    rating = models.IntegerField()
  
    # On Python 3: def __str__(self):
    def __unicode__(self):
        return self.headline

{%endhighlight%}

### 类级别权限

默认情况下，superuser可以访问admin界面的所有Model，但有时候只想让一些用户只能访问一些特定的Model。

可以定制自己的User对象的has_perm()方法：
 
{%highlight python linenos%}

class MyUser(AbstractBaseUser):
    ...
    def has_perm(self, perm, obj=None):
        if self.is_superuser:
            return True
        elif self.can_edit:
            if perm=='myapp.add_entry':
                return True
            else:
                return False
        else:
            return False
            
{%endhighlight%}
 
这样superuser具有全部权限。普通user的can_edit属性为True时，就具有了创建Entry实例的权限，其余用户无权限。

也可以定制ModelAdmin的has_add_permission(),has_change_permission(),has_delete_permission()方法：

{%highlight python linenos%}

def has_add_permission(self, request):
    """
    Returns True if the given request has permission to add an object.
    Can be overridden by the user in subclasses.
    """
    opts = self.opts
    codename = get_permission_codename('add', opts)
    if request.user.can_edit:
        return True
    else:
        return request.user.has_perm("%s.%s" % (opts.app_label, codename))

{%endhighlight%}

### 字段级别的权限

不同权限的可以编辑不同的内容，可以通过get_readonly_fileds()来添加字段只读权限。

{%highlight python linenos%}

class EntryAdmin(admin.ModelAdmin):
    list_display=(...)
    search_fields=(...)
    def get_readonly_fields(self,request,obj=None):
        if not request.user.is_superuser and not request.user.can_edit:
            return [f.name for f in self.model._meta.fields]
        return self.readonly_fields
        
{%endhighlight%}

重写Model的save行为

可以直接重写model的save()方法：

{%highlight python linenos%}

from django.db import models
 
class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()
 
    def save(self, *args, **kwargs):
        do_something()
        super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.
        do_something_else()
        
{%endhighlight%}

阻止save()：


{%highlight python linenos%}

from django.db import models
 
class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()
 
    def save(self, *args, **kwargs):
        if self.name == "Yoko Ono's blog":
            return # Yoko shall never have her own blog!
        else:
            super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.
        
{%endhighlight%}

也可以重写ModelAdmin的save_model()方法，根据不同的用户定制不同的save行为：

{%highlight python linenos%}

from django.contrib import admin
 
class ArticleAdmin(admin.ModelAdmin):
    def save_model(self, request, obj, form, change):
        obj.user = request.user
        obj.save()
        
{%endhighlight%}

其中obj是修改后的对象，当新建一个对象时 change = False, 当修改一个对象时 change = True，可以获得修改前的对象：

{%highlight python linenos%}

from django.contrib import admin
class ArticleAdmin(admin.ModelAdmin):
    def save_model(self, request, obj, form, change):
        if change:
            obj_old = self.model.objects.get(pk=obj.pk)
        else:
            obj_old = None
        obj.user = request.user
        obj.save()
        
{%endhighlight%}

### 不同的用户显示不同的数据行，重写列表页面返回的查询集

ModelAdmin提供了一个钩子程序 —— 它有一个名为queryset() 的方法，该方法可以确定任何列表页面返回的默认查询集。

{%highlight python linenos%}

class MyModelAdmin(admin.ModelAdmin):
    def get_queryset(self, request):
        qs = super(MyModelAdmin, self).get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(author=request.user)
        
{%endhighlight%}

### 定制过滤器list_filter

从django.contrib.admin.SimpleListFilter继承一个子类，提供title和parameter_name属性，并重写 lookups和queryset方法。

{%highlight python linenos%}

from datetime import date
 
from django.contrib import admin
from django.utils.translation import ugettext_lazy as _
 
class DecadeBornListFilter(admin.SimpleListFilter):
    # Human-readable title which will be displayed in the
    # right admin sidebar just above the filter options.
    title = _('decade born')
 
    # Parameter for the filter that will be used in the URL query.
    parameter_name = 'decade'
 
    def lookups(self, request, model_admin):
        """
        Returns a list of tuples. The first element in each
        tuple is the coded value for the option that will
        appear in the URL query. The second element is the
        human-readable name for the option that will appear
        in the right sidebar.
        """
        return (
            ('80s', _('in the eighties')),
            ('90s', _('in the nineties')),
        )
 
    def queryset(self, request, queryset):
        """
        Returns the filtered queryset based on the value
        provided in the query string and retrievable via
        `self.value()`.
        """
        # Compare the requested value (either '80s' or '90s')
        # to decide how to filter the queryset.
        if self.value() == '80s':
            return queryset.filter(birthday__gte=date(1980, 1, 1),
                                    birthday__lte=date(1989, 12, 31))
        if self.value() == '90s':
            return queryset.filter(birthday__gte=date(1990, 1, 1),
                                    birthday__lte=date(1999, 12, 31))
 
class PersonAdmin(admin.ModelAdmin):
    list_filter = (DecadeBornListFilter,)

{%endhighlight%}

parameter_name和title是必须的。look_up方法返回出现在列表页右侧过滤器中的选项和描述。parameter_name为附加在url后面get请求的参数名，self.value()返回该参数对应的值。

根据不同的用户定制：

{%highlight python linenos%}

class AuthDecadeBornListFilter(DecadeBornListFilter):
 
    def lookups(self, request, model_admin):
        if request.user.is_superuser:
            return super(AuthDecadeBornListFilter,
                self).lookups(request, model_admin)
 
    def queryset(self, request, queryset):
        if request.user.is_superuser:
            return super(AuthDecadeBornListFilter,
                self).queryset(request, queryset)
{%endhighlight%}

model_admin为ModelAdmin实例：

{%highlight python linenos%}

class AdvancedDecadeBornListFilter(DecadeBornListFilter):
 
    def lookups(self, request, model_admin):
        """
        Only show the lookups if there actually is
        anyone born in the corresponding decades.
        """
        qs = model_admin.get_queryset(request)
        if qs.filter(birthday__gte=date(1980, 1, 1),
                      birthday__lte=date(1989, 12, 31)).exists():
            yield ('80s', _('in the eighties'))
        if qs.filter(birthday__gte=date(1990, 1, 1),
                      birthday__lte=date(1999, 12, 31)).exists():
            yield ('90s', _('in the nineties'))
            
{%endhighlight%}

### 定制搜索功能

{%highlight python linenos%}

class PersonAdmin(admin.ModelAdmin):
    list_display = ('name', 'age')
    search_fields = ('name',)
 
    def get_search_results(self, request, queryset, search_term):
        queryset, use_distinct = super(PersonAdmin, self).get_search_results(request, queryset, search_term)
        try:
            search_term_as_int = int(search_term)
        except ValueError:
            pass
        else:
            queryset |= self.model.objects.filter(age=search_term_as_int)
        return queryset, use_distinct

{%endhighlight%}

queryset是查询集，search_term是搜索词。

### 外键字段过滤

在添加对象时显示外键选项时，太多的选项不太友好，这时候需要过滤出符合要求的对象供选择。

{%highlight python linenos%}

class MyModelAdmin(admin.ModelAdmin):
    def formfield_for_foreignkey(self, db_field, request, **kwargs):
        if db_field.name == "car":
            kwargs["queryset"] = Car.objects.filter(owner=request.user)
        return super(MyModelAdmin, self).formfield_for_foreignkey(db_field, request, **kwargs)

{%endhighlight%}

### 外键改为查询方式

{%highlight python linenos%}

class EntryAdmin(admin.ModelAdmin):
    raw_id_fields = ('blog',)

{%endhighlight%}

#### 参考

* [http://www.cnblogs.com/linxiyue/p/4075048.html](http://www.cnblogs.com/linxiyue/p/4075048.html)