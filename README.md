# Single Page Application SEO

Well, is about the approach, we have the classic app where the page are prerender in the server and shipped to client, with all meta tags already defined, and on the other hand we have the single page application (SPA), where we shipped to clients a static page, index.html that not change, or change in the client later, that in fact, seen to be not good for the crawlers bots, because, they do not wait for all the page finish loading.
   
In response to this problem, we have a lot of the crawlers paid services, that render all the page (that mean, wait for all JavaScript Process and DOM changes), and save as a snapshot, that later is shipped in demand to crawlers bots (Googlebot, Facebot, etc). 

Well how to avoid all these crawlers paid services and similar hacks based on PhantomJS, well as I said is about the approach, what if we merge the both approach, of the classic and the SPA, to have a hybrid approach, like seems to be used for Youtube, is like an Ajax app or SPA and the same time is classic app where the page is rendered in the server, when you request some video, you get a processed page, all meta tags has the video related data, but when you change to other video, the meta tags are not updated. 

As code example, we set some Yeoman Angularjs Express Firebase App, and then the first to do is activate the HTML5 URL in the angular route config:    

```javascript
  .config(['$routeProvider', '$locationProvider', function($routeProvider, $locationProvider) {

    $locationProvider.html5Mode(true).hashPrefix('!');

    $routeProvider
      .when('/', {
        templateUrl: 'static/assets/views/main.html',
        controller: 'MainController'
      })
      .when('/view-publication/:publicationId/:title', {
        templateUrl: 'static/assets/views/viewPublication.html',
        controller: 'ViewPublicationController'
      })
      .otherwise({redirectTo: '/'});
  }])
```
> ###### After analyzing the code visit http://londres.herokuapp.com/view-publication/-KBkgfwLwzyyBMjQGeLR/subaru-impreza-wrx-sti.html to see this example running.   
 
Then the Express Route config:
  
```javascript  
  var path = require('path');
  var fs = require('fs');
  var Q = require('q');
  var handlebars = require('handlebars');
  var Firebase = require("firebase");
  
  var basePath = '';
  
  switch(process.env.NODE_ENV) {
    case 'production':
      basePath = '/dist';
      break;
    case 'development':
      basePath = '/public';
      break;
  }
  
  //process.cwd() : /home/romelgomez/workspace/projects/berlin
  //__dirname : /home/romelgomez/workspace/projects/berlin/server
  
  var metaTags = {
    title:        'MarketOfLondon.UK - Jobs, Real Estate, Transport, Services, Marketplace related Publications',
    url:          'https://londres.herokuapp.com',
    description:  'Jobs, Real Estate, Transport, Services, Marketplace related Publications',
    image:        'https://londres.herokuapp.com/static/assets/images/uk.jpg'
  };
  
  function readFile(fileName){
    var deferred = Q.defer();
  
    fs.readFile( path.join(process.cwd(), basePath, fileName), function (error, source) {
      if (error) {
        deferred.reject(error);
      }else{
        deferred.resolve({source: source});
      }
    });
  
    return deferred.promise;
  }
  
  function getPublication (uuid){
    var deferred = Q.defer();
  
    var publicationRef = new Firebase('berlin.firebaseio.com/publications/'+uuid);
    publicationRef.once('value', function (dataSnapshot) {
      deferred.resolve({publication: dataSnapshot.val()});
    }, function (error) {
      deferred.reject(error);
    });
  
    return deferred.promise;
  }
  
  function slug(input) {
    return (!!input) ? String(input).toLowerCase().replace(/[^a-zá-źA-ZÁ-Ź0-9]/g, ' ').trim().replace(/\s{2,}/g, ' ').replace(/\s+/g, '-') : '';
  }
  
  function capitalize(input) {
    return (!!input) ? input.replace(/([^\W_]+[^\s-]*) */g, function(txt){return txt.charAt(0).toUpperCase() + txt.substr(1).toLowerCase();}) : '';
  }
  
  function setMetaTitle (publication){
    metaTags.title = capitalize(publication.title).trim();
    metaTags.title += publication.department === 'Real Estate'? ' for ' + (publication.res.reHomeFor | uppercase) : '';
    metaTags.title += ' - MarketOfLondon.UK';
  }
  
  function setMetaURL (publicationID, publication){
    metaTags.url = 'https://londres.herokuapp.com/view-publication/';
    metaTags.url += publicationID + '/';
    metaTags.url += slug(publication.title) + '.html';
  }
  
  function setMetaImage (publication){
    var images = [];
  
    for (var imageID in publication.images) {
      if( publication.images.hasOwnProperty( imageID ) ) {
        publication.images[imageID].$id = imageID;
        if(imageID !== publication.featuredImageId){
          images.push(publication.images[imageID]);
        }else{
          images.unshift(publication.images[imageID])
        }
      }
    }
  
    metaTags.image = images.length > 0? ('http://res.cloudinary.com/berlin/image/upload/c_fill,h_630,w_1200/'+ images[0].$id +'.jpg') : 'https://londres.herokuapp.com/static/assets/images/uk.svg';
  }
  module.exports = function(app) {
  
    app.get('/', function(req, res) {
  
      readFile('index1.html')
        .then(function(the){
          var template = handlebars.compile(the.source.toString());
          return Q.when({result: template(metaTags)})
        })
        .then(function(the){
          res.send(the.result);
        },function(error){
          throw error;
        });
  
    });
  
    app.get('/view-publication/:id/:title', function(req, res){
  
      getPublication(req.params.id)
        .then(function(the){
  
          setMetaTitle(the.publication);
          setMetaURL(req.params.id, the.publication);
          metaTags.description = the.publication.description;
          setMetaImage(the.publication);
  
          return readFile('index1.html');
        })
        .then(function(the){
          var template = handlebars.compile(the.source.toString());
          return Q.when({result: template(metaTags)})
        })
        .then(function(the){
          res.send(the.result);
        },function(error){
          throw error;
        });
  
    });  
  
    app.get('*', function(req, res){
  
      readFile('index1.html')
        .then(function(the){
          var template = handlebars.compile(the.source.toString());
          return Q.when({result: template(metaTags)})
        })
        .then(function(the){
          res.send(the.result);
        },function(error){
          throw error;
        });
 
    }); 
  
  };
```

Then in the index.html

```
<!doctype html>
<html class="no-js" ng-app="app">
<head>
  <meta charset="utf-8">
  <base href="/">
  <meta name="fragment" content="!" />
  <meta name="viewport" content="width=device-width">

  <title>{{title}}</title>

  <meta property="og:title" content="{{title}}" />
  <meta property="og:url" content="{{url}}" />
  <meta property="og:description" content="{{description}}" />
  <meta property="og:image" content="{{image}}" />
  <meta property="og:type" content="website" />
  <meta property="og:site_name" content="MarketOfLondon.UK" />

  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:site" content="@romelgomez07" />

  <meta name="twitter:title" content="{{title}}" />
  <meta name="twitter:url" content="{{url}}" />
  <meta name="twitter:description" content="{{description}}">
  <meta name="twitter:image:src" content="{{image}}" />
  
  ...
```

And that's all.

---
Prof. Romel Gomez. 
@romelgomez07
