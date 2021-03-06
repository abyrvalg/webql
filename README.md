# liteQL

LiteQL is a environment agnostic library which allows to get data from multiple resources using queries. It also can be used as a server API.

Source: https://github.com/abyrvalg/liteql

## How to use
### Use case 1. Single environment
Adding resources:
```javascript
    const liteql = new (require('liteql'))();
    liteql.addResources({
        test(){
            return "Test resource"
        },
        testWithParam(param){
            return "your param is "+param
        }
    });
```
Getting resource by query:
```javascript
    liteql.call(["test", {"testWithParam" : ["myParam"]}]).then((result)=>{
        console.log(result); //{test: "Test resource", testWithParam : "your param is myParam"}
    });
```
### Use case 2. Get data from server 
Server:
```javascript
    const express = require('express');
    const liteql = new (require('liteql'))();
    var app = express();    
    liteql.addResources({
        test(){
            return "Test resource"
        },
        testWithParam(param){
            return "your param is "+param
        }
    });
    app.all('/data', function(req, resp){
        var query = JSON.parse(req.query.query),
        promise = liteql.call(query);
        promise.then((result)=>{
            resp.send(result)
        })
    })
```
Client:
```javascript
    const liteql = new (require('liteql'))();
    liteql.addResources({
        __delegate__(query){
            return new Promise((resolver)=>{
                var xmlHttpRequest = new XMLHttpRequest();
                xmlHttpRequest.addEventListener('load', ()=>{
                    resolver(JSON.parse(xmlHttpRequest.responseText));
                });
                xmlHttpRequest.open('GET', '/data?query='+JSON.stringify(query));
                xmlHttpRequest.send();
            });	
        }
    });
    
    liteql.call(["test", {"testWithParam" : ["myParam"]}]).then((result)=>{
        console.log(result); //{test: "Test resource", testWithParam : "your param is myParam"}
    });
```

## LiteQL query syntax
### Overview
LiteQL query can be defined as an array or an object. It is required to define it as an array in case if resources depend on each other and order of resolving them matters, otherwise it ok to use objets. Each element of this array/object represents a resource. If we don't want to pass any parameters to the resource handler we define it as a string.
```javascript
    liteql.call(["resourceName"]);
```
When we want to pass any parameters to resource handlers, we define an object with one property where the key is the resource name and value is an array of parameters.
```javascript
    liteql.call({"resourceName": ["paramter1", "parameter2"]});
```
Parameters that we pass to a resource handler do not have to be strings or even primitives.
```javascript
    liteql.call({"resourceName" : ["string", 1, true, {"key": "value"}, ["a", "r", "r", "a", "y"]]});
```
### Using results of previous resource handlers as parameters for further ones
If we want a resource handler to use the result of other one, we can do that using "\_" at the beginning of parameter key.
Assume we defined two resource handlers:
```javascript
    liteql.addResources({
        first(){
            return {"key" : "val"}
        },
        second(param){
            var obj = {"val" : "value"}
            return obj[param]
        }
    });
```
 Now we can call:
 ```javascript
    liteql.call(["first", {"second" : ["_first.key"]}]).then((result)=>{
        console.log(result); //{first : {key : "val"}, second : "value"}
    }); 
```
### Dominant and buffer marker
 
 If we want to call a resource only to pass it as a parameter to another resource we can use '?' marker at the beginning of the resource name.
 ```javascript
    liteql.call(["?first", {"second" : ["_first.key"]}]).then((result)=>{
        console.log(result); //{second : "value"}
    });
```
If we are interested only in one resource, we don't need to get its key. So we can use "!"  marker to get rid of it and have only the resource handler result being returned.
```javascript 
     liteql.call(["?first", {"!second" : ["_first.key"]}]).then((result)=>{
        console.log(result); //"value"
    });
```
### Force delegating
If we want a method to be delegated regardless if it is available on the current instance, we define "\~" in the prefix. It make sense to do so for the "aggr" built-in method to decrease response size.
 ```javascript
    var inst1 = new LiteQL(),
        inst2 = new LiteQL();

    inst2.addResources({
        __delegate__(query){
            return inst1.call(query)
        },
        foo(){
            return "inst2"
        }
    }); 
    
    inst1.addResources({
        foo(){
            return "inst1"
        }
    });
    inst1.call(['foo']).then((result)=>{
        console.log(result); //inst1
    });
    
    inst1.call(['~foo']).then((result)=>{
        console.log(result); //inst2
    })
```

### Changing resource name
 If we want the result of a resource handler to be assign to a different key we can do that using "resourceName>newName" syntax
 ```javascript
    liteql.call(["?first>n1", {"second>n2" : ["_n1.key"]}]).then((result)=>{
        console.log(result); //{"n2" : "value"}
    });
```

## Adding resource handlers
There are to ways to add resources in LiteQL.
First using addResources() method, and second using "setResourceMethod".
Assume we store our resources in "resources" folder and we want to get them using format ```<fileName>.<methodToCall>``` so we can just write:

 ```javascript
 liteql.setResourceMethod((key)=>{
    key = key.split(.);
    return require('./resources/'+key[0])[key[1]];
 });
 ```
## Built-in methods
built-in methods help use with post-processing results of our queries. For now two methods are available. To call a built-in method we define '@' before its name.
### cache(\[\<Resource Name\>, \<Time in hours\>, \<is single served\>], ...) \: Cached keys
Cache method is used when we want to remember the resoult of some resource:
 ```javascript
    liteql.call(["first>n1", {"second>n2" : ["_n1.key"]}, 
        {
            "?@cache" : [["n1", 2, false], ["n2", 3, true]]
        }
    ]).then((result)=>{
        console.log(result); //The results of "first" result will be cached for 2 hours. The result of "second" resource will be cached for 3 hours. Also resource "second" will be removed after first time we get it from cache.
    });
 ```
### aggr("\<object to return\>") \: Object
Agregate method is used to change the "view" of the result
 ```javascript
 liteql.call(["first", {"second" : ["_n1.key"]}, 
    {"!@aggr" : {
        "obj" : {
            "n1" : "_first",
            "n2" : "_second"
        },
        "arr" : ["_first", "_second"]
    }
}]).then((result)=>{
    console.log(result); //{obj : {n1 : {key : "val"}, n2 : "value"}, arr : [{key : "val"}, "value"]}
});
```
### map(array, method) \: Array
Applies "method" to each item in the "array" parameter.
```javascript
    liteql.addResources({
        array(){
            return [{a : 1}, {a : 2}, {a : 3}] 
        },
        incrementA(item){
           item.a++	
       	   return item
        }
    });
    liteql.call(['array', 
        {'@map>mapped' : ['_array', 'each']}, 
        {'@!aggr' : '_mapped'}
    ]).then((r)=>{
         console.log(r); //'[{"a":2},{"a":3},{"a":4}]';
    });
```
### sort(array, key, desc)
Sorts "array" by "key". By default order is ascending if "desc" is true, the order is descending
```javascript
    liteql.addResources({
        array(){
            return [{a : 1}, {a : 2}, {a : 3}] 
        },
    });
    iteql.call(['array', {'!@sort' : ['_array', 'a', true]}]).then((r)=>{
        return console.log(r); //'[{"a":3},{"a":2},{"a":1}]';
    });
```
### frame(array, offset, limit)
Removes from "array" "offset" items from the begging and leaves only "limit" items in the array
```javascript
    liteql.addResources({
        array(){
            return [{a : 1}, {a : 2}, {a : 3}] 
        },
    });
    iteql.call(['array', {'!@frame' : ['_array', 1, 1]}]).then((r)=>{
        return console.log(r); //'[{"a":2}]';
    });
```
## Constructor parameter
 LiteQL constructor takes one parameter - "options"
  ```javascript
const liteql = new (require('liteql'))({
    resources : <object of resources we want to define on initialization>
    resourceMethod : <Function which returns resources based on query>
    delegatedBuiltin : <Array of strings, represents build in methods we want to delegate>
});
  ```
  
