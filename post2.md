# Vue.js bundled by Gulp.js + Browserify


Today we have a variety of tools and frameworks to build up our stack and gain productivity within our projects. The goal of this post is not point which is better or not. My main thought about this subject is that we should always rather learn the language itself before diving in into a full featured framework. If you do that you will most likely enjoy good parts of every framework and detect situations where one is better suited than other.


Anyway I've been working with Vue.js lately and I'm enjoying a lot. It reunited good parts from Angular that I had a little experience at the time and also matches my personal style back when I was coding only VanillaJS. It also supports very well CommonJS, a pattern that I'm a fan.


I've worked with Gulp.js before starting with Vue and the combination of automating things in a node.js environment, working with Node.js on backend and exporting frontend modules with CommonJS`s require/module.exports (<3) was easy peazy. It is just enjoyable to me work with this Stack. So homogeneous.


Well everything was fine until I noticed soon enough that every tutorial, post, official docs was kinda promoting Webpack which is an awesome tool as well. But I was not willing give up Gulp and I will share some caveats both working with Vue.js apps and Vue.js exportable components in a Build tool perspective.


First of all we need to load up some dependencies, the list is quite long but you know... Plugins.

## Gulp plugins
`npm install gulp gulp-babel gulp-browserify gulp-cache gulp-cond gulp-envify gulp-eslint gul
p-plumber gulp-postcss gulp-rename gulp-sass gulp-sourcemaps gulp-uglify gulp-util gulp-watch --save-dev`


## Vue dependencies
`npm install vue vue-template-compiler eslint-plugin-vue vueify --save-dev`


## Other dependencies
` npm install fs-extra babel-core babel-plugin-transform-runtime babel-preset-env babel-eslint
 babel-runtime babelify browserify eslint envify cross-env dotenv --save-dev`
 
 
## Putting together
Now you need to create a gulpfile.js and set it up. For sake of brevity I will share a Github link to the example file instead of declaring it all here: [Gist]. Declaration are on top of the file.


## Problem #1 - Applying correct env
The first problem that I've faced was the right way to declare production/test/beta environment within Vue.js transforms and builds. E.g remove warnings and Vue browser plugin without explicitly setting Vue global properties to false.

```
//js task
gulp.task("js", function () {
   browserifyInstance = browserify({
      "entries": YOUR_ENTRY_POINT.js,
      "noParse": ["vue.js"],
      "plugin": argv.w || argv.watch ? [watchify] : [],
      "cache": {},
      "packageCache": {},
      "debug": !isProd
   }).transform("envify", {
      "global": true,
      "NODE_ENV": process.env.NODE_ENV
   })
      .transform(babelify)
      .transform(vueify)
      .on("update", methods.bundleJS);
   return methods.bundleJS();
});
```

```
//bundleJS
"bundleJS": function () {
   if (isProd) {
      fse.remove(path.join(modulePath, "dist/js/bundle.js.map"));
   }
   browserifyInstance.bundle()
      .on("error", function (err) {
         gutil.log(err);
      })
      .pipe(source(path.join(modulePath, "js/main.js")))
      .pipe(cond(!isProd, plumber()))
      .pipe(buffer())
      .pipe(rename("bundle.js"))
      .pipe(cond(isProd, uglify()))
      .pipe(cond(!isProd, sourcemaps.init({"loadMaps": true})))
      .pipe(cond(!isProd, sourcemaps.write("./")))
      .pipe(gulp.dest(path.join(modulePath, "dist/js/")));
   return isProd ? gutil.log("FINISHED PRODUCTION BUILD") : gutil.log("FINISHED DEV BUILD");
}
```

Here when we run gulp js it will look for the path(s) declared on entries property to resolve requires.


We are also telling to browserify do not parse vue.js library, and apply watchify plugin if suited (not covered on this topic but you just need to pass a -w flag).


Cache and packageCache are to handle the hot reload and patch build strategy.


Now we start applying the first env variable, Vue.js does not accept simpy running browserify+gulp passing a PROD/TEST env flag on command line, so we use envify transform applying as a global property and controlling NODE_ENV as a parameter from a command line argument or a enviroment variable file.


After envify apply its declaration we then apply [babelify](https://github.com/babel/babelify) and [vueify](https://github.com/vuejs/vueify). For specification of these tools please refer to their documentation.


To finish this routine we just minify it if it is production or generate sourceMaps if test env. It has full support of vue components including scss/stylus transformations.


Seems straightforward right? Well took me some time to figure this config out. Notice that you will still need the .babelrc and .eslintrc config files.



## Problem #2 Working with Vue.js runtime only build.

[Vue.js has two builds](https://vuejs.org/v2/guide/installation.html) and it can be confusing to make it work with the most lightweight version with this stack.

First of all we need to do a configuration on our `package.json` file and set the browser property as following (assuming that Vue.js installation is done through NPM):
```
//package.json
"browser": {
    "vue": "vue/dist/vue.runtime.common.js"
}
```

After declaring it on package.json you should set your main application point (declared as entries on js task) should import a root component following the <template><script><style> pattern

```$xslt
	"use strict";
	const Vue = require("vue");

	new Vue({
		"el": "#app",
		"name": "admin",
		"render": function (h) {
			return h(require("./components/app.vue"));
		}
	});
```

This will load your component on the anchored el property. The required files can contain another requires from both .js and .vue files and make a complex application.


## Problem #3 Working with exported .vue components

On this one I almost gave up and use webpack. The problem was that I was not able to generate a valid UMD/CommonJS file to be required and used as intended. But turns out it was just a misconfiguration of Browserify and a little tweak make it possible, so now I'm finishing my first component to be exported and distributed.

```$xslt
 gulp.task("js", function () {
        browserifyInstance = browserify({
            "entries": path.join(modulePath, "src/js/main.js"),
            "noParse": ["vue.js"],
            "read": false,
            "standalone": "JngVueChat",
            "plugin": argv.w || argv.watch ? [watchify] : [],
            "cache": {},
            "packageCache": {},
            "debug": !isProd
        }).transform("envify", {
            "global": true,
            "NODE_ENV": process.env.NODE_ENV
        })
            .transform(babelify)
            .transform(vueify)
            .on("update", methods.bundleJS);
        return methods.bundleJS();
    });
```


The only change here is that I add a 'standalone' property passing a module name and it is ready to be exported as UMD/AMD and Browser.


Hope it can help you guys save some time. Any feedback is appreciated.

Regards,
Daniel Abrao