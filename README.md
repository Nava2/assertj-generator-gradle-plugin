# AssertJ Generator Gradle Plugin

[![Linux Build Status](https://travis-ci.org/Nava2/assertj-generator-gradle-plugin.svg?branch=master)](https://travis-ci.org/Nava2/assertj-generator-gradle-plugin)
[![Windows status](https://ci.appveyor.com/api/projects/status/ia2ki6wprow9ysnl/branch/master?svg=true)](https://ci.appveyor.com/project/Nava2/assertj-generator-gradle-plugin/branch/master)

[Gradle Plugin Portal](https://plugins.gradle.org/plugin/org.assertj.generator.gradle.plugin)

This is the source for the Gradle plugin for the 
[AssertJ Generator](http://joel-costigliola.github.io/assertj/assertj-assertions-generator.html). This plugin leverages
existing Gradle [SourceSet](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.SourceSet.html) API to make it 
easier to integrate the AssertJ Generator into existing Gradle-based projects. The configurations available mimic those 
provided by the [AssertJ Generator Maven Plugin](http://joel-costigliola.github.io/assertj/assertj-assertions-generator-maven-plugin.html).


## Quick Start

Below is a minimal configuration that will cause _all_ classes defined in the `main` source set to have an `AssertJ` 
assertion generated for it. All of these will be placed into the highest package in use.

```gradle
plugins { 
    // version matches the version for assertj-generator
    id 'org.assertj.generator.gradle.plugin' version '1.0.0' 
} 

sourceSets {
    main {
        // must specify assertJ block to have it applied
        assertJ { }
    }
}

// add some classpath dependencies
repositories {
    mavenCentral()
}
            
dependencies {
    // https://mvnrepository.com/artifact/org.assertj/assertj-core
    testCompile group: 'org.assertj', name: 'assertj-core', version: '3.8.0'
    
    testCompile group: 'junit', name: 'junit', version: '4.12'
}
```

## Configuration

Primary configuration of included/excluded files is done via the same mechanisms that are used for Gradle's SourceSet 
DSL. For explanation on how filters work, please review the 
[Java Plugin Tutorial for Gradle](https://docs.gradle.org/current/userguide/java_plugin.html#sec:changing_java_project_layout).
This plugin utilizes the filters to gather classes to run against the generator. **This is different than how the AssertJ 
Generator normally functions** -- this plugin does not work on class/package names, but folder/file patterns like the 
Java plugin.

### Plugin Version

The plugin version is tied to the version of the [AssertJ Generator](http://joel-costigliola.github.io/assertj/assertj-assertions-generator.html).
The version of the generator used will always match the plugin's version.

### Scope: Local vs. Global 

This plugin allows the generation configuration to appear in two places:

* "global" `assertJ {}` block -- outside `sourceSets`
* Locally within a `sourceSet` in an `assertJ {}` block (e.g. within `main{}`)

All "locally" defined parameters override those set in the "global" scope. 

Each of these provide the same configuration parameters except:

* Globally, files and patterns can not be manipulated -- this must be done at the local scope

### Parameters

The following sections outline the available parameters, linking to the appropriate generator docs when available. 

For any unlisted parameters, please see the 
[javadoc for `AssertJGeneratorOptions`](./src/main/groovy/org/assertj/generator/gradle/tasks/AssertJGeneratorOptions.groovy)
and open issues for unclear/missing documentation. 

#### P: skip - `boolean`

Default: `true`

The `skip` parameter provides a mechanism for quickly turning on/off `assertJ` generation. Unless overridden in the 
global scope, no code will be generated by default. When an `assertJ` block is defined in a local scope, this parameter
is set to `false` for the source set.
 
##### Example

The following example turns on generation for _only_ `main` All classes found within `main` and `test` will have an 
Assertion generated. This is the recommended way to turn on generation for source sets.

```gradle
assertJ {
    // Add any configuration options to apply to all source sets here
}

sourceSets {
    main { 
        assertJ { } // sets skip = false for the "main" sourceSet
    }
    test { }
}
```

If debugging a build, it may be useful to turn off generation for a `sourceSet`. To do this, inside the configuration 
 block, set `skip = true`. 
 
```gradle
assertJ {
    // Add any configuration options to apply to all source sets here
}

sourceSets {
    main { 
        assertJ { } // sets skip = false for the "main" sourceSet
    }
    
    brokenGeneration {
        assertJ {
            skip = true // no assertions will be generated for 
                        // "brokenGeneration"
            // other parameters follow, order of assignment of different
            // parameters does not matter, like with gradle
        }
    }
    test { }
}
```

The following example turns on generation for `main` and `test` without adding configuration blocks. Any class found 
 within `main` and `test` will have an Assertion generated. This is **not recommended** as it can lead to excessive 
 generation and unnecessary slowdowns. 

```gradle
assertJ {
    // Add any configuration options to apply to all source sets here
    skip = false // default: true
}

sourceSets {
    main { }
    test { }
}
```

#### P: outputDir - `File`
 
Default: `${buildDir}generated-srcs/${sourceSet.getTaskName('test', '')}/java`

The root directory where all generated files will be placed (within sub-folders for packages).

Changing this parameter will place all generated files into the passed directory. Any relative path specified is 
relative to the _build_ directory as any sources should not be checked into source control and destroyed with `clean`. 
However, in true Gradle tradition, this may use any absolute path.
  
The following example changes the output directory for _all_ source sets to be 
`${buildDir}/src-gen/${sourceSet.testName}/java/`. The same change could be applied to a local 
scope. 

```gradle
assertJ {
    // default: generated-srcs/${SOURCE_SET_NAME_TAG}/java
    outputDir = "src-gen/${SOURCE_SET_NAME_TAG}/java" 
}

sourceSets {
    main { 
        assertJ {} // turn on assertJ generation
    }
}
```

#### P: templates - `Templates`

Default: Built-in templates

Templates can be configured to change the output of the generator. The templates to be replaced can be specified via 
either `String`s or `File`s. Within this Closure, all paths are relative to where they are defined. For example, if 
defined in the global scope, files are relative to `${projectDir}/assertj-templates` by default, for local scopes, the
path of the source set (e.g. `main` implies templates are in `src/main/assertj-templates`). This convention may be 
overridden by specifying the `dir` parameter within the block. 

For more information on names and template syntax, please see the 
[AssertJ Generator Templates Section](http://joel-costigliola.github.io/assertj/assertj-assertions-generator.html#generated-assertions-templates).


The following example replaces the whole number field with a custom template for only the `main` sourceSet, any others 
remain unaffected. 

```gradle
assertJ { }

sourceSets {
    main {
        assertJ {
            templates {
                wholeNumberAssertion = 'public get${Property}() { }'
            }              
        }
    }
}
```

The following example changes the template to a `file()` template for _all_ source sets. 
```gradle
assertJ { 
    templates {
        // Set the template to file content: assertj-templates/wholeNumberOverride.txt
        wholeNumberAssertion = file('wholeNumberOverride.txt')
    }
}

sourceSets {
    main {
        assertJ { }
    }
}
```

The `templateDir` is not a scope-override property. If set, it only applies to the block it is defined within. This was 
 intentionally done to remove ambiguity. Additionally, this example can be applied to local scopes.
 
```gradle
assertJ { 
    templates {
        // Set all templates in this block to be relative to the folder specified
        dir = "${projectDir}/gradle/other-templates"
        
        // Change the file content to:
        //      ${projectDir}/gradle/other-templates/wholeNumberOverride.txt
        wholeNumberAssertion = file('wholeNumberOverride.txt')
    }
}

sourceSets {
    main {
        assertJ { }
    }
}
```


#### P: entryPoints - `EntryPointsGeneratorOptions`

Default: Only generate "standard" assertions

Entry points are classes that provide mechanisms to get access to all of the generated AssertJ types. 
The [AssertJ Generator - Entry Points](http://joel-costigliola.github.io/assertj/assertj-assertions-generator.html#generated-entry-points)
explains how these work and what is expected to be generated for each entry point. The following table shows a mapping 
value to entry point: 

| Value       | Enum Value  | Entry Point | 
|:------------|:------------|:------------|
| `bdd`       | `BDD`       | [BDD style](http://joel-costigliola.github.io/assertj/assertj-core-news.html#assertj-core-1.6.0-bdd-assertions-style) |
| `standard`  | `STANDARD`  | [Standard assertions style](http://joel-costigliola.github.io/assertj/assertj-assertions-generator.html#generated-entry-points) | 
| `junitSoft` | `JUNIT_SOFT`| [JUnit Soft](http://joel-costigliola.github.io/assertj/assertj-core-features-highlight.html#soft-assertions) |
| `soft`      | `SOFT`      | [Soft](http://joel-costigliola.github.io/assertj/assertj-core-features-highlight.html#soft-assertions)

By default, only the `standard` style is turned on. To adjust this, simply set the values to `true` within the 
`entryPoints` closure. 

```gradle
// For all source sets: 
assertJ { 
    entryPoints {
        standard = false // No more standard generation
        bdd = true       // Turn on BDD
        junitSoft = true // and JUnit Soft
    }
}

sourceSets {
    main {
        assertJ { }
    }
}
```

A useful trick to turn on _only_ a set of values is to just set the `entryPoints` to a collection of types: 

```gradle
// For all source sets: 
assertJ { }

sourceSets {
    main {
        assertJ {
            entryPoints = ['BDD', 'JUNIT_SOFT'] // turn on _only_ BDD and JUnit Soft
        }
    }
}
```

## Task Provided
 
* `generateAssertJ` - Generates all sources via the AssertJ Generator
* `cleanAssertJ` - Cleans all generated files 
  * TODO Not implemented

### Lifecycle

This plugin injects itself where it is needed for complete compilation. 

1. Java code that will be "generated from" is compiled
2. These compiled classes and the source files "classpath" are used to generate the files
3. These files are placed into the source path for the `test` sourceSet (TODO configurable?)
4. The `test` set is compiled

## Alternatives

There exists several other alternative Gradle plugins that perform the same functionality as this one.
Currently known:

* [opengl-8080/assertjGen-gradle-plugin](https://github.com/opengl-8080/assertjGen-gradle-plugin) - built to wrap the 
  example provided by @joel-costigiola, [here](https://github.com/joel-costigliola/assertj-assertions-generator/blob/master/src/main/scripts/build.gradle)
  which does not allow for configuration. This plugin does _not_ tie the plugin to a specific version of the AssertJ 
  Generator
* [fhermansson/assertj-generator-gradle-plugin](https://github.com/fhermansson/assertj-generator-gradle-plugin) - only 
  works on a single source-set and still just wraps package/class names as strings
  
## License

This plugin is licensed under the Apache License. See [LICENSE](./LICENSE) for more details. 