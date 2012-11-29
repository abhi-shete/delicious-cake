===============
Delicious Cake
===============

Delicious Cake is a flexible, Tastypie derived, REST framework for Django.

Over the years I've used both Tastypie and Piston to create RESTful APIs.  While they both have their advantages neither is as flexible as plain old Django views.  


Why Delicious Cake?
===================

Delicious Cake is a framework that removes much of the pain of creating RESTful APIs, without imposing many constraints. 


How is this different from Tastypie?
====================================

Tastypie makes taking your models and quickly exposing them in a RESTful manner extremely easy.  Unfortunately, that's not often the best way to expose an api.  Some functionality is simply difficult to express within Tastypie's constraints.

Tastypie resources tightly couple several features:  List views, detail views, data hydration/dehydration, url construction, and pre-serialization processing of objects.  For simple APIs this can reduce the amount of code necessary to get your project off the ground.  As your project evolves it can become increasingly difficult to express your ideal API with Tastypie.

Delicious Cake is an attempt to take the best of Tastypie and present it in a more flexible form.


This is an experiment 
=====================

I want to get this out quickly to see if there is any interest in this model of API development.  

Both tests and documentation are sparse.  If there's enough interest, test coverage and documentation will become a priority.

Let me know what you think!

mike@theitemshoppe.com


What does it look like?
=======================

**resources.py**:
::

   class CakeListResource(ListResource):
       '''A simple list view'''
       def get(self, request, *args, **kwargs):
           return Cake.objects.all()

       def head(self, request, *args, **kwargs):
           return self.get(request, *args, **kwargs)

       def post(self, request, *args, **kwargs):
           cake_form = CakeForm(request.DATA)

           if not cake_form.is_valid():
               raise ValidationError(cake_form.errors)

           # Return the newly created instance and indicate that 
           # HTTP 201 CREATED should be used in the response.

           # return object, created (boolean)
           return cake_form.save(), True

       def delete(self, request, *args, **kwargs):
           Cake.objects.all().delete()
   
       # Used to get the base uri when paginating.   
       @models.permalink
       def get_resource_uri(self):
           return ('cake-list',)
   
       class Meta(object):
           # See delicious_cake/options.py for more 'Resource' options.
   
           # 'Entity' classes are used to pre-process objects before 
           # serialization.        
   
           # The 'list_entity_cls' will be used to pre-process the returned 
           # objects when viewed as a list.
           list_entity_cls = CakeListEntity
   
           # The 'detail_entity_cls' will be used to pre-process the returned 
           # objects when returned individually.        
           detail_entity_cls = CakeDetailEntity
   
           # If the same representation of the object is used in both list and 
           # details views the 'entity_cls' option can be used
           # (e.g.  entity_cls = CakeDetailEntity) 
   
   
   class CakeDetailResource(DetailResource):
       '''A simple detail view'''
       def get(self, request, *args, **kwargs):
           return get_object_or_404(Cake, pk=kwargs['pk'])
   
       def put(self, request, *args, **kwargs):
           pk = kwargs['pk']
   
           try:
               created = False
               instance = Cake.objects.get(pk=pk)
           except Cake.DoesNotExist:
               created = True
               instance = Cake(id=pk)
   
           cake_form = CakeForm(request.DATA, instance=instance)
   
           if not cake_form.is_valid():
               raise ValidationError(cake_form.errors)
   
           # Return the newly created instance and indicate that 
           # HTTP 201 CREATED should be used in the response.
           # OR
           # Return the updated instance with HTTP 200 OK
           return cake_form.save(), created

       def delete(self, request, *args, **kwargs):
           get_object_or_404(Cake, pk=kwargs['pk']).delete()
   
       def head(self, request, *args, **kwargs):
           return self.get(self, request, *args, **kwargs)
   
       class Meta(object):
           detail_entity_cls = CakeDetailEntity


   class CakeListResourceExtra(ListResource):
       # Add a response header to all responses.
       def process_http_response(self, http_response, entities):
           http_response['X-The-Cake-Is-A-Lie'] = False
   
       # Add a response header to all GET responses.
       def process_http_response_get(self, http_response, entities):
           http_response['X-Cake-Count'] = len(entities)
   
       def get(self, request, *args, **kwargs):
           # Tell the resource to use the 'CakeDetailEntity' instead of the 
           # default ('CakeListEntity' in this case) by specifying 'entity_cls'.
           return ResourceResponse(
              Cake.objects.all(), entity_cls=CakeDetailEntity)

       def post(self, request, *args, **kwargs):
           cake_form = CakeForm(request.DATA)
   
           if not cake_form.is_valid():
               raise ValidationError(cake_form.errors)
   
           cake = cake_form.save()

           # You can return 'ResourceResponse's if you need to 
           # use a custom 'HttpResponse' class or pass in specific parameters to 
           # the 'HttpResponse' class's constructor.  
   
           # For example, in this method we want to return an HTTP 201 (CREATED) 
           # response, with the newly created cake's uri in 'Location' header.  
           # To do this we set the 'response_cls' argument to 'http.HttpCreated' 
           # and add a 'location' key to 'response_kwargs' dict.  
   
           # This is equilivant to returning "cake_form.save(), created"

           # In this case, the value passed into the location parameter of our 
           # 'HttpCreated' response will be  a callable.  When invoked it will be 
           # passed one parameter, the entity created from our cake object.

           # And, just for fun, let's set 'include_entity' to False.
   
           # So again, we'll return HTTP 201 (CREATED), with a Location header,
           # the X-The-Cake-Is-A-Lie header, and no entity body.
   
           return ResourceResponse(
               cake, include_entity=False,
               response_cls=http.HttpCreated,
               response_kwargs={
                   'location': lambda entity: entity.get_resource_uri()})
   
       @models.permalink
       def get_resource_uri(self):
           return ('cake-list-extra',)
   
       class Meta(object):
           entity_cls = CakeListEntity


**urls.py**:
::
   
   urlpatterns = patterns('',
       url(r'^cake/(?P<pk>\d+)/$', CakeDetailResource.as_view(), name='cake-detail'),
       url(r'^cake/$', CakeListResource.as_view(), name='cake-list'),
       url(r'^cake/extra/$', CakeListResourceExtra.as_view(), name='cake-list-extra'),)


**entities.py**:
::       
   
   class CakeEntity(Entity):
       @models.permalink
       def get_resource_uri(self):
           return ('cake-detail', (self.obj.pk,))


   class CakeListEntity(CakeEntity):
       CAKE_TYPE_CHOICES_LOOKUP = dict(Cake.CAKE_TYPE_CHOICES)

       resource_id = fields.IntegerField(attr='pk')
       cake_type = fields.CharField(attr='cake_type')

       def process_cake_type(self, cake_type):
           return self.CAKE_TYPE_CHOICES_LOOKUP.get(cake_type, 'Unknown')


   class CakeDetailEntity(CakeListEntity):
       message = fields.CharField(attr='message')


**models.py**:
::

   class Cake(models.Model):
       CAKE_TYPE_BIRTHDAY = 1
       CAKE_TYPE_GRADUATION = 2
       CAKE_TYPE_SCHADENFREUDE = 3

       CAKE_TYPE_CHOICES = (
           (CAKE_TYPE_BIRTHDAY, u'Birthday Cake',),
           (CAKE_TYPE_GRADUATION, u'Graduation Cake',),
           (CAKE_TYPE_SCHADENFREUDE, u'Shameful Pride Cake',),)

       message = models.CharField(max_length=128)
       cake_type = models.PositiveSmallIntegerField(db_index=True)

