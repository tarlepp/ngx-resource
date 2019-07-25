[![npm version](https://img.shields.io/npm/v/%40ngx-resource%2Fcore.svg)](https://www.npmjs.com/package/@ngx-resource/core)

# @ngx-resource/core
Resource Core is an evolution of ngx-resource lib which provides flexibility for developers. Each developer can implement their own request handlers to easily customize the behavior.
In fact, `@ngx-resource/core` is an abstract common library which uses `ResourceHandler` to make requests, so it's even possible to use the lib on node.js server side with typescript. You just need to implement a `ResourceHandler` for it.

All my examples will be based on angular 4.4.4+

### Known ResourceHandlers
* `@ngx-resource/handler-ngx-http`. Based on `HttpClient` from `@angular/common/http`. Includes `ResourceModule.forRoot`.
* `@ngx-resource/handler-ngx-http-legacy`. Based on `Http` from `@angular/http`. Includes `ResourceModule.forRoot`.
* `@ngx-resource/handler-cordova-advanced-http`. Based on [Cordova Plugin Advanced HTTP](`https://github.com/silkimen/cordova-plugin-advanced-http`).
* `@ngx-resource/handler-fetch`. Besed on [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API). Not yet created.


## Creation of Resource class

```typescript
@Injectable()
@ResourceParams({
  // IResourceParams
  pathPrefix: '/auth'
})
export class MyAuthResource extends Resource {

  @ResourceAction({
    // IResourceAction
    method: ResourceRequestMethod.Post,
    path: '/login'
  })
  login: IResourceMethod<{login: string, password: string}, IReturnData>; // will make an post request to /auth/login

  @ResourceAction({
    // IResourceAction
    //method: ResourceRequestMethod.Get is by default
    path: '/logout'
  })
  logout: IResourceMethod<void, void>;
  
  constructor(handler: ResourceHandler) {
    super(handler);
  }
  
}

@Injectable()
@ResourceParams({
  // IResourceParams
  pathPrefix: '/user'
})
export class UserResource extends Resource {
  
  @ResourceAction({
    path: '/{!id}'
  })
  getUser: IResourceMethodPromise<{id: string}, IUser>; // will call /user/id
  
  @ResourceAction({
    method: ResourceRequestMethod.Post
  })
  createUser: IResourceMethodPromiseStrict<IUser, IUserQuery, IUserPathParams, IUser>;
  
  @ResoucreAction({
    path: '/test/data',
    asResourceResponse: true
  })
  testUserRequest: IResourceMethodPromiseFull<{id: string}, IUser>; // will call /test/data and receive repsponse object with headers, status and body
  
  constructor(restHandler: ResourceHandler) {
    super(restHandler);
  }
  
}

// Using created Resource
@Injectable
export class MyService {
  
  private user: IUser = null;

  constructor(private myResource: MyAuthResource, private userResource: UserResource) {}
  
  doLogin(login: string, password: string): Promise<any> {
    return this.myResource.login({login, password});
  }
  
  doLogout(): Promise<any> {
    return this.myResource.logout();
  }
  
  async loginAndLoadUser(login: string, password: string, userId: string): Promise<any> {
    await this.doLogin(login, password);
    this.user = await this.userResource.getUser({id: userId});
  }
  
}

```

Final url is generated by concatination of `$getUrl`, `$getPathPrefix` and `$getPath` methods of `Resource` base class.

### [IResourceParams](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/Declarations.ts#L2-L23) 

Is used by `ResourceParams` decorator for class decoration

List of params:
* `url?: string;` - url of the api server; *default `''`*
* `pathPrefix?: string;` - path prefix of the api; *default `''`*
* `path?: string;` - path of the api; *default `''`*
* `headers?: any;` - headers; *default `{}`*
* `body?: any;` - default body; *default `null`*
* `params?: any;` - default url params; *default `null`*
* `query?: any;` - defualt query params; *default `null`*
* `rootNode?: string;` - key to assign all body; *default `null`*
* `removeTrailingSlash?: boolean;` - *default `true`*
* `addTimestamp?: boolean | string;` - *default `false`*
* `withCredentials?: boolean;` - *default `false`*
* `lean?: boolean;` - do no add `$` properties on result. Used only with `toPromise: false` *default `false`*
* `mutateBody?: boolean;` - if need to mutate provided body with response body. *default `false`*
* `returnAs?: ResourceActionReturnType;` - what type response should be returned by action/method . *default `ResourceActionReturnType.Observable`*
* `keepEmptyBody?: boolean;` - if need to keep empty body object `{}`
* `requestBodyType?: ResourceRequestBodyType;` - request body type. *default: will be detected automatically*.
Check for possible body types in the sources of [ResourceRequestBodyType](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/Declarations.ts#L114-L122). Type detection algorithm [check here](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/ResourceHelper.ts#L12-L34).
* `responseBodyType?: ResourceResponseBodyType;` - response body type. *default: `ResourceResponseBodyType.JSON`* Possible body type can be checked here [ResourceResponseBodyType](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/Declarations.ts#L124-L129).

### [IResourceAction](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/Declarations.ts#L2-L31) 

Is used by `ResourceAction` decorator for methods.

List of params (is all of the above) plus the following:
* `method?: ResourceRequestMethod;` - method of request. *Default `ResourceRequestMethod.Get`*. All possible methods listed in [ResourceRequestMethod](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/Declarations.ts#L131-L139)
* `expectJsonArray?: boolean;` - if expected to receive an array. The field is used only with `toPromise: false`. *Default `false`*.
* `resultFactory?: IResourceResultFactory;` - custom method to create result object. *Default: `returns {}`*
* `map?: IResourceResponseMap;` - custom data mapping method. *Default: `returns without any changes`*
* `filter?: IResourceResponseFilter;` - custom data filtering method. *Default: `returns true`*

### [ResourceGlobalConfig](https://github.com/troyanskiy/ngx-resource-core/blob/master/src/ResourceGlobalConfig.ts)
Mainly used to set defaults.


## Models
An object with built-in in methods to save, update, and delete a model.
Here is an example of a `User` model.

Note: UserResource should be injected at the beginning in order to use static
model method like `User.get(<id>)`, `User.query()`, `User.remove(<id>)`

```typescript

export interface IPaginationQuery {
  page?: number;
  perPage?: number;
}

export interface IGroupQuery extends IPaginationQuery {
  title?: string;
}

export interface IUserQuery extends IPaginationQuery {
  firstName?: string;
  lastName?: string;
  groupId?: number;
}

export interface IUser {
  id: number;
  userName: string;
  firstName: string;
  lastName: string;
  groupId: string;
}

export class GroupResource extends ResourceCRUDPromise<IGroupQuery, Group, Group> {
  
  constructor(restHandler: ResourceHandler) {
    super(restHandler);
  }
  
  $resultFactory(data: any, options: IResourceActionInner = {}): any {
    return new Group(data);
  }
  
}

export class Group extends ResourceModel {
  
  readonly $resource = GroupResource;

  id: number;
  title: string;
  
  constructor(data?: IGroup) {
    super();
    if (data) {
      this.$setData(data);
    }
  }
  
  $setData(data: IGroup) {
    this.id = data.id;
    this.title = data.title;
  }
  
}

export class UserResource extends ResourceCRUDPromise<IUserQuery, User, User> {
  
  constructor(restHandler: ResourceHandler) {
      super(restHandler);
  }
  
  $resultFactory(data: any, options: IResourceActionInner = {}): any {
    return new User(data);
  }
  
}

export class User extends ResourceModel implements IUser {

  readonly $resource = UserResource;

  id: number;
  userName: string;
  firstName: string;
  lastName: string;
  groupId: string;
  
  fullName: string; // generated from first name and last name
  
  constructor(data?: IUser) {
    super();
    if (data) {
      this.$setData(data);
    }
  }
  
  $setData(data: IUser): this {
    Object.assign(this, data);
    this.fullName = `${this.firstName} ${this.lastName}`;
    return this;
  }
  
  toJSON() {
    // here i'm using lodash lib pick method.
    return _.pick(this, ['id', 'firstName', 'lastName', 'groupId']);
  }

}


// example of using the staff
async someMethodToCreateGroupAndUser() {

  // Creation a group
  const group = new Group();
  group.title = 'My group';
  
  // Saving the group
  await group.$save();
  
  // Creating an user
  const user = new User({
    userName: 'troyanskiy',
    firstName: 'firstName',
    lastName: 'lastName',
    groupId: group.id
  });
  
  // Saving the user
  await user.$save();
  
  
  // Query data from server
  
  const user1 = await this.userResource.get('1');
  
  // or
  
  const user2: User = await User.get('id');
  
}

```


## QueryParams Conversion

You can define the way query params are converted.
Set the global config at the root of your app.

`ResourceGlobalConfig.queryMappingMethod = ResourceQueryMappingMethod.<CONVERSION_STRATEGY>`

```
{
  a: [{ b:1, c: [2, 3] }]
}
```

With `<CONVERSION_STRATEGY>` being one of the following:

#### Plain (default)
No conversion at all

Output: `?a=[Object object]`

#### Bracket
All array elements will be indexed

Output: `?a[0][b]=10383&a[0][c][0]=2&a[0][c][1]=3`

#### JQueryParamsBracket 
Implements the standard $.params way of converting

Output: `?a[0][b]=10383&a[0][c][]=2&a[0][c][]=3`

