#NODE ACL - Access Control Lists for Node

此模块是一个简约的ACL实现，受到了Zend_ACL 所使用的Redis进行持久化的启发。

当你在开发一刚网站或者应用时，你会很快发现仅仅依靠session无法更好的保护可用资源的安全。
为了避免一些用户恶意访问另外一些用户的私密信息，我们的程序经常会比预期的药复杂。
采用ACL可用通过更加灵活和巧妙的方式来解决这一问题。


创建角色，并将分配给用户，又时候我们可用将需要管理的每一个用户分配一刚角色，以细化我们管理的颗粒度。
通过给“＊”分配权限对其他用户进行授权。

##功呢

- 用户
- 角色
- 层次
- 资源
- Express middleware for protecting resources.
- Robust implementation with good unit test coverage.

##安装

使用 npm:

	npm install acl

##文档

* [addUserRoles](#addUserRoles)
* [removeUserRoles](#removeUserRoles)
* [userRoles](#userRoles)
* [addRoleParents](#addRoleParents)
* [removeRole](#removeRole)
* [removeResource](#removeResource)
* [allow](#allow)
* [removeAllow](#removeAllow)
* [allowedPermissions](#allowedPermissions)
* [isAllowed](#isAllowed)
* [areAnyRolesAllowed](#areAnyRolesAllowed)
* [whatResources](#whatResources)
* [clean](#clean)
* [middleware](#middleware)

##例子

Create your acl module by requiring it and instantiating it with a valid node_redis instance and an optional prefix:

	var acl = require('acl')
	acl = new acl(redisClient)

All the following functions take a callback with an err parameter as last parameter. We omit it in the examples for simplicity.

Create roles implicitly by giving them permissions:

	// guest is allowed to view blogs
	acl.allow('guest', 'blogs', 'view')

	// allow function accepts arrays as any parameter
	acl.allow('member', 'blogs', ['edit','view', 'delete'])


Users are likewise created implicitly by assigning them roles:

	acl.addUserRoles('joed', 'guest')


Hierarchies of roles can be created by assigning parents to roles:

	acl.addRoleParents('baz', ['foo','bar'])


Note that the order in which you call all the functions is irrelevant (you can add parents first and assign permissions to roles later)

	acl.allow('foo', ['blogs','forums','news'], ['view', 'delete'])


Use the wildcard to give all permissions:

	acl.allow('admin', ['blogs','forums'], '*')


Sometimes is necessary to set permissions on many different roles and resources. This would
lead to unnecessary nested callbacks for handling errors. Instead use the following:

	acl.allow([{roles:['guest','member'], 
                    allows:[
                          {resources:'blogs', permissions:'get'},
                          {resources:['forums','news'], permissions:['get','put','delete']}]
				},
			    {roles:['gold','silver'], 
                    allows:[
                          {resources:'cash', permissions:['sell','exchange']},
                          {resources:['account','deposit'], permissions:['put','delete']}]
				}
				]})

You can check if a user has permissions to access a given resource with *isAllowed*:

	acl.isAllowed('joed', 'blogs', 'view', function(err, res){
		if(res){
			console.log("User joed is allowed to view blogs")
		}
	}


Of course arrays are also accepted in this function:

	acl.isAllowed('jsmith', 'blogs', ['edit','view','delete'])

Note that all permissions must be full filed in order to get *true*.


Sometimes is necessary to know what permissions a given user has over certain resources:

	acl.allowedPermissions('james', ['blogs','forums'], function(err, permissions){
		console.log(permissions)
	})

It will return an array of resource:[permissions] like this:

	[{'blogs' : ['get','delete']},
     {'forums':['get','put']}]


Finally, we provide a middleware for Express for easy protection of resources. 

	acl.middleware()

We can protect a resource like this:

	app.put('/blogs/:id', acl.middleware(), function(req, res, next){…}

The middleware will protect the resource named by *req.url*, pick the user from *req.session.userId* and check the permission for *req.method*, so the above would be equivalent to something like this:

	acl.isAllowed(req.session.userId, '/blogs/12345', 'put')

The middleware accepts 3 optional arguments, that are useful in some situations. For example, sometimes we 
cannot consider the whole url as the resource:

	app.put('/blogs/:id/comments/:commentId', acl.middleware(3), function(req, res, next){…}

In this case the resource will be just the three first components of the url (without the ending slash).

It is also possible to add a custom userId or check for other permissions than the method:

	app.put('/blogs/:id/comments/:commentId', acl.middleware(3, 'joed', 'post'), function(req, res, next){…}


## Methods

<a name="addUserRoles"/>
### addUserRoles( userId, roles, function(err) )

Adds roles to a given user id.

__Arguments__
 
    userId   {String} User id.
    roles    {String|Array} Role(s) to add to the user id.
    callback {Function} Callback called when finished.

---------------------------------------

<a name="removeUserRoles"/>
### removeUserRoles( userId, roles, function(err) )
  
Remove roles from a given user.

__Arguments__

    userId   {String} User id.
    roles    {String|Array} Role(s) to remove to the user id.
    callback {Function} Callback called when finished.

---------------------------------------

<a name="userRoles" />
### userRoles( userId, function(err, roles) )

Return all the roles from a given user.

__Arguments__
  
    userId   {String} User id.
    callback {Function} Callback called when finished.

---------------------------------------

<a name="addRoleParents" />
### addRoleParents( role, parents, function(err) )

Adds a parent or parent list to role.

__Arguments__

    role     {String} User id.
    parents  {String|Array} Role(s) to remove to the user id.
    callback {Function} Callback called when finished.

---------------------------------------

<a name="removeRole" />
### removeRole( role, function(err) )
  
Removes a role from the system.

__Arguments__
  
    role     {String} Role to be removed
    callback {Function} Callback called when finished.

---------------------------------------

<a name="removeResource" />
### removeResource( resource, function(err) )
  
Removes a resource from the system

__Arguments__
  
    resource {String} Resource to be removed
    callback {Function} Callback called when finished.

---------------------------------------

<a name="allow" />
### allow( roles, resources, permissions, function(err) )

Adds the given permissions to the given roles over the given resources.

__Arguments__
  
    roles       {String|Array} role(s) to add permissions to.
    resources   {String|Array} resource(s) to add permisisons to.
    permissions {String|Array} permission(s) to add to the roles over the resources.
    callback    {Function} Callback called when finished.
  

### allow( permissionsArray, function(err) )
  
__Arguments__

    permissionsArray {Array} Array with objects expressing what permissions to give.
       [{roles:{String|Array}, allows:[{resources:{String|Array}, permissions:{String|Array}]]
  
    callback         {Function} Callback called when finished.

---------------------------------------

<a name="removeAllow" />
###  removeAllow( role, resources, permissions, function(err) )

Remove permissions from the given roles owned by the given role.

Note: we loose atomicity when removing empty role_resources.

__Arguments__
  
    role        {String}
    resources   {String|Array}
    permissions {String|Array}
    callback    {Function}

---------------------------------------

<a name="allowedPermissions" />
### allowedPermissions( userId, resources, function(err, obj) )

Returns all the allowable permissions a given user have to
access the given resources.
  
It returns an array of objects where every object maps a 
resource name to a list of permissions for that resource.

__Arguments__
  
    userId    {String} User id.
    resources {String|Array} resource(s) to ask permissions for.
    callback  {Function} Callback called when finished.

---------------------------------------

<a name="isAllowed" />
### isAllowed( userId, resource, permissions, function(err, allowed) )
  
Checks if the given user is allowed to access the resource for the given 
permissions (note: it must fulfill all the permissions).

__Arguments__
  
    userId      {String} User id.
    resource    {String|Array} resource(s) to ask permissions for.
    permissions {String|Array} asked permissions.
    callback    {Function} Callback called wish the result.

---------------------------------------
<a name="areAnyRolesAllowed" />
### areAnyRolesAllowed( roles, resource, permissions, function(err, allowed) )
  
Returns true if any of the given roles have the right permissions.

__Arguments__
  
    roles       {String|Array} Role(s) to check the permissions for.
    resource    {String} resource(s) to ask permissions for.
    permissions {String|Array} asked permissions.
    callback    {Function} Callback called wish the result.

---------------------------------------
<a name="whatResources" />
### whatResources(role, function(err, {resourceName: [permissions]})

Returns what resources a given role has permissions over.

__Arguments__

    role        {String|Array} Roles
    callback    {Function} Callback called with the result.

whatResources(role, permissions, function(err, resources) )
    
Returns what resources a role has the given permissions over.

__Arguments__
  
    role        {String|Array} Roles
    permissions {String[Array} Permissions
    callback    {Function} Callback called wish the result.

---------------------------------------
<a name="clean" />
### clean ()

Cleans all the keys with the given prefix from redis.
    
Note: this operation is not reversible!.

---------------------------------------

<a name="middleware" />
### middleware( [numPathComponents, userId, permissions] )

Middleware for express. 

__Arguments__

    numPathComponents {Number} number of components in the url to be considered part of the resource name.
    userId 			  {String} the user id for the acl system (or if not specified, req.userId)
    permissions 	  {Array} the permissions to check for.

##Tests

Run tests with vows:
 	vows test/*

##Persistence

Currently the persistence is achieved using redis, which was chosen due to its excellent support for
set operations that are heavily used by acl. 

## Future work

- Support for denials (deny a role a given permission)


##License 

(The MIT License)

Copyright (c) 2011 Manuel Astudillo <manuel@optimalbits.com>

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
