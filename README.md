# NgxLazyPlugins

LazyLoading + AOT + Recompile modules = Super Lazy Reusable Plugins

## How it works

### angular.json hacks

Firstly, you need to configure the app build

angular.json
```js
// you should turn off all optimizations so that you do not break the plugins
projects.APP.architect.build.configurations.production = {
    ...,
    "outputHashing": "none", // for easy management of plugins 
    "namedChunks": true, // for easy management of plugins 
    "optimization": false, // disabled tree shaking
    "commonChunk": false, // disabled chunk of chunks :)
    "preserveSymlinks": true, // for using dependencies in plugins
    "vendorChunk": false,
    "buildOptimizer": true
}

// you should add each plug-in to this list
projects.APP.architect.build.options.lazyModules = [
    "./src/plugins/example/example.module.ts"
]
```

For example, in our angular.json there are two configurations of the app.
`ng-lazy-plugins` without plug-ins, and `ng-lazy-plugins-example` with plug-ins.
```json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "ng-lazy-plugins": { ... },
    "ng-lazy-plugins-example": { ... }
  }
}
```

### Example of plugin

Next, you need to create a plugin
```typescript
// Use token to provide the main application with the component
import { PLUGIN_PROVIDER } from 'src/core';

@Component({
  selector: 'example-plugin',
  template: `
    <h1>Hello World! I'm plugin! And i recompiled!</h1>
    <a [routerLink]="'/lazy/hello'">Go to link</a>
  `
})
export class ExamplePluginComponent {}

@NgModule({
  imports: [
    RouterModule.forChild([
      {
        path: 'hello',
        component: ExamplePluginComponent
      }
    ])
  ],
  declarations: [ ExamplePluginComponent ],
  providers: [
    {
      provide: PLUGIN_PROVIDER,
      useValue: ExamplePluginComponent
    }
  ]
})
export class ExamplePluginModule {}
```

### Using DynamicNgModuleFactoryLoader to download plugins

Use DynamicNgModuleFactoryLoader to load the plugin
```typescript
import { DynamicNgModuleFactoryLoader, PLUGIN_PROVIDER, PluginManifest } from 'src/core';

@Component({...})
export class AppComponent {
  constructor(
    private injector: Injector,
    private ngModuleFactoryLoader: DynamicNgModuleFactoryLoader
  ) {
    // you should specify a manifest to load the module
    const manifest: PluginManifest = new PluginManifest({
      id: './src/plugins/example/example.module',
      name: 'ExamplePluginModule',
      path: 'src-plugins-example-example-module-ts-ngfactory'
    });

    // loading a lazy module, its your plugin
    this.ngModuleFactoryLoader.load(manifest.importUrl)
        .then((ngModuleFactory: NgModuleFactory<any>) => {
          this.ngModuleFactory = ngModuleFactory;
  
          // Create ng module reference
          const ngModuleRef: NgModuleRef<any> = ngModuleFactory.create(this.injector);
  
          // We can resolve by any provider from lazy loaded module
          this.component = ngModuleRef.injector.get(PLUGIN_PROVIDER);
  
          // And we can attach routes from lazy loaded module
          this.router.resetConfig([
            ...this.router.config,
            { path: 'lazy', loadChildren: () => ngModuleFactory }
          ]);
        });
  }

}
```

All of this, you can safely re-build plugins via `npm build ng-lazy-plugins-example --prod`.
You will get the compiled plugins in the directory with the build:
 + src-plugins-example-example-module-ts-ngfactory.js
 + src-plugins-example2-example2-module-ts-ngfactory.js

Just put them in the folder with your the app.
