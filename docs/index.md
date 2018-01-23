# Proxy Databases, "DRYer" HTTP requests
Wouldn't it be cool if you didn't have to write any more requests to your server? Imagine if you could interact with your database just like you interact with any other object in javascript, and now imagine if you could do that without any javascript at all.

First let's talk about why. The majority of web applications are skins around databases. You have some data that you want to show the user and maybe you want them to interact with that data.

In order to make your lovely database skin you end up writing a lot of boiler plate code which basically consists of calling endpoints on your server that might look something like this:

```
get_my_data("/api/get/my_cool_data").then((my_cool_data)=>{
	/* Got my data, time to show it to the user! */
}).catch((error)=>{
	/* Do something about your error */
})
```

```
modify_my_data("/api/update/my_cool_data",{"name":"Michael Green"}).then(function(response){
	/* Yay I changed some stuff in my database */
}).catch((error)=>{
	/* Oh no an error occured! */
})

```. 

Of course the actual code is much longer (this is with some awesome imaginary helper methods) but you get the idea.

### Why is this not so great?

```
1.) You end up writing a lot of boiler plate code, it's not that bad but it could be better.
2.) It's difficult to maintain consistent state across your entire application, especially if you're using something like web components and need to notify different components of changes to data with minimal effort
3.) It's difficult to maintain, when endpoints change you have to change them everywhere and you also have to keep track of what endpoints do what.
4.) So much JavaScript... I love JavaScript but if I can just write some HTML I'd like that better.
```

How can we fix all of these problems? We can use what we're calling "proxy databases" the basic idea is that you *trap* the basic operations of an object to fetch and modify your database automatically.

(Proxies were introduced in ES6, you can read in depth about them on MDN:
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

A trap is basically something that captures built in operations like getting and settings keys within an object or modifying the contents of an array. For example imagine if you had the following object:

```
let user = {
	"name":"Michael Green",
	"age":21,
	"preferences":{
		"email_notifications":false,
		"text_notifications":false
	}
}
```

The normal behavior of getting a value from a key is that you input a key and you get back the value.

```
	console.log(user["name"]); /* Prints out "Michael Green" */
```
However Proxy objects allow us to trap this default behavior and modify it.

```
sample_handler = {
	get: function(target,key,reciver){
		if(key == "name"){
			return "Scott Lindeneau"
		}
	}
}

proxy_user = new Proxy(user,sample_handler)
console.log(proxy_user["name"]); /* Prints Scott Lindeneau */
```

You can trap getting, setting, and deleting as well as a lot of other properties.

We can utilize these traps to create a reflection of our database within an object. The easiest way to imagine this is with a document store like MongoDB that stores things in BSON as documents already look like javascript objects (BSON == Binary JavaScript Object Notation). 

This allows us to achieve some rather useful behavior:

First by trapping our getter we can make it so that if we call `proxyDb["my_key"]` it will first attempt to return the value if it is currently stored within our proxyDb object however if it is not present we can perform an ajax call to get and store the data within the object.

That might look something like this:

```
let handler = {
	get: function(target,key,reciver){
		if(key == "name"){
			if(key in target){
				return target[key]
			} else{
				/* Request data from our backend, this is asynchronous so we mark the value as pending and then overwrite it as soon as the request from our server resolves */
				target[key] = get_my_data("/api/get/my_cool_data_endpoint");
				return target[key]
			}
		}
	}
}

/* The initial state of our proxy database is empty as we have not attempted
to get any keys from our object yet. */
let proxyDb = new Proxy({},handler);
console.log(proxyDb["name"]); /* Returns a pending promise */
console.log(proxyDb["name"]); /* Returns the resolved name from our database */
```

The next thing you can do is trap the assignment operator to reflect changes you make to the object back up to your database. We do this by using the set property within our proxy's handler.

```
	let handler = {
		get: function(target,key,reciver){
			if(key == "name"){
				if(key in target){
					return target[key]
				} else{
					/* Request data from our backend, this is asynchronous so we mark the value as pending and then overwrite it as soon as the request from our server resolves */
					target[key] = get_my_data("/api/get/my_cool_data_endpoint");
					return target[key]
				}
			}
		}

		set: function(target,key,value){
			if(key == "name"){
				/* Send up your data to the db... There's a lot left to be desired here, you'll have to make sure that the data gets sent up successfully, so you might only want to set the key if it's successful. */
				modify_my_data("/api/update/my_cool_data",{"name":value});
				target[key] = value;
			}
			return true;
		}
	}

	/* This time let's assume we have a name within our database and that's already on the server */
	let proxyDb = new Proxy({
		"name":"Michael Green"
	},handler);
	proxyDb["name"] = "Scott Lindeneau"; /* Automatically updates the document on the server */
```

So that's pretty cool, and you can set up your backend endpoints such that you don't have to necessarily check what type of key you're modifying every time which results in even less boilerplate code. However we can take this a step further by combining our proxy database with your favorite frontend framework's variable binding. 

At Rosey, we use Polymer 2, a frontend framework that augments native web components for easier usability. One of the difficult parts about Polymer is maintaining state between components, when you modify data within one component you want to notify other components that also use that data. What you end up with is **a lot** of duplicate properties and having to explicitly pass properties through a troff of nested components when really maybe only two or three of those components have anything to do with the data you're modifying.

So how do we fix this? First let's think about our goals:

1.) We want to be able to refer to our data within our component's HTML so we can display it to the user. 
2.) We want the user to be able to modify this data and for those changes to be instantly reflected everywhere
3.) We want to do both of those things while writing the least amount of code as possible.

We can achieve the first goal simply by binding our proxy database to a polymer variable. However in order for changes to be reflected everywhere we have to modify our database such that whenever a property is updated we notify everyone that's using it without having to declare any properties or special syntax.

WORK IN PROGRESS NOT FINISHED.
