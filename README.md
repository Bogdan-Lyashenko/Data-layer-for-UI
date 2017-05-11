# Data layer for UI

### The problem 

How do we work with API to get some data from back-end? In the worst case scenario it looks just like direct call of *fetch* or *$.ajax* from a view layer.  

For example, we fetch some user details to display it on our 'Home' page:
```javascript
//home/users.js
fetch(‘api/users/get?id=’ + userId)
```
and, the same one API call in several more places in the code and so on.
```javascript
//details/form.js
fetch(‘api/users/get?id=’ + userId)
```
Looks like un-controlled code duplication, right?
But usually, it even has a bit more lines of code, additional parameters and response data preparing, etc. So it comes like that instead.
```javascript
const url = ‘api/users/get?id=’ + 'username.lastname.id';

if (specialUsers.contains(userId)) {
    url += '?identity-prove=keycode';
}

fetch(url, {
    method: 'post',
    headers: {
        Authority: 'tokenAjuib34=='
    }
}).then((data)=> {
    if (!data) {
        return Promise.reject('Got empty profile data');
    }
    
    return data.friends.map((friend)=> {
        return {
            ...friend,
            importance: GLOBAL_IMPORTANCE_CODE
        };
    });
});
```
So, you got the idea. And now, multiply this code in 2x times (because we have it in two files), and imagine for 5x more API calls, and several more different API providers. Sounds crazy. But we can fix it. First and (obviously) not right way to do that is make small refactoring and move all API call related code to separate method and maybe to separate file.  

```javascript
//api/user.js
const USER_URL = ‘api/users/get?id=’;

export const getUserById = (id) => {
  const url = USER_URL; 
		
  if (specialUsers.contains(userId)) {
      url += '?identity-prove=keycode';
  }

  fetch(url, {
      method: 'post',
      headers: {
          Authority: 'tokenAjuib34=='
      }
  }).then((data)=> {
      if (!data) {
          return Promise.reject('Got empty profile data');
      }

      return data.friends.map((friend)=> {
          return {
              ...friend,
              importance: GLOBAL_IMPORTANCE_CODE
          };
      });
  });
}
```
and then we can use it from our controllers etc. Like
```javascript
//details/form.js
import {getUserById} from 'api/services/user';
...
getUserById('username1.lastname1.id').then((data)=> {	
    //use user data for UI form
});
```
and from another place
```javascript
//home/users.js
import {getUserById} from 'api/services/user';
...
getUserById('username2.lastname2.id').then((data)=> {	
    //display user on home page
});
``` 
And everything seems to be working fine, we can reduce code duplication so much, we even have USER_URL as constant in one place, so if url is changed on server side, we change it here as well, everything is under control, right?.   No, that's just an illusion. 

You will be surprised how soon you will need to do change like 
```javascript
//api/user.js
...
export const getUserById = (id, homePageOnlyFlag) => {
  const url = USER_URL; 
  
  if (homePageOnlyFlag) {
      url += '?target=from-home';
  }

  fetch(url, {
      method: 'post',
      headers: {
...
``` 
to extend or change behaviour for calls from 'Home' page only. You can mess it up with more flags, sometimes it will be easy when it's just extension of url, sometimes it will be harder, when you need to remove some already existing behaviour. And some moment it will be so messy that you will just copy half of the logic to new method 'getUserByIdCallFromHomePage' to avoid that flags like 'homePageOnlyFlag' and will use separate method for the same API call from different part of your application. That's exactly not that thing which we were expecting to have when started refactoring, right? That actually happened because  the way how we refactored it was wrong from the beginning. 

There is a key thing in work with server API calls, it's hard to standardize configuration and parameters for different calls of one API end-point. Well, it's possible to put some common behaviour as 'basic' (or 'default') directly to the call configuration, but it's on our own risk, so more 'basic' configuration we have, so higher risk to face the case where some logic from default behaviour is conflicting with currently needed case. And vise-versa, so less 'default' behaviour we define, so more duplicated code we put into parents code (files), which trigger API call. We, kind of need 'golden middle way' for that. 

### The solution 
The solution is based on the way how we plan to handle required mutations, that behaviour changes which are needed for some corner cases, and, some additional abstractions, to make the code flexible. 
The key points are:
* A parent should have ability to change API call behaviour externally, instead of only saying to API service what is needed right now. It will help to avoid logic for all cases inside 'getUserById' method. 
* Postponed API call. A parent should trigger call when it's known that call is configured properly for exact case.
* API call method (like 'getUserById') should contain 'current call parameters' state. This will make possible to change configuration at any moments of time.
* API call method should provide interfaces to modify parameters state or add pre/post interceptors for call.
* An abstractions layer of models and services should be added

Alright, let's check the code can looks like
```javascript
//home/users.js
...
import user from 'api/models/user';
...
user.getDetails('username2.lastname2.id').perfrom().then((data)=> {	
    //display user on home page
});
```
The first thing you can notice is path to 'user', it's 'api/models/user'.
The directories sctructure could be as following
```javascript
project/
-/api/
--/models
---/user
---/post
---/...
--/services
---/friends
```
The idea is to keep models as 'small building blocks', and more complex logic, which involves several models or several API calls to get final data put to services. 
So, method in a model can be like:
```javascript
//api/models/user.js
...
const getDetails = (userId) => {	
    //one API call
});
```
and let's check method in a service. For example, we can imagine the next scenario: we want to know all friends who liked our last post. According to API implementation we need to combine two requests, get last user post and then get post 'likes'.
```javascript
//api/services/friends.js
import user from 'api/models/user';
import post from 'api/models/post';
...
const getFriendLikes = (userId) => {
    //1) user.getLastPost
    //2) post.getFriendsLikeForPost
});
```

There is an important moment, despite  on the fact services includes  models, 'services layer' in not above 'models level', they both are on the same abstraction level, you can use both of them directly from 'view layer' (controllers, etc.). That's important because if put models below services, half of services interfaces will be idle wrappers of models methods, like
```javascript
//api/services/friends.js
import user from 'api/models/user';
import post from 'api/models/post';
import like from 'api/models/like';
...
const getUser = () => {	
    user.getUser()   
});

const getPost = () => {	
    post.getPost()   
});

const getLike = () => {	
    like.getLike()   
});
```
it looks and actually is meaningless, so, just avoid this.

Alright, with a structure it is clear. Let's see initial problem with basic API parameters.
```javascript
//home/users.js
...
user.getDetails('username2.lastname2.id')
    .perfrom()
    .then((data)=> 
        //display user on home page
    });
```
Your remember the point we've mentioned about postponed call. You can see 'user.getDetails' returns something else now, not a promise as we used to see. *That's a key trick.*.
Imagine that we allowed to do something like that
```javascript
//home/users.js
...
user.getDetails('userId')
  .addHeaders({
      ['Header-Only-For-Home-Page']: 'additionalToken=='
  })
  .removeUrlParams(['listOffest'])
  .addInterceptor({
  	postCall: () => {
    	logger.info('Heave API call performed from home page.')
    }
  })
  .perfrom()
  .then((data)=> {	
      //display user on home page
  });
```  

Looks pretty powerful, right. Remember, we still have some default behavior defined in the model, but exactly for one API call case we can change it in any ways we want without affecting other call, without modifying model implementation.

Alright, let's check what's inside 'user.getDetails' method. 

```javascript
//api/models/user.js
...
getDetails = (userId) => {	
    const apiConfigCall = new ApiCallConfigurationObject();
    
    apiConfigCall.setBaseUrl(URL_CONST.USER_DETAILS);
    apiConfigCall.addUrlParams(userId);
    
    return apiConfigCall;
}  
```

Obviously, it's some special object which contains methods to change request state parameters. Let's see below the pseudo-code how it can be implemented

```javascript
//ApiCallConfigurationObject
class ApiCallConfigurationObject() {
  const state = {
      urlParams: {},
      headers: {},
      interceptors: {},
      ...
  }

  addHeaders(config) {
      //for each header in config
      //put it into state.headers map

     return this;
  }
    
  perform() {  	
    //gather all config from this.state and prepate it for fetch call
    return fetch(this.getCallConfiguration());
  }
  ...
}

```
Alright, now it looks pretty straightforward.  ApiCallConfigurationObject contains all logic to manage values for 'state',  and then, when parent call 'perform'  method, the data from state will be gathered and converted into right format to use for fetch call.

Then, as a next step, we can go furher and add manging of all user.getDetails calls on global level.
It can be something like

```javascript
//init.js
import user from 'api/models/user';
...
user.global().
  .addInterceptor({
  	preCall: () => {
    	if (isNotAllowed) {
	    logger.info('API call for User data is temporary restricted.')
	    return Promise.reject();
        }
    }
});

```
Look pretty powerful, right?  In fact, you can extend it as far as you want, because now you have endpoint to manage work with back-end. End-point which was hard to imagine with implementation when you just use 
```javascriptfetch
('/url/id') 
```

directly from view layer. Nice.

### More complexity?
What if you have a lot of APIs and do not want to be affected too much by each its change? Well, if your API is pretty stable I think you should not over-complicate things, otherwise, one more good thing for you is *data-source* layer.
Data-sources layer helps you to organize loose coupling with APIs end-points.
Remember the implementation of user model method?
```javascript
//api/models/user.js
...
getDetails = (userId) => {	
    const apiConfigCall = new ApiCallConfigurationObject();
    
    apiConfigCall.setBaseUrl(URL_CONST.USER_DETAILS);
    apiConfigCall.addUrlParams(userId);
    
    return apiConfigCall;
}  
```
As you can see it's tightly coupled with URL_CONST.USER_DETAILS for exact data-source (like you-api-provider-for-user.com/user). 
But what if you can process current source dynamicaly? like

```javascript
//api/models/user.js
...
getDetails = (userId) => {	
    const currentDataSource = getCurrentSource();
    ...
    return currentDataSource.getDetails(userId);
}  
```
And getCurrentSource is a method which depends on configuration return you current data source implementation.
