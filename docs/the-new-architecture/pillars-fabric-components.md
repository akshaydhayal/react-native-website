---
id: pillars-fabric-components
title: Fabric Components
---

import Tabs from '@theme/Tabs'; import TabItem from '@theme/TabItem'; import constants from '@site/core/TabsConstants';

A Fabric Component is a UI component rendered on the screen using the [Fabric Renderer](https://reactnative.dev/architecture/fabric-renderer).

Using Fabric Components instead of Native Components allows us to reap all the [benefits](./why) of the **New Architecture**. Specifically, we are able to leverage JSI to efficiently connect the Native UI code JavaScript.

A Fabric Component is created starting from a **JavaScript specification**. This, with the help of [**CodeGen**](./pillars-codegen), will create some C++ code, integrated in the platform native layer and shared among all the React Native platforms. The C++ code is boilerplate code that the component-specific logic needs to use to be properly used by React Native. After the component-specific logic has been connected with the generated code, the component can be integrated in the app.

The following section will guide you through the creation of a Fabric Component, step-by-step.

:::caution
Fabric Components only works with the **New Architecture** enabled.
To migrate to the **New Architecture**, follow the [Migration guide](../new-architecture-intro)
:::

## How to Create a Fabric Components

To create a Fabric Component, we have to follow these steps:

- Define a set of JavaScript specifications.
- Configure the component so that it can be consumed by an app.
- Write the native code required to make it work.

Once these steps are done, the component is ready to be consumed by an app. Therefore, the guide shows how to add it to an app, leveraging _autolinking_, and how to reference it from the JavaScript code.

### Folder Setup

The easiest way to create a component is as a separate module we will then import as a dependency for our apps. This keeps the component decoupled from the app, and auto-linking can be used to manage it properly.

For this guide, we are going to create a Fabric Component that centers some text on the screen.

Let's create a new folder at the same level of our app and let's call it `RTNCenteredText`.

In this folder, we are going to create three subfolders: `js`, `ios` and `android`.

The final result should look like this:

<figure>
  <img width="500" alt="Folder Structure for a Fabric Component" src="/docs/assets/NewArchitecture/AppFolderStructure.png"/>
  <figcaption>Initial folder structure for a Fabric Component.</figcaption>
</figure>

### JavaScript Specification

The **New Architecture** requires to specify a single source of truth for your component interfaces, using a typed dialect of JavaScript (either [Flow](https://flow.org/) or [TypeScript](https://www.typescriptlang.org/)). We need a typed dialect because **Codegen** has to generate code in strongly-typed languages, including C++, Objective-C++ and Java.

Another important aspect of the JavaScript specification is the file name. A component filename must end with the `NativeComponent.js` (or `jsx`, `ts`, `tsx`) suffix. For example, if you want to create a `MyFabricComponent` component, the specification file must be named `MyFabricComponentNativeComponent.js` (or any other supported extension). The **Codegen** process will look for files whose name ends with `NativeComponent` to generate the required code.

The following are the specification of our `RTNCenteredText` component in both Flow and TypeScript: let's create a `RTNCenteredText` file with the proper extension in the `js` folder.

<Tabs groupId="fabric-component-specs" defaultValue={constants.defaultJavaScriptSpecLanguages} values={constants.javaScriptSpecLanguages}>
<TabItem label='flow'>

```typescript
// @flow strict-local

import type {ViewProps} from 'react-native/Libraries/Components/View/ViewPropTypes';
import type {HostComponent} from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

type NativeProps = $ReadOnly<{|
  ...ViewProps,
  text: ?string,
  // add other props here
|}>;

export default (codegenNativeComponent<NativeProps>(
   'RTNCenteredText',
): HostComponent<NativeProps>);
```

</TabItem>
<TabItem label='typescript'>

```typescript
import type { ViewProps } from 'ViewPropTypes';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

export interface NativeProps extends ViewProps {
  ...ViewProps,
  text: string | null | undefined,
  // add other props here
}

export default codegenNativeComponent<NativeProps>(
  'RTNCenteredText'
) as HostComponent<NativeProps>;
```

</TabItem>
</Tabs>

Let's break these down a little.

At the beginning of the spec files, there are the imports. Here, we can import what we need from React Native and other dependencies if needed. The most important imports, that are present in all the Fabric Components, are:

- The `HostComponent`, which is the type our exported component needs to conform to;
- The `codegenNativeComponent` function, which is responsible to actually register the component in the JS runtime.

The second section of the files contains the **props** of the component. [Props](https://reactnative.dev/docs/next/intro-react#props) (short for "properties") are component-specific information that let you customize React components. In this case, we want to control the `text` the component will render.

Finally, we invoke the `codegenNativeComponent` generic function, passing the name we want to use for our component. The returned value is then exported by the JavaScript file in order to be used by the app.

:::caution
We are writing JavaScript files importing types from libraries, without setting up a proper node module and installing its dependencies. The outcome of this is that your IDE may have troubles resolving the import statements and you may see errors and warnings.
These will disappear as soon as we add the the Fabric Component as a dependency of our React Native app.
:::

### Component Configuration

The second element we need to properly develop the Fabric Component is a bit of configuration, that will help you setting up:

- all the data the **CodeGen** process requires to run properly
- the files required to link the Fabric Component into the app

Some of these configuration are shared between iOS and Android, while the others are platform-specific.

#### Shared

The shared configuration is a `package.json` file that will be used by yarn when installing your component. Create the `package.json` file in the root of the `RTNCenteredText` directory.

```json
{
  "name": "rnt-centered-text",
  "version": "0.0.1",
  "description": "Showcase a Fabric Component with a centered text",
  "react-native": "js/index",
  "source": "js/index",
  "files": [
    "js",
    "android",
    "ios",
    "rnt-centered-text.podspec",
    "!android/build",
    "!ios/build",
    "!**/__tests__",
    "!**/__fixtures__",
    "!**/__mocks__"
  ],
  "keywords": ["react-native", "ios", "android"],
  "repository": "https://github.com/<your_github_handle>/rnt-centered-text",
  "author": "<Your Name> <your_email@your_provider.com> (https://github.com/<your_github_handle>)",
  "license": "MIT",
  "bugs": {
    "url": "https://github.com/<your_github_handle>/rnt-centered-text/issues"
  },
  "homepage": "https://github.com/<your_github_handle>/rnt-centered-text#readme",
  "devDependencies": {},
  "peerDependencies": {
    "react": "*",
    "react-native": "*"
  },
  "codegenConfig": {
    "libraries": [
      {
        "name": "RNTCenteredTextSpecs",
        "type": "components",
        "jsSrcsDir": "js"
      }
    ]
  }
}
```

Let's break it down.

The upper part of the file contains some descriptive information like the name of the component, its version and what composes it.
Make sure to update the various placeholders which are wrapped in `<>`, so replace all the occurrences of the `<your_github_handle>`, `<Your Name>`, and `<your_email@your_provider.com>` tokens.

Then we have the dependencies for this package, specifically we need `react` and `react-native`, but you can add all the other dependencies you may have.

Finally, the last important bit is the `codegenConfig` field. This configures the **CodeGen** process. It contains an array of libraries, each of which is defined by three fields:

- `name`: The name of the library we are going to generate. By convention, we add the `Specs` suffix.
- `type`: The type of module contained by this package. In this case, we have a component, thus the key to use is `components`.
- `jsSrcsDir`: the relative path to access the `js` specification that will be parsed by the CodeGen.

#### iOS: Create the `podspec` file

Now we configure the native component so that we can generate the code necessary for integration with our app. For further information on how the **CodeGen**, have a look [here](/docs/pillars-codegen.md).

For iOS, we need to create a `.podspec` file which will define the component as a dependency. It will stay in the root folder of the `RTNCenteredText` component, outside the `ios` folder.
The `.podspec` file for our component will look like this

```ruby title="rnt-centered-text.podspec"
require "json"

package = JSON.parse(File.read(File.join(__dir__, "package.json")))

folly_version = '2021.06.28.00-v2'
folly_compiler_flags = '-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1 -Wno-comma -Wno-shorten-64-to-32'

Pod::Spec.new do |s|
  s.name            = "rnt-centered-text"
  s.version         = package["version"]
  s.summary         = package["description"]
  s.description     = package["description"]
  s.homepage        = package["homepage"]
  s.license         = package["license"]
  s.platforms       = { :ios => "11.0" }
  s.author          = package["author"]
  s.source          = { :git => package["repository"], :tag => "#{s.version}" }

  s.source_files    = "ios/**/*.{h,m,mm,swift}"

  s.dependency "React-Core"

  s.compiler_flags = folly_compiler_flags + " -DRCT_NEW_ARCH_ENABLED=1"
  s.pod_target_xcconfig    = {
    "HEADER_SEARCH_PATHS" => "\"$(PODS_ROOT)/boost\"",
    "OTHER_CPLUSPLUSFLAGS" => "-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1",
    "CLANG_CXX_LANGUAGE_STANDARD" => "c++17"
  }

  s.dependency "React-RCTFabric"
  s.dependency "React-Codegen"
  s.dependency "RCT-Folly", folly_version
  s.dependency "RCTRequired"
  s.dependency "RCTTypeSafety"
  s.dependency "ReactCommon/turbomodule/core"
end
```

The `podspec` file has to be a sibling of the `package.json` file and its name is the one we set in the `package.json`'s `name` property: `rnt-centered-text`.

The first part of the file prepares some variables we will use throughout the rest of it. Then, there is a section that contains some information used to configure the pod, like its name, version, and description. Finally, we have a set of dependencies that are required by the new architecture.

#### Android: Create the `build.gradle` file

We need to create a `build.gradle` file in the `android` folder, with the following contents:

```kotlin title="build.gradle"
buildscript {
  ext.safeExtGet = {prop, fallback ->
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
  }
  repositories {
    google()
    gradlePluginPortal()
  }
  dependencies {
    classpath("com.android.tools.build:gradle:7.1.1")
  }
}

apply plugin: 'com.android.library'
apply plugin: 'com.facebook.react'

android {
  compileSdkVersion safeExtGet('compileSdkVersion', 31)

  defaultConfig {
    minSdkVersion safeExtGet('minSdkVersion', 21)
    targetSdkVersion safeExtGet('targetSdkVersion', 31)
    buildConfigField("boolean", "IS_NEW_ARCHITECTURE_ENABLED", "true")
  }
}

repositories {
  maven {
    // All of React Native (JS, Obj-C sources, Android binaries) is installed from npm
    url "$projectDir/../node_modules/react-native/android"
  }
  mavenCentral()
  google()
}

dependencies {
  implementation 'com.facebook.react:react-native:+'
}

react {
    jsRootDir = file("../js/")
    libraryName = "RTNCenteredText"
    codegenJavaPackageName = "com.RTNCenteredText"
}
```

The `gradle` contains various section to properly configure the library and to access the required dependencies. The most interesting bits are:

- the `defaultConfig` block within the `android` block, where we add a `buildConfigField` to enable the new architecture.
- the `react` block where we configure the CodeGen process. For Android, we need to specify:
  - The `jsRootDir` which contains the relative path to the JavaScript specs
  - The `libraryName` we will use to link the library to the app.
  - The `codegenJavaPackageName` which corresponds to the name of the Java Package we will use to group the code generated by the **CodeGen**.

### Native Code

#### iOS

#### Android

### Adding the Fabric Component To Your App