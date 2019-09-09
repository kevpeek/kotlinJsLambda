# Kotlin/JS AWS Lambda Demo

## Description
This guide walks through the steps of writing an AWS Lambda function using
Kotlin/JS. My goal is to keep code and configuration to a minimum to facilitate
getting a functioning Lambda function.

## Environment
The following describes the environment I used when compiling this guide. Other
versions of dependencies may work. PRs with updates an corrections are welcome.

- Kotlin 1.3.50
- Gradle 5.6.2
- Java 1.8
- AWS Lambda function targeting Node.js 10.x

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
fun handle(event: Any, context: Any) {
  println("hello lambda!")
}
```

### Compilation

The project is ready to compile! Run `gradle build` and everything should compile successfully.

First, the compiled Lambda function can be found in `./web/output.js`.  In addition, the Kotlin standard library was compiled into `./web/lib/kotlin.js`.

### Creating A Lambda Function

Log into the AWS console and go the Lambda page.

Click `Create Function` and set up a Node.js function according to the docs.

In the configuration page for your function, scroll down to the “Function code” section and create two new files `kotlinjslambda.js` and `kotlin.js`. Copy the contents of your local files into the respective files in AWS.

To direct Lambda to our function’s entry point, specify the handler as `kotlinjslambda.handler`. This should match the value supplied to the `@JsName` annotation earlier.

Before saving and testing the function, one small change needs to be made to the compiled code. In `kotlinjslambda.js` find the import for the kotlin module and change it to expect a local module. Specifically, change the following line:

`}(module.exports, require('kotlin')));`
to instead be:
`}(module.exports, require('./kotlin')));`

Save the function, and press the “Test” button. You will need to create a test input. The provided template will suffice, so give it a name and press “Create”.

Now press “Test” again and your function should run successfully.

### Using A Layer For The Standard Library

It is inconvenient to copy/paste the (50,000 line) Kotlin standard library with every Kotlin/JS Lambda function
you create. Thankfully, Lambda provides [layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) to help with this. A layer can be created to provide the Kotlin standard library, then individual Lambda functions can simply include this layer and only concern themselves with the logic for the function itself.

To create a layer for the Kotlin standard library, place the `kotlin.js` file into the directory structure `./nodejs/node_module/kotlin.js` and zip up the `nodejs` directory into a .zip archive. Now, open the Lambda page in the AWS console and go to the "layers" section. Click "create layer", and give the layer a name, description, target runtime, etc. Upload the zip archive containing the `kotlin.js` file and press "Create."

Now, go back to the Lambda function created earlier. Click the "layers" button in the "Designer" panel. Clikc "Add Layer" and specify the new layer. Once the layer has been added, return to the "Function Code" view and delete the `kotlin.js` file created earlier. Finally, we need to undo the module import change from earlier. In `kotlinjslambda.js` change this line
`}(module.exports, require('./kotlin')));`
back to:
`}(module.exports, require('kotlin')));`

The Lambda function now gets the Kotlin standard library from the layer.
