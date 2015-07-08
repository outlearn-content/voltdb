<!--
{
"name" : "creating",
"version" : "0.1",
"title" : "Creating the Database",
"description": "In VoltDB you define your database schema using SQL data definition language (DDL) statements just like other SQL databases.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

### Getting started

### Overview

After working with [Backbone](http://backbonejs.org/), [Meteor](http://meteor.com/), [AngularJS](http://angularjs.org/) and [Ember](http://emberjs.com/) (however, I have not dived deep into Ember yet), I feel that AngularJS is prepared the best for [Test Driven Development](http://en.wikipedia.org/wiki/Test-driven_development) (TDD). It truly makes it a cinch removing all excuses to not use tests in your application!

I have had the pleasure of focusing on testing within an [AngularJS](http://angularjs.org/) client application recently and lost a small amount of time testing a [directive](http://docs.angularjs.org/guide/directive). The issue revolved around the template being located as an external html file as opposed to being included within the directive itself. There were a couple of head scratching errors that [@amscotti](https://twitter.com/amscotti) and I had whilst seeking a solution but here’s a suggested approach.



### Tools

We will start from scratch, but it won’t take long to get up and running if we use [Yeoman](http://yeoman.io/). In fact there are a few libs I can recommend using:

*   [Grunt](http://gruntjs.com/)
*   [Bower](http://bower.io/)
*   [Karma](http://karma-runner.github.io/)

`sudo npm install -g yo grunt-cli bower karma`

Two yeoman generators you may find useful are:

*   [generator-angular](https://github.com/yeoman/generator-angular)
*   [generator-karma](https://github.com/yeoman/generator-karma)

`sudo npm install -g generator-angular generator-karma`

(thanks to [@amscotti](https://twitter.com/amscotti) for the heads up on these).

An alternative to the manual installation routine above is to use the awesome [Boxen](http://boxen.github.com/), read more on this [here](http://newtriks.com/2013/04/16/setting-up-node-dot-js-on-boxen/).

<!-- @task, "text" : "Install the tools."-->

### Create the project

```sh
mkdir directive-example && cd $_
yo angular
```

<!-- @task, "text" : "Create the project."-->

Here is a further list of [AngularJS generators](https://github.com/yeoman/generator-angular#generators)



<!-- @section -->

### Configure Karma

Open the [karma.conf.js](https://github.com/newtriks/angularjs-directives-testing-project/blob/master/karma.conf.js) file and make the following changes (any changes to this file require you to restart Karma):

**Base path**

To enable Karma to use the correct template path _and_ have the directive load template file you will need to change the _basePath_:

`basePath = 'app';`

**File patterns**

Amend the paths to reflect the newly defined _basePath_:

```javascript
files = [
  JASMINE,
  JASMINE_ADAPTER,
  'components/angular/angular.js',
  'components/angular-mocks/angular-mocks.js',
  'scripts/*.js',
  'scripts/**/*.js',
  'views/**/*.html',
  '../test/mock/**/*.js',
  '../test/spec/**/*.js'
];
```

**Compiling templates**

Templates need to be compiled to javascript otherwise you will get a parse error on running Karma. The [html2js preprocessor](https://github.com/karma-runner/karma-ng-html2js-preprocessor) is the solution and simply requires adding the following to your [karma.conf.js](https://github.com/newtriks/angularjs-directives-testing-project/blob/master/karma.conf.js) (this is based on you storing template files within a directory in _views_):

```javascript
preprocessors = {
  'views/**/*.html': 'html2js'
};
```

**Auto watch**

This is a really cool part of Karma where you can enable watching files and then auto executing tests as you develop:

`autoWatch = true;`

<!-- @task, "text" : "Go through all the configuration."-->

**Optional**

I use [PhantomJS](http://phantomjs.org/) to run the Angular tests as opposed to relying on a browser such as Chrome.

Change the browser for running tests to PhantomJS:

`browsers = ['PhantomJS'];`

<!-- @section -->

### Create your own example

### Create a directive

```javascript
yo angular:directive albums
```



### Start Karma

To start Karma which will also auto run the tests when we update the files use:

`karma start`

You should now see that two tests have been executed successfully:

`Executed 2 of 2 SUCCESS (0.284 secs / 0.015 secs)`



### Create a failing test


```javascript
'use strict';

describe('Directive: albums', function() {
  beforeEach(module('directiveExampleApp'));

    var element, scope;

    beforeEach(module('views/templates/albums.html'));

    beforeEach(inject(function($rootScope, $compile) {
        element = angular.element('<div class="well span6">' +
            '<h3>Busdriver Albums:</h3>' +
            '<albums ng-repeat="album in albums" title="{{album.title}}">' +
            '</albums></div>');

        scope = $rootScope;

        scope.albums = [{
            'title': 'Memoirs of the Elephant Man'
        }, {
            'title': 'Temporary Forever'
        }, {
            'title': 'Cosmic Cleavage'
        }, {
            'title': 'Fear of a Black Tangent'
        }, {
            'title': 'RoadKillOvercoat'
        }, {
            'title': 'Jhelli Beam'
        }, {
            'title': 'Beaus$Eros'
        }];

        $compile(element)(scope);
        scope.$digest();
    }));

    it("should have the correct amount of albums in the list", function() {
        var list = element.find('li');
        expect(list.length).toBe(7);
    });
});
```



(Our list is sourced from the crazy talented [Busdriver](http://en.wikipedia.org/wiki/Busdriver#Discography)!)

Let’s fix the first error we see:

`Error: No module: views/templates/albums.html`

<!-- @section -->

### Build a simple template

```sh
mkdir -p app/views/templates
touch app/views/templates/albums.html

```

Our test is still failing, let’s add the new template to the [albums.js](https://github.com/newtriks/angularjs-directives-testing-project/blob/master/app/scripts/directives/albums.js#L6) directive, change the line of code declaring the template to:

`templateUrl: 'views/templates/albums.html',`

The test will still fail as we are not creating a list within the template. Let’s do that now by adding the following to the [albums.html](https://github.com/newtriks/angularjs-directives-testing-project/blob/master/app/views/templates/albums.html):

`<ul><li></li></ul>`

And update the [albums.js](https://github.com/newtriks/angularjs-directives-testing-project/blob/master/app/scripts/directives/albums.js) directive as follows:


```javascript
'use strict';

angular.module('directiveExampleApp')
  .directive('albums', function() {
    return {
        templateUrl: 'views/templates/albums.html',
        restrict: 'E',
        scope: {}
    };
});
```


Word! The test passes. Let’s add another test to check the first album in the list has the correct title.

```javascript
it("should display the correct album title for the first item in the albums list", function() {
    var list = element.find('li');
    expect(list.eq(0).text()).toBe('Memoirs of the Elephant Man');
});
```

The test fails. Let’s add the title attribute to the directives scope in [albums.js](https://github.com/newtriks/angularjs-directives-testing-project/blob/master/app/scripts/directives/albums.js):



```javascript
'use strict';

angular.module('directiveExampleApp')
  .directive('albums', function() {
    return {
        templateUrl: 'views/templates/albums.html',
        restrict: 'E',
        scope: {
            title: '@title'
        }
    };
});
```


Sweet, the test’s are passing again.

All the source to this project can be found on [Github](https://github.com/newtriks/angularjs-directives-testing-project).


<!-- @section -->

### Potential errors and solutions

**[Spawn error solution](https://github.com/karma-runner/karma/issues/452)**

Set an env variable to your _~/.profile_ or _~/.bash_profile_.

```sh
which phantomjs
export PHANTOMJS_BIN='YOUR_PATH_TO_PHANTOMJS'
~/.profile to reload
```

**[Missing mocks error solution](https://github.com/yeoman/generator-angular#testing)**

`bower install angular-mocks`

<!-- @section -->

### Useful links

*   Vojta’s (godfather of Angular TDD) [ng-directive-testing](https://github.com/vojtajina/ng-directive-testing) example.
*   Vojta’s [presentation](http://www.youtube.com/watch?v=rB5b67Cg6bc) on testing directives.
*   AngularJS Developers [guide](http://docs.angularjs.org/guide/index)
