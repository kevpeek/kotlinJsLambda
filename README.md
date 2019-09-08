# Kotlin/JS AWS Lambda Demo

## Description
TODO

## Environment Setup
TODO

## Steps
Below are the steps I followed to set up my Lambda function. These steps were valid
at the time of writing (8 September 2019).

### Project Setup

First, create a project directory, cd into it, and initialize a gradle project.

```
mkdir kotlinjslambda
cd kotlinjslambda
gradle init
```

I used the `basic` project type, the Groovy DSL (because there is more documentation available online), and the default project name.

Next, I paste the following into the `build.gradle` file:

```
group 'org.example'
version '1.0-SNAPSHOT'

buildscript {
    ext.kotlin_version = '1.3.50'
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

apply plugin: 'kotlin2js'

repositories {
    mavenCentral()
}

dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib-js:$kotlin_version"
}
```

This is the boilerplate `build.gradle` file provided by the Kotlin/JS docs [here](https://kotlinlang.org/docs/tutorials/javascript/getting-started-gradle/getting-started-with-gradle.html).

The Kotlin/JS docs list two changes that need to be made to `build.gradle` . Add the following to the bottom of the file.

```
compileKotlin2Js {
    kotlinOptions.outputFile = "${projectDir}/web/output.js"
    kotlinOptions.moduleKind = "commonjs"
    kotlinOptions.sourceMap = true
}

task assembleWeb(type: Sync) {
    configurations.compile.each { File file ->
        from(zipTree(file.absolutePath), {
            includeEmptyDirs = false
            include { fileTreeElement ->
                def path = fileTreeElement.path
                path.endsWith(".js") && (path.startsWith("META-INF/resources/") ||
                    !path.startsWith("META-INF/"))
            }
        })
    }
    from compileKotlin2Js.destinationDir
    into "${projectDir}/web/lib"

    dependsOn classes
}

assemble.dependsOn assembleWeb
```

Note: I have made two small changes from the Kotlin/JS docs:
* I specify `commonjs` as the module system.
* I copy the standard library into `${projectDir}/web/lib` instead of just `${projectDir}/web`.

### Writing The Code

Next, I create a Kotlin source file located at  `./src/main/kotlin/Main.kt` which looks as follows:

```
@JsName("handler")
fun handle(event: Any?, context: Any?) {
  println("hello lambda!")
}
```

### Compilation

We’re ready to compile! Run `gradle build` and everything should compile successfully.

First, we should find the compiled version of our code at `./web/output.js`.  In addition, the Kotlin standard library was compiled into `./web/lib/kotlin.js`.

### Creating A Lambda Function

Log into the AWS console and go the Lambda page.

Click `Create Function` and set up a Node.js function according to the docs.

In the configuration page for your function, scroll down to the “Function code” section and create two new files `kotlinjslambda.js` and `kotlin.js`. Copy the contents of your local files into the respective files in AWS.

To direct Lambda to our function’s entry point, specify the handler as `kotlinjslambda.handler`. This should match the value supplied to the `@JsName` annotation earlier.

Before we save and test our function, we need to make one small change to the compiled code. In `kotlinjslambda.js` find the import for the kotlin module and change it to expect a local module. Specifically, change the following line:

`}(module.exports, require('kotlin')));`
to instead be:
`}(module.exports, require('./kotlin')));`

Save the function, and press the “Test” button. You will need to create a test input. The provided template will suffice, so give it a name and press “Create”.

Now press “Test” again and your function should run successfully.

### Using A Layer For The Standard Library

TODO
