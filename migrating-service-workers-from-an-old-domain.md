# Migrating your service workers from an old domain to your new domain

Recently we ran into the following problem, we had our website on https://www.hackages.io then we decided to migrate it to https://hackages.io, sounds easy right? Just write a redirect from the old domain to the new one?

So that's the first thing we did, we used NGINX to redirect all the traffic from https://www.hackages.io to https://hackages.io
```conf
server {
    listen       80;
    server_name  localhost;

    location / {
        return 301 https://hackages.io$request_uri;
    }
}
```

The thing we did not think about was that service workers don't play nice with redirects.

The issue was the following:

The user would go on https://www.hackages.io, nginx would send them a 301 redirect for every ressources they'd send a request for.

This worked perfectly fine for ***new*** users but users who had already visited the website would encounter some issues.

Service-workers update themselves automatically if there's a new version available, in our case it tried to get the new version on https://www.hackages.io/service-worker.js which redirects to https://hackages.io/service-worker.js.
So the service-worker would try to update itself from a new service worker *behind* a redirect which caused the following error:

![Service-worker redirect error](http://i.imgur.com/QqhI8Oo.png)

Since the service worker could not get the ressources behind the redirect, users would just see the old version of the website served from the old service worker.

## Reproducing the issue locally to tackle it down

Let's build two NGINX docker image both using this conf
(Associated dockerfiles can be found [here](https://github.com/0xClpz/migrating-service-workers-from-an-old-domain))
```conf
server {
    listen       80;
    server_name  localhost;
    root /usr/share/nginx/html;

    location = /service-worker.js {
        add_header Cache-Control no-cache;
        add_header Cache-Control no-store;
        add_header Max-Age 0;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
and using the following service-worker
```JavaScript
const ressources = [
  '/',
  '/index.html',
  '/style.css'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('sw-demo').then((cache) =>
      cache.addAll(ressources)
    )
  );
 });

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) =>
      response || fetch(event.request)
    )
  );
});
```

We'll run one of them on localhost:1500 (the old website, we'll call it _blue_) and localhost:2000 (the new website, let's call it _red_).

![website-a and website-b](http://i.imgur.com/G7MoSCX.png)

Now, let's kill _blue_ and run this nginx conf instead to redirect all the trafic from _blue_ to _red_
```conf
server {
    listen       80;
    server_name  localhost;

    location / {
        return 301 http://localhost:2000$request_uri;
    }
}
```

New users will get the new version, no problem, but old users will still encounter the issue mentionned above, the service-worker will try to get the new service worker behind a redirect and that doesn't work.

So now all our old users are stuck with on blue and are not being redirected to red because the service worker hijacks it.

## Destroying the old service-worker
The strategy to fix the problem will be the following:
Craft a service worker that is going to delete the previous one and serve that service-worker in the nginx that handles the redirect (_blue_).

We'll use this nginx conf to redirect everything but still serve our specially crafted service-worker:
```nginx
server {
    listen       80;
    server_name  localhost;

    location = /service-worker.js {
        root /usr/share/nginx/html;
        add_header Cache-Control no-cache;
        add_header Cache-Control no-store;
        add_header Max-Age 0;
    }

    location / {
        return 301 http://localhost:2000$request_uri;
    }
}
```

The first version we had was the following:
```JavaScript
self.addEventListener('install', () => self.skipWaiting());

self.addEventListener('activate', () => {
  self.registration.unregister();
});
```
Let's break it down:
```JavaScript
self.addEventListener('install', () => self.skipWaiting());
```
_self.skipWaiting()_ forces the waiting service worker to become the active service worker so even, if an user has multiple tabs open it kicks in instantly.
(If theres multiple tab open for a same website only one service worker runs for the three of them).

```JavaScript
self.addEventListener('activate', () => {
  self.registration.unregister();
});
```
The unregister method of the registration will delete any service worker registered for the host:port combo the service worker is registered on.

This pattern is going to kill any service worker that exists and old users will be redirected to _red_ on their second visit on _blue_.

## Improving our solution
This method works .. fine, but let's build a better version that will force the user to navigate to the new domain because in our case we did not want the users to still see the _blue_ website so we had to find a way to reload their browser after unregistering the service-worker.

In a service worker you can't simply do:
```JavaScript
window.location.replace('whatever.com');
```
because you don't have access to *a lot* of things in a service-worker, including window. 

First let's grab the list of clients using:
```JavaScript
self.clients.matchAll({type: 'window'});
```
This returns us a promise containing the list of clients of type window (tabs).

Each client will expose a navigate method that allows us to redirect the client to another page.

```JavaScript
self.clients.matchAll({type: 'window'})
  .then(clients => {
    for(const client of clients){
      client.navigate(client.url);
    }
  });
```
Here we make the client navigate to itself to reload the page.

## Putting it all together:
```JavaScript
self.addEventListener('install', () => self.skipWaiting());

self.addEventListener('activate', () => {
  self.registration.unregister();
  self.clients.matchAll({ type: 'window' }).then(clients => {
    for (const client of clients) {
      client.navigate(client.url);
    }
  });
});
```
To recap:
- It's going to activate instantly the service-worker
- It's going to tell the service-worker to unregister itself
- It's going to refresh each tab of the user
- NGINX will send a 301 redirect and the user'll navigate from _blue_ to _red_


## In action:

![Service-worker kill switch](https://i.imgur.com/cNLHYAr.gif)

